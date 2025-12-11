# Service Descriptor: Docling MCP Server

**Service Name**: docling-mcp  
**Node**: hx-docling-mcp-server.hx.dev.local  
**IP Address**: 192.168.10.217  
**Port**: 8000  
**Protocol**: MCP over HTTP/SSE  
**Status**: ✅ OPERATIONAL  
**Operational Date**: 2025-12-04

---

## Table of Contents

1. [Overview](#overview)
2. [MCP Transport Protocols](#mcp-transport-protocols)
3. [Service Architecture](#service-architecture)
4. [MCP Tools Reference](#mcp-tools-reference)
5. [LLM Models & Processing](#llm-models--processing)
6. [Integration Endpoints](#integration-endpoints)
7. [Configuration](#configuration)
8. [Administration](#administration)
9. [Monitoring & Health](#monitoring--health)
10. [Troubleshooting](#troubleshooting)
11. [Related Documentation](#related-documentation)

---

## Overview

### Description

Document processing MCP (Model Context Protocol) server providing AI agents with standardized access to document conversion, knowledge graph generation, and manipulation capabilities. Built on FastMCP framework with Docling for multi-format document parsing.

### Primary Value Proposition

- **MCP Protocol Standardization**: AI agents discover and invoke 19 document processing tools through standard MCP protocol
- **Knowledge Graph RAG**: Entity-relationship graphs enable intelligent retrieval beyond flat vector search
- **Multimodal Processing**: Handle complex documents (PDF, DOCX, PPTX, XLSX, HTML, images) with preserved structure
- **Zero Custom Integration**: AI agents integrate via standard MCP clients

### Supported Document Formats

| Format | Type | OCR Support |
|--------|------|-------------|
| PDF | Digital & Scanned | ✅ EasyOCR |
| DOCX | Microsoft Word | N/A |
| PPTX | PowerPoint | N/A |
| XLSX | Excel | N/A |
| HTML | Web Pages | N/A |
| PNG/JPG/TIFF | Images | ✅ EasyOCR |
| Markdown | Text | N/A |
| EPUB | eBooks | N/A |

---

## MCP Transport Protocols

The Docling MCP Server supports **three transport protocols** for MCP communication:

### Transport Comparison

| Transport | Endpoint | Use Case | Client Examples |
|-----------|----------|----------|-----------------|
| **HTTP** | `http://hx-docling-mcp-server.hx.dev.local:8000/mcp` | Request/response, simple integration | curl, REST clients |
| **SSE** | `http://hx-docling-mcp-server.hx.dev.local:8000/mcp` | Streaming responses, real-time updates | Claude Code, web clients |
| **stdio** | Local process stdin/stdout | Local AI agent integration | Claude Desktop, LM Studio |

### HTTP Transport

Standard HTTP POST requests for synchronous tool invocation.

```bash
# Request
curl -X POST http://hx-docling-mcp-server.hx.dev.local:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'

# Response
{"jsonrpc":"2.0","result":{"tools":[...]},"id":1}
```

### SSE (Server-Sent Events) Transport

Streaming transport for long-running operations with real-time progress updates.

```bash
# SSE Request with streaming response
curl -X POST http://hx-docling-mcp-server.hx.dev.local:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"convert_pdf","arguments":{"source":"/path/to/large.pdf"}},"id":2}'

# Response (streamed events)
event: progress
data: {"status":"processing","percent":25}

event: progress
data: {"status":"processing","percent":75}

event: result
data: {"jsonrpc":"2.0","result":{...},"id":2}
```

**Claude Code Configuration (SSE)**:
```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "docling-mcp": {
      "type": "sse",
      "url": "http://hx-docling-mcp-server.hx.dev.local:8000/mcp"
    }
  }
}
```

### stdio Transport

Local process communication for desktop AI agents (Claude Desktop, LM Studio).

**Claude Desktop Configuration**:
```json
// ~/.config/claude-desktop/claude_desktop_config.json (Linux)
// ~/Library/Application Support/Claude/claude_desktop_config.json (macOS)
{
  "mcpServers": {
    "docling-mcp": {
      "command": "ssh",
      "args": [
        "agent0@hx-docling-mcp-server.hx.dev.local",
        "/opt/docling-mcp/venv/bin/python",
        "-m", "docling_mcp.server",
        "--transport", "stdio"
      ]
    }
  }
}
```

**LM Studio Configuration**:
```json
// mcp.json
{
  "servers": {
    "docling-mcp": {
      "type": "stdio",
      "command": "ssh",
      "args": ["agent0@hx-docling-mcp-server.hx.dev.local", "/opt/docling-mcp/venv/bin/python", "-m", "docling_mcp.server", "--transport", "stdio"]
    }
  }
}
```

### Recommended Transport by Client

| Client | Recommended Transport | Configuration |
|--------|----------------------|---------------|
| Claude Code | SSE | `~/.claude/settings.json` |
| Claude Desktop | stdio | `claude_desktop_config.json` |
| LM Studio | stdio | `mcp.json` |
| LangChain | HTTP | Python SDK |
| Custom agents | HTTP or SSE | API integration |
| n8n workflows | HTTP | REST node |

---

## Service Architecture

### Deployment Model

- **Type**: Bare-metal systemd service (no Docker)
- **OS**: Ubuntu 24.04 LTS
- **Python**: 3.11+ with virtual environment
- **Framework**: FastMCP 0.5.0

### Component Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP Clients                              │
│     (Claude Desktop, LM Studio, Custom Agents)              │
└─────────────────────────┬───────────────────────────────────┘
                          │ MCP Protocol (HTTP/SSE)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              hx-docling-mcp-server:8000                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                FastMCP Server                        │   │
│  │  ┌─────────────┬──────────────┬─────────────────┐   │   │
│  │  │ Conversion  │  Generation  │  Manipulation   │   │   │
│  │  │   Tools     │    Tools     │     Tools       │   │   │
│  │  └─────────────┴──────────────┴─────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │            Processing Pipeline                       │   │
│  │  ┌─────────────┬──────────────┬─────────────────┐   │   │
│  │  │   Docling   │   LightRAG   │      OCR        │   │   │
│  │  │  Processor  │  Processor   │   (EasyOCR)     │   │   │
│  │  └─────────────┴──────────────┴─────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┬─────────────────┐
         ▼                ▼                ▼                 ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   LiteLLM    │ │    Qdrant    │ │    Redis     │ │   LiteRAG    │
│   :4000      │ │    :6333     │ │    :6379     │ │    :8080     │
│  LLM Proxy   │ │Vector Store  │ │Session/Cache │ │Knowledge Grph│
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

### Directory Structure

```
/opt/docling-mcp/
├── venv/                    # Python virtual environment
├── application/             # Application code
│   ├── docling_mcp/        # Main package
│   │   ├── server.py       # FastMCP server entry point
│   │   ├── tools/          # MCP tool implementations
│   │   ├── processors/     # Docling, LightRAG, OCR processors
│   │   ├── clients/        # LiteLLM, Qdrant, Redis clients
│   │   └── models/         # Pydantic data models
│   └── requirements.txt    # Python dependencies
└── backups/                # Configuration backups

/etc/docling-mcp/
├── .env                    # Environment variables (secrets)
├── .env.template           # Template (no secrets)
└── logging.conf           # Logging configuration

/var/lib/docling-mcp/
├── cache/                  # Docling cache
├── workspace/              # Document processing workspace
└── lightrag/               # LightRAG working directory

/var/log/docling-mcp/
├── docling-mcp.log         # Main application log
├── error.log               # Error-level logs
└── access.log              # MCP request/response log
```

---

## MCP Tools Reference

### Conversion Tools (3 tools)

| Tool | Description | Input | Output |
|------|-------------|-------|--------|
| `convert_pdf` | Convert PDF to DoclingDocument | `source`: file path/URL | DoclingDocument JSON |
| `convert_docx` | Convert Word document | `source`: file path/URL | DoclingDocument JSON |
| `convert_url` | Convert web page | `url`: HTTP/HTTPS URL | DoclingDocument JSON |

### Generation Tools (11 tools)

| Tool | Description |
|------|-------------|
| `generate_knowledge_graph` | Extract entities and relationships from document |
| `generate_title` | Generate document title |
| `generate_toc` | Generate table of contents |
| `generate_section` | Create document section |
| `generate_heading` | Create heading element |
| `generate_paragraph` | Create paragraph element |
| `generate_list` | Create list element |
| `generate_table` | Create table element |
| `generate_image` | Insert image reference |
| `generate_codeblock` | Create code block |
| `generate_reference` | Create cross-reference |

### Manipulation Tools (5 tools)

| Tool | Description |
|------|-------------|
| `split_document` | Split document into sections |
| `merge_documents` | Merge multiple documents |
| `export_markdown` | Export to Markdown format |
| `export_html` | Export to HTML format |
| `export_json` | Export to JSON format |

### Usage Example (MCP Protocol)

```bash
# List available tools
curl -X POST http://hx-docling-mcp-server.hx.dev.local:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'

# Convert a PDF document
curl -X POST http://hx-docling-mcp-server.hx.dev.local:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"convert_pdf","arguments":{"source":"/path/to/document.pdf"}},"id":2}'
```

---

## LLM Models & Processing

### On-Premises Model Architecture

All document processing and LLM inference uses **on-premises Ollama servers** - no cloud LLM calls.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    hx-litellm-server:4000                           │
│                   (LLM Gateway - 192.168.10.212)                    │
│              Routes all LLM requests to Ollama servers              │
└───────────┬───────────────────┬───────────────────┬─────────────────┘
            │                   │                   │
            ▼                   ▼                   ▼
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────────┐
│  hx-ollama1-server│ │  hx-ollama2-server│ │   hx-ollama3-server   │
│   192.168.10.204  │ │   192.168.10.205  │ │    192.168.10.206     │
│   General Purpose │ │   Code Models     │ │  Document & Embedding │
├───────────────────┤ ├───────────────────┤ ├───────────────────────┤
│ • gpt-oss:20b ◄── │ │ • qwen3-coder:30b │ │ • ibm/granite-docling │
│   Entity Extract  │ │ • qwen2.5:7b      │ │   :258m ◄── Doc VLM   │
│ • gemma3:27b      │ │ • cogito:3b       │ │ • bge-m3:567m ◄──     │
│ • mistral:7b      │ │                   │ │   Embeddings (1024D)  │
│                   │ │                   │ │ • bge-reranker-v2-m3  │
└───────────────────┘ └───────────────────┘ └───────────────────────┘
```

### IBM Granite Docling VLM

**Primary document processing model** - a purpose-built Vision Language Model (VLM) for enterprise document understanding.

| Property | Value |
|----------|-------|
| **Model** | `ibm/granite-docling:258m` |
| **Location** | hx-ollama3-server.hx.dev.local:11434 |
| **Size** | 258M parameters (521 MB) |
| **License** | Apache 2.0 |
| **Output Format** | DocTags → Markdown/HTML/JSON |

**Capabilities:**
- Single unified model replaces traditional multi-stage OCR pipeline
- Integrates OCR, layout analysis, and content extraction in one pass
- Purpose-built DocTags markup preserves tables, code blocks, math, hierarchy
- Optimized for Latin scripts + Japanese, Chinese, Arabic support

**Usage via Docling CLI:**
```bash
docling --to md --pipeline vlm --vlm-model granite_docling "/path/to/document.pdf"
```

**Reference:** [IBM Granite Docling Documentation](https://www.ibm.com/granite/docs/models/docling)

### Models by Task

| Task | Model | Server | Description |
|------|-------|--------|-------------|
| **Document Processing** | `ibm/granite-docling:258m` | hx-ollama3-server | VLM for PDF/image → structured content |
| **Entity Extraction** | `gpt-oss:20b` | hx-ollama1-server | Knowledge graph entity extraction |
| **Embedding Generation** | `bge-m3:567m` | hx-ollama3-server | 1024D dense vectors for Qdrant |
| **Reranking** | `bona/bge-reranker-v2-m3` | hx-ollama3-server | Search result reranking |

### Embedding Specifications

| Property | Value |
|----------|-------|
| Model | `bge-m3:567m` |
| Server | hx-ollama3-server.hx.dev.local:11434 |
| Dimensions | 1024 |
| Distance Metric | Cosine |
| Batch Size | 32 |
| Storage | Qdrant collections |

### LiteLLM Model Routing

All LLM calls route through hx-litellm-server which proxies to the appropriate Ollama server:

| LiteLLM Model ID | Routes To | Purpose |
|------------------|-----------|---------|
| `ollama/gpt-oss:20b` | hx-ollama1-server | Entity extraction, general LLM |
| `ollama/ibm/granite-docling:258m` | hx-ollama3-server | Document VLM processing |
| `ollama/bge-m3:567m` | hx-ollama3-server | Embedding generation |

### Configuration from .env.production

```env
# From /opt/docling-mcp/.env.production (verified)
LITELLM_API_BASE=http://192.168.10.212:4000
LITELLM_TIMEOUT_SECONDS=60
ENTITY_EXTRACTION_MODEL=litellm
MAX_DOCUMENT_SIZE_MB=100
OCR_ENABLED=true
OCR_LANGUAGE=eng
CHUNK_SIZE=4096
CHUNK_OVERLAP=512
```

### Model Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| Temperature | 0.1 | Deterministic output for extraction |
| Max Tokens | 8192 | Granite Docling generation limit |
| Timeout | 60s | Per-request timeout |
| Retry | 3 attempts | Exponential backoff |

---

## Integration Endpoints

### Upstream Dependencies

| Service | Hostname | Port | Purpose | Required |
|---------|----------|------|---------|----------|
| LiteLLM | hx-litellm-server.hx.dev.local | 4000 | LLM gateway routing to Ollama | Yes |
| **Ollama3** | hx-ollama3-server.hx.dev.local | 11434 | **Granite Docling VLM + Embeddings** | **Yes** |
| Qdrant | hx-qdrant-server.hx.dev.local | 6333 | Vector storage for embeddings | Yes |
| Redis | hx-redis-server.hx.dev.local | 6379 | Session management and caching | Optional |
| LiteRAG | hx-literag-server.hx.dev.local | 8080 | Knowledge graph extraction | Yes |

**Note:** Document processing uses `ibm/granite-docling:258m` on hx-ollama3-server exclusively.

### Connection Configuration

```env
# LiteLLM Integration
LITELLM_BASE_URL=http://hx-litellm-server.hx.dev.local:4000
LITELLM_API_KEY=<from vault>
LLM_ENTITY_EXTRACTION_MODEL=gemma3:27b

# Qdrant Integration
QDRANT_HOST=hx-qdrant-server.hx.dev.local
QDRANT_PORT=6333
QDRANT_API_KEY=<from vault>
QDRANT_COLLECTION_ENTITIES=hx_docling_mcp_entities
QDRANT_COLLECTION_RELATIONSHIPS=hx_docling_mcp_relationships

# Redis Integration
REDIS_HOST=hx-redis-server.hx.dev.local
REDIS_PORT=6379
REDIS_PASSWORD=<from vault>
REDIS_SESSION_TTL_HOURS=24

# LiteRAG Integration  
LITERAG_BASE_URL=http://hx-literag-server.hx.dev.local:8080
```

---

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_HOST` | 0.0.0.0 | Server bind address |
| `MCP_PORT` | 8000 | Server port |
| `MCP_WORKERS` | 4 | Uvicorn workers |
| `LOG_LEVEL` | INFO | Logging level |
| `DOCLING_CACHE_DIR` | /var/lib/docling-mcp/cache | Docling cache path |
| `OCR_ENABLED` | true | Enable OCR for scanned documents |
| `OCR_LANGUAGES` | eng | Tesseract language codes |

### Service Account

- **User**: `docling-mcp@hx.dev.local` (Samba AD)
- **Group**: `domain users@hx.dev.local`
- **Home**: `/opt/docling-mcp`

### Credentials (Ansible Vault)

Location: `/home/agent0/HX-Infrastructure/nodes/hx-docling-mcp-server/vault/`

```bash
# View credentials (requires vault password)
ansible-vault view vault/hx-docling-mcp-credentials.yml
```

---

## Administration

### Service Management

```bash
# Check service status
ssh agent0@hx-docling-mcp-server.hx.dev.local 'systemctl status docling-mcp'

# Start service
ssh agent0@hx-docling-mcp-server.hx.dev.local 'sudo systemctl start docling-mcp'

# Stop service
ssh agent0@hx-docling-mcp-server.hx.dev.local 'sudo systemctl stop docling-mcp'

# Restart service
ssh agent0@hx-docling-mcp-server.hx.dev.local 'sudo systemctl restart docling-mcp'

# Enable on boot
ssh agent0@hx-docling-mcp-server.hx.dev.local 'sudo systemctl enable docling-mcp'

# View logs (live tail)
ssh agent0@hx-docling-mcp-server.hx.dev.local 'journalctl -u docling-mcp -f'

# View recent logs
ssh agent0@hx-docling-mcp-server.hx.dev.local 'journalctl -u docling-mcp -n 100 --no-pager'
```

### Restart Policy

- **Restart**: on-failure
- **RestartSec**: 10 seconds
- **StartLimitBurst**: 3 attempts
- **StartLimitInterval**: 60 seconds

---

## Monitoring & Health

### Health Check Endpoint

```bash
# HTTP health check
curl http://hx-docling-mcp-server.hx.dev.local:8000/health

# Expected response
{"status": "healthy", "version": "1.0.0", "uptime": 12345}
```

### Dependency Connectivity Test

```bash
# Test LiteLLM
curl -s http://hx-litellm-server.hx.dev.local:4000/health

# Test Qdrant
curl -s http://hx-qdrant-server.hx.dev.local:6333/health

# Test Redis
redis-cli -h hx-redis-server.hx.dev.local ping

# Test LiteRAG
curl -s http://hx-literag-server.hx.dev.local:8080/health
```

### Key Metrics

| Metric | Location | Alert Threshold |
|--------|----------|-----------------|
| Service Status | systemctl status | != active |
| Health Endpoint | :8000/health | != 200 OK |
| Error Log | /var/log/docling-mcp/error.log | Any entries |
| Memory Usage | htop / free -m | > 80% |
| Disk Usage | df -h /var/lib/docling-mcp | > 90% |

---

## Troubleshooting

### Common Issues

**Service Won't Start**
```bash
# Check logs for startup errors
journalctl -u docling-mcp -n 50 --no-pager

# Verify dependencies are reachable
curl -s http://hx-litellm-server.hx.dev.local:4000/health
curl -s http://hx-qdrant-server.hx.dev.local:6333/health
```

**MCP Tools Not Responding**
```bash
# Test MCP endpoint directly
curl -X POST http://hx-docling-mcp-server.hx.dev.local:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

**Document Conversion Fails**
```bash
# Check Docling cache permissions
ls -la /var/lib/docling-mcp/cache/

# Check disk space
df -h /var/lib/docling-mcp/
```

**OCR Not Working**
```bash
# Verify Tesseract installed
tesseract --version

# Check EasyOCR models
ls -la /var/lib/docling-mcp/cache/easyocr/
```

---

## Related Documentation

### Node Documentation

| Document | Location | Description |
|----------|----------|-------------|
| Charter | `nodes/hx-docling-mcp-server/charter/charter.md` | Project vision and scope |
| Specification | `nodes/hx-docling-mcp-server/specification/node-spec.md` | Full service specification |
| Architecture | `nodes/hx-docling-mcp-server/planning/deployment-architecture.md` | Technical design |
| Configuration | `nodes/hx-docling-mcp-server/planning/configuration-spec.md` | Detailed configuration |
| Test Plan | `nodes/hx-docling-mcp-server/tests/test-plan.md` | Test strategy and cases |

### Claude Code Integration

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "docling-mcp": {
      "type": "sse",
      "url": "http://hx-docling-mcp-server.hx.dev.local:8000/mcp"
    }
  }
}
```

### HX-Infrastructure References

- **Node Inventory**: `/home/agent0/HX-Infrastructure/inventory/nodes.md`
- **Naming Standards**: `/home/agent0/HX-Infrastructure/standards/naming-conventions.md`
- **Deployment Requirements**: `/home/agent0/HX-Infrastructure/standards/deployment-requirements.md`

---

**Last Updated**: 2025-12-04  
**Maintained By**: HX-Infrastructure Team
