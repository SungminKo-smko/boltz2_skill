# boltz2-predict

**Boltz-2 단백질 구조 예측 자동화 스킬 for Claude Code**

Boltz-2 MCP 서버를 통해 단백질 구조 예측 잡을 제출하고 관리하는 Claude Code 스킬입니다. 시퀀스 입력, 구조 파일 업로드, 나노바디-타겟 cross-model workflow 등 다양한 예측 시나리오를 지원합니다.

> Protein structure prediction automation skill powered by Boltz-2. Submit prediction jobs via MCP server — supports sequence-based prediction, structure file upload, and nanobody-target cross-model workflows.

---

## 목차

- [설치](#설치)
- [MCP 서버 설정](#mcp-서버-설정)
- [API Key 발급](#api-key-발급)
- [사용 예시](#사용-예시)
- [MCP Tools (13종)](#mcp-tools)
- [REST API 엔드포인트](#rest-api-엔드포인트)
- [Boltz-2 YAML Spec 레퍼런스](#boltz-2-yaml-spec-레퍼런스)
- [Runtime Options](#runtime-options)
- [prediction_type 옵션](#prediction_type-옵션)
- [문제 해결](#문제-해결)
- [관련 저장소](#관련-저장소)

---

## 설치

```bash
git clone https://github.com/SungminKo-smko/boltz2_skill ~/.claude/skills/boltz2-predict
```

스킬이 설치되면 Claude Code가 자동으로 SKILL.md를 인식하여 구조 예측 관련 요청 시 활성화됩니다.

---

## MCP 서버 설정

### HTTP 모드 (권장 - OAuth 2.1 인증)

Streamable HTTP 전송을 사용하는 원격 MCP 서버에 연결합니다. 처음 연결 시 브라우저가 열리며 Google OAuth 로그인이 진행됩니다.

```bash
claude mcp add boltz2 --transport http https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/mcp/mcp
```

**원격 환경 (SSH, Codespaces, Dev Container 등):**

브라우저를 직접 열 수 없는 원격 환경에서는 `--callback-port`를 지정하여 OAuth 콜백을 받을 포트를 고정합니다. 해당 포트를 로컬로 포트 포워딩한 뒤 연결하세요.

```bash
claude mcp add boltz2 --transport http \
  https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/mcp/mcp \
  --callback-port 9999
```

포트 포워딩 예시 (별도 터미널):
```bash
ssh -L 9999:localhost:9999 your-remote-host
```

### Local stdio 모드 (개발용)

로컬에 Boltz-2 서비스가 설치된 경우 stdio 모드로 직접 실행할 수 있습니다.

```bash
cd ~/workspace/boltz2_MSA && source .venv/bin/activate
claude mcp add boltz2 python3 -m boltz2_service.mcp.stdio
```

### 등록 확인

```bash
claude mcp list 2>/dev/null | grep boltz2
```

---

## API Key 발급

`@shaperon.com` Google 계정으로 자동 발급됩니다.

### 방법 1: 브라우저에서 발급

1. [https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/auth/login](https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/auth/login) 접속
2. `@shaperon.com` Google 계정으로 로그인
3. 반환된 API Key를 복사

### 방법 2: HTTP MCP 사용 시 (OAuth 2.1)

HTTP 전송 모드로 MCP 서버에 연결하면 첫 사용 시 브라우저에서 Google 로그인이 자동으로 진행됩니다. 별도의 API Key 설정 없이 인증이 완료됩니다.

### 방법 3: 수동 저장

발급받은 키를 스킬 환경 파일에 저장합니다:

```bash
mkdir -p ~/.claude/skills/boltz2-predict
echo "API_KEY=b2_your_key_here" > ~/.claude/skills/boltz2-predict/.env
```

---

## 사용 예시

Claude Code에서 자연어로 요청하면 스킬이 자동으로 활성화됩니다.

### 시퀀스 기반 구조 예측

가장 일반적인 사용 방식입니다. 아미노산 시퀀스를 직접 제공합니다.

```
이 두 단백질의 복합체 구조를 예측해줘:

Chain A: MKTLLILAVLCLGFASSALAGTEDSALVNQGRFFSDLEAIPRYNISGS...
Chain B: EVQLVESGGGLVQPGGSLRLSCAASGFTFSSYWMSWVRQAPGKGLEWV...
```

```
boltz2로 이 시퀀스의 구조 예측해줘:
EVQLVESGGGLVQPGGSLRLSCAASGFTFSSYWMS...
```

### 구조 파일 기반 예측

`.cif` 또는 `.pdb` 파일을 업로드한 뒤 추가 시퀀스와 함께 예측합니다.

```
이 구조 파일과 리간드의 복합체 구조를 예측해줘:
파일: /path/to/target.cif
리간드 SMILES: c1ccc(cc1)C(=O)O
```

```
target.pdb 파일을 업로드하고, 이 나노바디 시퀀스와의 복합체를 예측해줘:
EVQLVESGGGLVQPGGSLRLSCAAS...
```

### 나노바디-타겟 Cross-Model Workflow

BoltzGen으로 디자인한 나노바디 시퀀스를 Boltz-2로 구조 예측하는 워크플로입니다.

```
boltzgen에서 디자인된 나노바디 시퀀스를 Boltz-2로 구조 예측해줘:
나노바디: EVQLVESGGGLVQPGGSLRLSCAAS...
타겟 asset_id: abc-123-def
```

```
나노바디 디자인 결과를 구조 예측으로 연결해줘
```

### 잡 상태 확인 / 결과 다운로드 / 잡 취소

```
잡 상태 확인해줘: job_id = abc-123-def
```

```
결과 다운로드해줘: abc-123-def
```

```
실행 중인 잡 취소해줘: abc-123-def
```

```
현재 실행 중인 잡 목록 보여줘
```

---

## MCP Tools

총 13개의 MCP 도구를 제공합니다.

| Tool | 설명 |
|------|------|
| `validate_spec` | Boltz-2 YAML spec 검증 (문법, 시퀀스 유효성 확인) |
| `submit_job` | 검증된 spec으로 구조 예측 잡 제출 |
| `get_job` | 잡 상태 조회 (status, progress, failure 정보) |
| `get_logs` | 잡 실행 로그 조회 (진행 단계, 에러 메시지) |
| `list_jobs` | 잡 목록 조회 (상태별 필터링 가능) |
| `cancel_job` | 실행 중인 잡 취소 |
| `get_artifacts` | 완료된 잡의 결과 파일 다운로드 URL 조회 |
| `create_upload_url` | 구조 파일 업로드용 SAS URL 생성 |
| `upload_structure` | 구조 파일 직접 업로드 (create_upload_url + curl 자동화) |
| `render_template` | 업로드된 구조 파일로부터 YAML spec 자동 생성 |
| `list_templates` | 사용 가능한 spec 템플릿 목록 조회 |
| `list_workers` | 워커 상태 및 가용성 조회 |
| `submit_nanobody_structure_prediction` | 나노바디-타겟 복합체 구조 예측 (spec 생성+검증+제출 일괄 처리) |

---

## REST API 엔드포인트

Base URL: `https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io`

| Method | Path | 설명 |
|--------|------|------|
| `GET` | `/healthz` | 헬스체크 |
| `GET` | `/auth/login` | Google OAuth 로그인 시작 |
| `POST` | `/auth/device-code` | Device Auth 코드 발급 |
| `POST` | `/v1/boltz2/uploads` | 구조 파일 업로드 URL 생성 |
| `GET` | `/v1/boltz2/spec-templates` | Spec 템플릿 목록 |
| `POST` | `/v1/boltz2/spec-templates/render` | Spec 렌더링 (구조 파일 기반) |
| `POST` | `/v1/boltz2/specs/validate` | YAML Spec 검증 |
| `POST` | `/v1/boltz2/prediction-jobs` | 예측 잡 제출 |
| `GET` | `/v1/boltz2/prediction-jobs` | 잡 목록 조회 |
| `GET` | `/v1/boltz2/prediction-jobs/{id}` | 잡 상세 정보 |
| `POST` | `/v1/boltz2/prediction-jobs/{id}:cancel` | 잡 취소 |
| `GET` | `/v1/boltz2/prediction-jobs/{id}/artifacts` | 결과 파일 다운로드 |
| `GET` | `/docs` | Swagger UI (API 문서) |
| `GET/POST` | `/mcp/mcp` | MCP Streamable HTTP 엔드포인트 |

---

## Boltz-2 YAML Spec 레퍼런스

### 기본 구조

```yaml
version: 1
sequences:
  - protein:
      id: A
      sequence: MKTLLILAVLCLGFASS...
  - protein:
      id: B
      sequence: EVQLVESGGGLVQPGG...
```

### Entity 타입

**protein** -- 아미노산 시퀀스

```yaml
- protein:
    id: A                        # 체인 ID (필수, 대문자)
    sequence: MKTLLILAVLCLGFASS  # 1-letter 아미노산 시퀀스 (필수)
```

**cif** -- 기존 구조 파일

```yaml
- cif:
    path: targets/input.cif      # 업로드된 파일의 상대 경로
```

**ligand** -- 소분자 (SMILES 또는 CCD 코드)

```yaml
- ligand:
    id: L
    smiles: 'CCO'                # SMILES 표기
    # ccd: ATP                   # 또는 Chemical Component Dictionary 코드
```

### Constraints (선택)

```yaml
constraints:
  - bond:
      atom1: [A, 1, CA]         # [chain_id, residue_number, atom_name]
      atom2: [B, 50, CA]
```

### 예시: 두 단백질 복합체

```yaml
version: 1
sequences:
  - protein:
      id: A
      sequence: MKTLLILAVLCLGFASS...
  - protein:
      id: B
      sequence: EVQLVESGGGLVQPGG...
```

### 예시: 나노바디-타겟 복합체

```yaml
version: 1
sequences:
  - cif:
      path: targets/target.cif
  - protein:
      id: N
      sequence: EVQLVESGGGLVQPGGSLRLSCAAS...
```

### 예시: 단백질 + 리간드

```yaml
version: 1
sequences:
  - protein:
      id: A
      sequence: MKTLLILAVLCLGFASS...
  - ligand:
      id: L
      smiles: 'c1ccc(cc1)C(=O)O'
```

---

## Runtime Options

| 파라미터 | 기본값 | 범위 | 설명 |
|----------|--------|------|------|
| `diffusion_samples` | 1 | 1-10 | 디퓨전 샘플 수 (많을수록 다양한 구조 생성) |
| `sampling_steps` | 200 | 50-1000 | 디퓨전 샘플링 스텝 수 |
| `recycling_steps` | 3 | 1-10 | 구조 리파인먼트 반복 횟수 |
| `step_scale` | - | 0.5-3.0 | 스텝 스케일 팩터 |
| `output_format` | mmcif | pdb / mmcif | 출력 파일 형식 |
| `use_msa_server` | true | - | MSA 서버로 다중 시퀀스 정렬 수행 |
| `use_potentials` | false | - | 에너지 기반 구조 가이드 사용 |
| `seed` | - | int | 랜덤 시드 (재현성 확보) |
| `write_full_pae` | false | - | 전체 PAE (Predicted Aligned Error) 행렬 출력 |

---

## prediction_type 옵션

| 값 | 설명 | 용도 |
|----|------|------|
| `structure` | 3D 구조 예측 | 기본값. 가장 많이 사용 |
| `affinity` | 결합 친화도 예측 | 결합 세기 정량화 |
| `structure+affinity` | 구조 + 친화도 동시 예측 | 구조와 결합 세기 모두 필요할 때 |
| `virtual_screening` | 가상 스크리닝 | 대규모 리간드 스크리닝 |

---

## 문제 해결

### API Key 관련

| 증상 | 해결 방법 |
|------|-----------|
| `401 Unauthorized` | API Key 확인. `~/.claude/skills/boltz2-predict/.env`에 `API_KEY=<key>` 저장 |
| Key 발급 안됨 | `@shaperon.com` 계정으로만 발급 가능. [로그인 페이지](https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/auth/login) 확인 |

### MCP 연결 관련

| 증상 | 해결 방법 |
|------|-----------|
| `NOT_REGISTERED` | `claude mcp add boltz2 --transport http <URL>` 실행 |
| OAuth 브라우저 안 열림 | 원격 환경이면 `--callback-port 9999` 추가 + 포트 포워딩 |
| 연결 타임아웃 | 서버 상태 확인: `curl https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/healthz` |

### 잡 실행 관련

| 증상 | 해결 방법 |
|------|-----------|
| `503 boltz2 binary not available` | 서버에 Boltz-2 바이너리 미설치. 관리자 문의 |
| `429 Rate limit` | 일일 한도(20건) 또는 동시 실행 한도(2건) 초과. 대기 후 재시도 |
| `boltz2_run_failed` | Boltz-2 실행 오류. `get_logs`로 상세 로그 확인 |
| `boltz2_run_timeout` | 4시간 타임아웃 초과. 입력 크기 줄이거나 `diffusion_samples` 감소 |
| `worker_timeout` | 워커 heartbeat 없음. `list_workers`로 워커 상태 확인 |
| Spec 검증 실패 | `validate_spec` 반환 `errors` 확인. 시퀀스 형식, 체인 ID 중복 등 점검 |

### 파일 업로드 관련

| 증상 | 해결 방법 |
|------|-----------|
| 업로드 실패 | `create_upload_url`로 SAS URL 재발급 후 재시도 |
| 파일 형식 오류 | `.cif` 또는 `.pdb` 형식만 지원 |
| 컨텍스트 초과 | 파일 내용을 Read로 읽지 말고 반드시 `create_upload_url` + `curl`로 업로드 |

---

## BoltzGen 통합 (Cross-Service)

boltz2와 boltzgen은 동일한 Supabase 인증 시스템(nanomapAIDEN 프로젝트)을 공유합니다.
boltz2 API Key를 이미 발급받은 사용자는 동일한 로그인으로 boltzgen용 Key도 별도 발급받을 수 있습니다.

- 두 서비스 모두 `b2_` 접두사를 사용하지만, Key는 서비스별로 분리됩니다
- **boltz2 Key는 boltz2에서만, boltzgen Key는 boltzgen에서만 유효합니다**
- 인증 엔드포인트 패턴은 동일: `/auth/login`, `/auth/callback`, `/auth/device-code`

---

## 관련 저장소

| 저장소 | 설명 |
|--------|------|
| [SungminKo-smko/boltz2_skill](https://github.com/SungminKo-smko/boltz2_skill) | 이 저장소 (Claude Code 스킬) |
| [SungminKo-smko/boltz2_MSA](https://github.com/SungminKo-smko/boltz2_MSA) | Boltz-2 API 서버 (백엔드) |
| [SungminKo-smko/boltzgen_skill](https://github.com/SungminKo-smko/boltzgen_skill) | BoltzGen 나노바디 디자인 스킬 |
| [jwohlwend/boltz](https://github.com/jwohlwend/boltz) | Boltz-2 원본 저장소 |
