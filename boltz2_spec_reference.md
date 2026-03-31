# Boltz-2 Spec YAML Reference

> Source: https://github.com/jwohlwend/boltz
> Boltz-2 v2 YAML specification format

---

## 기본 구조

```yaml
version: 2
entities:
  - protein: ...     # 단백질 체인
  - cif: ...         # 기존 구조 파일
  - ligand: ...      # 소분자 리간드
constraints:         # (선택) 구조 제약
  - bond: ...
```

---

## Entities

### protein — 단백질 시퀀스

```yaml
- protein:
    id: A                        # 체인 ID (필수, 대문자)
    sequence: EVQLVESGGGLVQ...   # 1-letter 아미노산 시퀀스 (필수)
```

**다중 체인 예시:**
```yaml
version: 2
entities:
  - protein:
      id: A
      sequence: MKTLLILAVLCLGFASS...
  - protein:
      id: B
      sequence: EVQLVESGGGLVQPGG...
  - protein:
      id: C
      sequence: DIQMTQSPSSLSAS...
```

### cif — 기존 구조 파일

```yaml
- cif:
    path: targets/input.cif    # 업로드된 파일의 상대 경로
```

업로드 시 `create_upload_url` 또는 `upload_structure`로 asset 생성 후,
`relative_path`가 자동으로 `targets/<filename>` 형태로 설정됨.

### ligand — 소분자

```yaml
- ligand:
    id: L
    smiles: 'CCO'              # SMILES 표기
```

또는 CCD 코드:
```yaml
- ligand:
    id: L
    ccd: ATP                   # Chemical Component Dictionary 코드
```

---

## Constraints (선택)

### bond — 공유결합 제약

```yaml
constraints:
  - bond:
      atom1: [A, 1, CA]       # [chain_id, residue_number, atom_name]
      atom2: [B, 50, CA]
```

---

## 예시: 나노바디-타겟 복합체

```yaml
version: 2
entities:
  - cif:
      path: targets/target.cif
  - protein:
      id: N
      sequence: EVQLVESGGGLVQPGGSLRLSCAASASTFTVGHMSWVRQAPGKGLEWVSD...
```

## 예시: 두 단백질 복합체

```yaml
version: 2
entities:
  - protein:
      id: A
      sequence: MKTLLILAVLCLGFASS...
  - protein:
      id: B
      sequence: EVQLVESGGGLVQPGG...
```

## 예시: 단백질 + 리간드

```yaml
version: 2
entities:
  - protein:
      id: A
      sequence: MKTLLILAVLCLGFASS...
  - ligand:
      id: L
      smiles: 'c1ccc(cc1)C(=O)O'
```

---

## prediction_type 옵션

| 값 | 설명 | 용도 |
|----|------|------|
| `structure` | 3D 구조 예측 | 기본, 가장 많이 사용 |
| `affinity` | 결합 친화도 예측 | 결합 세기 정량화 |
| `structure+affinity` | 구조 + 친화도 동시 | 구조와 결합 세기 모두 필요 |
| `virtual_screening` | 가상 스크리닝 | 대규모 리간드 스크리닝 |

## runtime_options

| 파라미터 | 기본 | 범위 | 설명 |
|----------|------|------|------|
| `diffusion_samples` | 1 | 1-10 | 구조 샘플 수 (다양성↑) |
| `sampling_steps` | 200 | 50-1000 | 디퓨전 샘플링 스텝 수 |
| `recycling_steps` | 3 | 1-10 | 구조 리파인먼트 반복 |
| `step_scale` | - | 0.5-3.0 | 스텝 스케일 팩터 |
| `output_format` | mmcif | pdb/mmcif | 출력 파일 형식 |
| `use_msa_server` | true | - | MSA 서버로 시퀀스 정렬 |
| `use_potentials` | false | - | 에너지 기반 구조 가이드 |
| `seed` | - | int | 랜덤 시드 (재현성) |
| `write_full_pae` | false | - | 전체 PAE 행렬 출력 |
