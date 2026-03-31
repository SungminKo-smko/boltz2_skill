# boltz2-predict

Claude Code skill for Boltz-2 protein structure prediction.

## Features

- Protein complex structure prediction from amino acid sequences
- Structure file (.cif/.pdb) upload and prediction
- Nanobody-target cross-model workflow (boltzgen → Boltz-2)
- Job submission, status tracking, and artifact download
- MCP integration (Streamable HTTP + stdio)

## Install

```bash
git clone https://github.com/SungminKo-smko/boltz2_skill ~/.claude/skills/boltz2-predict
```

## MCP Server Setup

### Remote (recommended)

```bash
claude mcp add boltz2 --transport http https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/mcp/mcp
```

### Local (development)

```bash
cd ~/workspace/boltz2_MSA && source .venv/bin/activate
claude mcp add boltz2 python3 -m boltz2_service.mcp.stdio
```

## API Key

Get your API key via Google OAuth:
1. Visit https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io/auth/login
2. Login with your @shaperon.com account
3. Save the returned API key

Or save it directly:
```bash
echo "API_KEY=b2_your_key_here" > ~/.claude/skills/boltz2-predict/.env
```

## Usage

In Claude Code, just say:
- "이 두 단백질의 구조를 예측해줘" + sequences
- "이 나노바디와 타겟의 복합체 구조 예측"
- "boltz2로 structure prediction 해줘"

## API

Base URL: `https://boltz2-api.politebay-55ff119b.westus3.azurecontainerapps.io`

- Swagger UI: `/docs`
- MCP endpoint: `/mcp/mcp`
- Health check: `/healthz`
