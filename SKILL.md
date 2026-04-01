---
name: boltz2-predict
version: 1.0.0
description: |
  Boltz-2 단백질 구조 예측 자동화 스킬. 시퀀스 또는 구조 파일을 입력받아
  업로드 → spec 생성 → 검증 → 잡 제출 → 상태 추적까지 전체 워크플로를 자동화한다.
  나노바디-타겟 cross-model workflow (boltzgen → Boltz-2) 도 지원.
  Use when asked to "구조 예측", "structure prediction", "boltz2 실행",
  "단백질 복합체 예측", "나노바디 구조 예측", "docking prediction".
allowed-tools:
  - Bash
  - Read
  - Write
  - Agent
---

# /boltz2-predict

Boltz-2 MCP 서버를 통해 단백질 구조 예측 잡을 제출하는 스킬.

## 설치

```bash
git clone https://github.com/SungminKo-smko/boltz2_skill ~/.claude/skills/boltz2-predict
```

## MCP 서버 확인 (run first)

```bash
claude mcp list 2>/dev/null | grep boltz2 || echo "NOT_REGISTERED"
```

**NOT_REGISTERED** 출력 시 — Streamable HTTP MCP 등록:
```bash
claude mcp add boltz2 --transport http https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/mcp/mcp
```

> **OAuth 2.1 인증**: HTTP 전송 모드를 사용하면 첫 연결 시 브라우저가 열리며 Google OAuth 로그인이 진행됩니다. `@shaperon.com` 계정으로 로그인하면 인증이 자동 완료됩니다.

> **원격 환경 (SSH, Codespaces 등)**: 브라우저를 직접 열 수 없는 환경에서는 `--callback-port 9999`를 추가하고, 해당 포트를 로컬로 포워딩하세요:
> ```bash
> claude mcp add boltz2 --transport http \
>   https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/mcp/mcp \
>   --callback-port 9999
> ```

또는 로컬 stdio 모드 (개발용):
```bash
cd ~/workspace/boltz2_MSA && source .venv/bin/activate
claude mcp add boltz2 python3 -m boltz2_service.mcp.stdio
```

## Step 0: API KEY 로드

```bash
SKILL_ENV="$HOME/.claude/skills/boltz2-predict/.env"
BOLTZ2_API_KEY=""

if [ -f "$SKILL_ENV" ]; then
    BOLTZ2_API_KEY=$(grep -E "^API_KEY=" "$SKILL_ENV" | cut -d= -f2-)
fi

[ -z "$BOLTZ2_API_KEY" ] && BOLTZ2_API_KEY="${BOLTZ2_API_KEY:-${API_KEY:-}}"

echo "BOLTZ2_API_KEY=${BOLTZ2_API_KEY}"
```

- **값이 있으면** → 그 값을 모든 tool 호출의 `api_key` 인수로 전달
- **값이 없으면** → 사용자에게 직접 입력 요청:
  > "Boltz-2 API KEY를 입력해 주세요. (https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/auth/login 에서 발급)"

  입력받은 키를 저장:
  ```bash
  mkdir -p "$HOME/.claude/skills/boltz2-predict"
  echo "API_KEY=<입력받은값>" > "$HOME/.claude/skills/boltz2-predict/.env"
  ```

## Step 1: 입력 모드 판별

사용자 입력을 분석하여 **3가지 모드** 중 하나를 선택:

### 모드 A: 시퀀스 기반 예측 (가장 일반적)
- 사용자가 아미노산 시퀀스를 직접 제공한 경우
- FASTA 형식이거나 raw 시퀀스
- → **Step 3A**로 이동

### 모드 B: 구조 파일 기반 예측
- 사용자가 .cif 또는 .pdb 파일을 제공한 경우
- → **Step 2**로 이동 (파일 업로드 필요)

### 모드 C: 나노바디-타겟 Cross-Model Workflow
- 사용자가 나노바디 시퀀스 + 타겟 구조를 모두 제공한 경우
- 또는 "나노바디 구조 예측", "nanobody docking" 등 키워드
- → **Step 2** (타겟 업로드) → **Step 3C**로 이동

### 요구사항 수집

사용자가 이미 정보를 제공했으면 묻지 않고 바로 진행.

**모드 A 필수:**
1. 단백질 시퀀스 (1개 이상)
2. 각 체인 ID (기본: A, B, C...)

**모드 B 필수:**
1. 구조 파일 경로 (.cif 또는 .pdb)
2. 추가 sequence (선택 — 단백질 시퀀스, 리간드 SMILES 등)

**모드 C 필수:**
1. 나노바디 시퀀스
2. 타겟 구조 파일 경로 또는 asset_id

**공통 선택:**
- `prediction_type` — structure (기본), affinity, structure+affinity
- `diffusion_samples` — 기본 1 (1-10)
- `output_format` — mmcif (기본) 또는 pdb

## Step 2: 구조 파일 업로드 (모드 B, C만)

> **절대 금지**: base64 인코딩, Read로 파일 내용 읽기 — 컨텍스트 초과 유발.
> `create_upload_url`로 SAS URL을 받아 curl로 직접 업로드한다.

### Claude Desktop 감지

```bash
if [ -d "/mnt/user-data/uploads" ]; then
    echo "CLAUDE_DESKTOP=true"
    ls /mnt/user-data/uploads/
else
    echo "CLAUDE_DESKTOP=false"
fi
```

### 업로드

```python
create_upload_url(
    filename="<파일명>",
    api_key="<BOLTZ2_API_KEY>"
)
# → asset_id, upload_url, content_type, curl_hint
```

반환된 `curl_hint`의 `<FILE_PATH>`를 실제 경로로 교체 후 Bash 실행:

```bash
curl -s -X PUT -T "<절대경로>" \
  -H "x-ms-blob-type: BlockBlob" \
  -H "Content-Type: <content_type>" \
  "<upload_url>"
```

## Step 3A: 시퀀스 기반 Spec 생성 + 잡 제출

YAML spec을 직접 생성하여 검증 + 제출:

```python
validate_spec(
    raw_yaml="""version: 1
sequences:
  - protein:
      id: A
      sequence: <시퀀스1>
  - protein:
      id: B
      sequence: <시퀀스2>
""",
    asset_ids=[],
    api_key="<BOLTZ2_API_KEY>"
)
# → spec_id (valid=True 일 때)
```

검증 성공 시:

```python
submit_job(
    spec_id="<spec_id>",
    prediction_type="structure",
    diffusion_samples=1,
    api_key="<BOLTZ2_API_KEY>"
)
# → job_id
```

**검증 실패 시:**
- `errors` 내용을 사용자에게 보여주고 수정 안내
- 서버에 `boltz` 바이너리가 없으면 503 반환 — 이 경우 검증을 건너뛰고 raw spec으로 직접 제출 불가. 사용자에게 알린다.

## Step 3B: 구조 파일 기반 Spec 생성

### 템플릿 렌더링 (권장)

```python
render_template(
    target_asset_id="<asset_id>",
    additional_sequences=[
        {"protein": {"id": "B", "sequence": "<추가 시퀀스>"}},
        # {"ligand": {"id": "L", "smiles": "CCO"}}  # 리간드 추가 시
    ],
    constraints=[],  # 선택
    api_key="<BOLTZ2_API_KEY>"
)
# → spec_id, canonical_yaml
```

### Raw YAML (복잡한 케이스)

```python
validate_spec(
    raw_yaml="<yaml 문자열>",
    asset_ids=["<asset_id>"],
    api_key="<BOLTZ2_API_KEY>"
)
```

이후 `submit_job`으로 제출.

## Step 3C: 나노바디-타겟 Cross-Model Workflow

boltzgen에서 디자인한 나노바디 시퀀스를 Boltz-2로 구조 예측:

```python
submit_nanobody_structure_prediction(
    nanobody_sequence="<나노바디 시퀀스>",
    target_asset_id="<타겟 asset_id>",
    nanobody_chain_id="N",
    prediction_type="structure",
    diffusion_samples=1,
    api_key="<BOLTZ2_API_KEY>"
)
# → job_id, spec_id, status, spec_yaml
```

이 도구는 spec 생성 + 검증 + 제출을 한 번에 처리한다.

## Step 4: 상태 확인

잡 제출 후 바로 상태를 확인하고, running 상태이면 정보를 출력한다:

```python
get_job(
    job_id="<job_id>",
    api_key="<BOLTZ2_API_KEY>"
)
```

**출력 예시:**
```
잡 제출 완료!
- Job ID: abc-123-def
- Status: queued → running 대기 중
- 예상 소요시간: 구조 크기에 따라 10분~4시간

상태 확인: get_job(job_id="abc-123-def", ...)
결과 다운로드: get_artifacts(job_id="abc-123-def", ...) (succeeded 후)
```

## Step 5: 결과 확인 및 다운로드

사용자가 결과를 요청하면:

```python
get_job(job_id="<job_id>", api_key="<BOLTZ2_API_KEY>")
```

- `status == "succeeded"` 이면:

```python
get_artifacts(job_id="<job_id>", api_key="<BOLTZ2_API_KEY>")
# → artifacts: {filename: download_url, ...}
```

다운로드 URL을 사용자에게 제공하거나, Bash로 다운로드:

```bash
curl -o results.zip "<download_url>"
```

- `status == "failed"` 이면: `failure_code`, `failure_message` 출력
- `status == "running"` 이면: `progress_percent`, `current_stage` 출력

## 잡 관리 도구

```python
get_job(job_id, api_key="<key>")               # 상태 확인
get_logs(job_id, tail=100, api_key="<key>")    # 진행 상황 + 로그
list_jobs(status="running", api_key="<key>")   # 잡 목록
cancel_job(job_id, api_key="<key>")            # 잡 취소
list_templates(api_key="<key>")               # 템플릿 목록
list_workers(api_key="<key>")                 # 워커 상태
```

## Boltz-2 YAML Spec 레퍼런스

### 기본 구조 (v1)

```yaml
version: 1
sequences:
  - protein:
      id: A
      sequence: MKTL...
  - protein:
      id: B
      sequence: EVQL...
```

### Entity 타입

**protein** — 아미노산 시퀀스
```yaml
- protein:
    id: A                    # 체인 ID (필수)
    sequence: MKTLLILAVL...  # 1-letter 아미노산 시퀀스 (필수)
```

**cif** — 기존 구조 파일
```yaml
- cif:
    path: targets/input.cif  # 업로드된 파일 경로
```

**ligand** — 소분자 (SMILES 또는 CCD 코드)
```yaml
- ligand:
    id: L
    smiles: 'CCO'           # SMILES 표기
    # ccd: ATP               # 또는 CCD 코드
```

### Constraints (선택)

```yaml
constraints:
  - bond:
      atom1: [A, 1, CA]     # [chain, residue, atom]
      atom2: [B, 50, CA]
```

### prediction_type 옵션

| 값 | 설명 |
|----|------|
| `structure` | 기본 구조 예측 |
| `affinity` | 결합 친화도 예측 |
| `structure+affinity` | 구조 + 친화도 동시 |
| `virtual_screening` | 가상 스크리닝 |

### runtime_options 주요 파라미터

| 파라미터 | 기본값 | 범위 | 설명 |
|----------|--------|------|------|
| `diffusion_samples` | 1 | 1-10 | 디퓨전 샘플 수 (많을수록 다양한 구조) |
| `sampling_steps` | 200 | 50-1000 | 샘플링 스텝 |
| `recycling_steps` | 3 | 1-10 | 리사이클링 스텝 |
| `output_format` | mmcif | pdb/mmcif | 출력 형식 |
| `use_msa_server` | true | - | MSA 서버 사용 여부 |
| `use_potentials` | false | - | 에너지 기반 가이드 |

## BoltzGen 통합 (Cross-Service)

boltz2와 boltzgen은 동일한 Supabase 인증 시스템(nanomapAIDEN 프로젝트)을 공유한다.
boltz2 API KEY를 이미 발급받은 사용자는 동일한 로그인(`/auth/login`, `/auth/callback`, `/auth/device-code`)으로 boltzgen용 KEY도 별도 발급받을 수 있다.

- 두 서비스 모두 `b2_` 접두사를 사용하지만 KEY는 서비스별로 분리됨
- **boltz2 KEY는 boltz2에서만, boltzgen KEY는 boltzgen에서만 유효**
- 인증 엔드포인트 패턴은 동일: `/auth/login`, `/auth/callback`, `/auth/device-code`

## 오류 처리

- **API_KEY 미설정**: `~/.claude/skills/boltz2-predict/.env`에 `API_KEY=<key>` 추가
- **MCP 미등록**: `claude mcp add boltz2 --transport http <URL>` 실행
- **503 boltz2 binary not available**: 서버에 boltz 바이너리가 설치되지 않음. 검증 불가.
- **401 인증 실패**: API KEY 확인
- **429 Rate limit**: 일일 한도(20) 또는 동시 한도(2) 초과
- **잡 실패**: `get_job`의 `failure_code` 확인
  - `boltz2_run_failed` — Boltz-2 실행 오류
  - `boltz2_run_timeout` — 4시간 타임아웃 초과
  - `worker_timeout` — 워커 heartbeat 없음

## API 엔드포인트 (REST 직접 호출 시)

Base URL: `https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io`

| Method | Path | 설명 |
|--------|------|------|
| GET | `/healthz` | 헬스체크 |
| GET | `/auth/login` | Google OAuth 시작 |
| POST | `/auth/device-code` | Device Auth 코드 발급 |
| POST | `/v1/boltz2/uploads` | 업로드 URL 생성 |
| GET | `/v1/boltz2/spec-templates` | 템플릿 목록 |
| POST | `/v1/boltz2/spec-templates/render` | Spec 렌더링 |
| POST | `/v1/boltz2/specs/validate` | Spec 검증 |
| POST | `/v1/boltz2/prediction-jobs` | 잡 제출 |
| GET | `/v1/boltz2/prediction-jobs` | 잡 목록 |
| GET | `/v1/boltz2/prediction-jobs/{id}` | 잡 상세 |
| POST | `/v1/boltz2/prediction-jobs/{id}:cancel` | 잡 취소 |
| GET | `/v1/boltz2/prediction-jobs/{id}/artifacts` | 결과 다운로드 |
| GET | `/docs` | Swagger UI |
