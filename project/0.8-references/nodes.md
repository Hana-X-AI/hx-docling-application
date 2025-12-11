# HX-Infrastructure Platform Nodes – Capabilities & Status

**Document Type**: Current State Documentation - Infrastructure Inventory  
**Created**: 2025-11-05  
**Last Updated**: 2025-12-11  
**Status**: ACTIVE - Authoritative Infrastructure Baseline  
**Location**: `/home/agent0/HX-Infrastructure/inventory/nodes.md`

---

## Document Purpose

This document provides a comprehensive inventory of all server nodes in HX-Infrastructure, including their capabilities, responsibilities, operational status, and deployment details. This is a **snapshot of actual production infrastructure** as of the last update date.

### Document Classification

**Type**: Current State Documentation - Infrastructure Inventory
- ✅ Represents actual deployed infrastructure
- ✅ Specific server configurations and status are production values
- ✅ Updated as servers are deployed or decommissioned
- ⚠️ For node documentation template, see `templates/node-template.md` (when created)

**Status Legend**: 
- ✅ **Operational** - Deployed, tested, and validated for production use
- 🛠️ **In Progress** - Deployed but not yet fully validated/configured
- ⬜ **Planned** - To be deployed (design complete, awaiting implementation)
- ⚠️ **Reserved** - IP allocated, not yet deployed

---

## Identity, Trust, and Control

### hx-dc-server (192.168.10.200) – ✅ Operational
**Role:** Domain Controller & Authentication Hub (Samba/LDAP, Kerberos) for `hx.dev.local`.  
**Primary Responsibilities:** 
- Central user/group management
- Single Sign-On (SSO) for Open WebUI and internal applications
- Policy enforcement across infrastructure
- DNS services for hx.dev.local domain

**Data/Paths:** 
- `/var/lib/samba` - Samba domain data
- `/etc/samba` - Samba configuration

**Integration Points:**
- All servers authenticate via this DC
- DNS resolution for all hx.dev.local hosts

**Validation Status:** ✅ All services operational, domain joined servers verified

---

### hx-ca-server (192.168.10.201) – ✅ Operational
**Role:** Internal Certificate Authority (Root CA + Issuing CA).  
**Primary Responsibilities:** 
- Issue and renew TLS certificates for all internal services
- Maintain Certificate Revocation List (CRL) and OCSP responder
- Provide SSH certificate trust foundation
- Certificate lifecycle management

**Data/Paths:** 
- `/var/ca/private` - Private keys (secured)
- `/var/ca/certs` - Issued certificates
- `/var/ca/crl` - Certificate revocation lists

**Integration Points:**
- All HTTPS services use certificates from this CA
- hx-ssl-server retrieves certificates for reverse proxy

**Validation Status:** ✅ Certificate issuance tested, trust chain verified

---

### hx-ssl-server (192.168.10.202) – ✅ Operational
**Role:** Reverse Proxy / TLS Termination (Nginx/Traefik).  
**Primary Responsibilities:** 
- Front all user-facing UIs and APIs
- Enforce HTTPS and mutual TLS (mTLS) where required
- Route traffic to application backends
- Load balancing (when multiple backends available)

**Data/Paths:** 
- `/etc/nginx` or `/etc/traefik` - Proxy configuration
- `/etc/ssl/hx` - TLS certificates from hx-ca-server

**Integration Points:**
- Terminates TLS for hx-webui-server, hx-agui-server, hx-demo-server
- Routes to backend services

**Validation Status:** ✅ Reverse proxy operational, HTTPS routing verified

---

### hx-control-node (192.168.10.203) – ✅ Operational
**Role:** Ansible Control Plane.  
**Primary Responsibilities:** 
- Fleet-wide configuration management
- Idempotent server provisioning
- Secrets management via Ansible Vault
- Infrastructure-as-Code deployment automation

**Data/Paths:** 
- `/srv/ansible/inventories` - Server inventories
- `/srv/ansible/roles` - Ansible roles
- `/srv/ansible/playbooks` - Deployment playbooks
- `/srv/ansible/group_vars` - Group variables

**Integration Points:**
- SSH access to all servers for configuration management
- Manages deployment of services across infrastructure

**Validation Status:** ✅ Ansible automation operational, fleet management verified

---

## Model Serving and Inference Mesh

### hx-ollama{1,2,3}-server (192.168.10.204–206) – ✅ Operational
**Role:** LLM Model Hosts (Ollama).  
**Primary Responsibilities:**
- **hx-ollama1-server (.204):** Primary LLM serving node for general inference workloads
- **hx-ollama2-server (.205):** Code-focused model serving (specialized models for code generation/analysis)
- **hx-ollama3-server (.206):** Embeddings host **and** prompt-enhancement model

**Data/Paths:** 
- `/var/lib/ollama` - Model storage and cache

**Architecture Notes:** 
- The prompt-enhancement model on **ollama3** bypasses LiteLLM for low-latency preprocessing
- Connects **directly to UI-layer apps** (e.g., Open WebUI, custom applications)
- Other model traffic routes via **hx-litellm-server** for unified API gateway

**Integration Points:**
- hx-litellm-server proxies requests to all Ollama servers
- hx-webui-server connects directly to ollama3 for prompt enhancement

**Validation Status:** ✅ All three Ollama servers operational, models loaded and tested

---

### hx-litellm-server (192.168.10.212) – ✅ Operational
**Role:** Unified LLM API Gateway.  
**Primary Responsibilities:** 
- Route requests to appropriate Ollama nodes
- Broker tool calls via Model Context Protocol (MCP)
- Authentication and authorization integration
- Load balancing across model servers
- Request/response logging and metrics

**Data/Paths:** 
- `/etc/litellm` - Configuration files
- `/var/log/litellm/` - Request/response logs

**Integration Points:**
- Frontend: hx-webui-server, application servers
- Backend: hx-ollama1/2/3-server
- MCP: hx-fastmcp-server for tool execution

**Validation Status:** ✅ API gateway operational, integrated with Open WebUI

---

## Data Plane: Structured, Cache, and Vectors

### hx-postgres-server (192.168.10.209) – ✅ Operational
**Role:** System-of-Record Relational Database.  
**Primary Responsibilities:** 
- Application state persistence
- Metadata storage
- Audit logging
- Write-Ahead Log (WAL) archiving for point-in-time recovery

**Data/Paths:** 
- `/var/lib/postgresql/data` - Database cluster
- WAL archive configured for backup/recovery

**Integration Points:**
- Used by: hx-docling-server, hx-crawl4ai-server, hx-literag-server, application servers

**Validation Status:** ✅ Database operational, WAL archiving configured

---

### hx-redis-server (192.168.10.210) – ✅ Operational
**Role:** Redis Cache Server with Web UI.  
**Primary Responsibilities:** 
- Low-latency caching layer
- Message queues for async processing
- Rate limiting and session storage
- Redis UI provides graphical interface (eliminates CLI-only management)

**Data/Paths:** 
- `/var/lib/redis` - Persistence data (RDB/AOF)

**Ports:**
- 6379 - Redis server
- 8001 - Redis UI web interface

**Integration Points:**
- Used by: Application servers, hx-lang-server (when operational)

**Validation Status:** ✅ Redis operational, Redis UI accessible

---

### hx-qdrant-server (192.168.10.207) – ✅ Operational
**Role:** Vector Database.  
**Primary Responsibilities:** 
- Store embeddings for semantic search
- RAG (Retrieval-Augmented Generation) indices
- Vector similarity search
- Snapshots aligned with Postgres backups

**Data/Paths:** 
- `/var/lib/qdrant` - Vector collections and indices

**Ports:**
- 6333 - REST API
- 6334 - gRPC API

**Integration Points:**
- Frontend: hx-qmcp-server (MCP interface)
- Backend: hx-literag-server, application services

**Validation Status:** ✅ Vector database operational, collections verified

---

### hx-qdrant-ui-server (192.168.10.208) – ✅ Operational
**Role:** Qdrant Web UI.  
**Primary Responsibilities:** 
- Graphical interface for managing Qdrant vector databases
- Support for both self-hosted and cloud Qdrant instances
- View, interact with, and manage vector collections
- Similar to Kibana for Elasticsearch

**Port:** 3000 - Web UI

**Integration Points:**
- Connects to hx-qdrant-server for visualization

**Validation Status:** ✅ Web UI operational, connected to Qdrant server

---

### hx-qmcp-server (192.168.10.211) – ✅ Operational
**Role:** Qdrant Model Context Protocol (MCP) Server.  
**Primary Responsibilities:** 
- Connect AI agents to Qdrant vector database
- Provide persistent, semantic memory for agents
- Expose Qdrant's vector search capabilities through standardized MCP interface
- Enable agents to store and retrieve embeddings

**Integration Points:**
- MCP Gateway: hx-fastmcp-server
- Vector DB: hx-qdrant-server
- Clients: AI agents via MCP protocol

**Validation Status:** ✅ MCP server operational, Qdrant integration verified

---

## Agentic + Toolchain

> **Note:** `hx-fastmcp-server` includes a **Brave Search MCP** integration (plugin/internal client) that enables secure web and document search, exposed through FastMCP's unified interface.

### hx-fastmcp-server (192.168.10.213) – ✅ Operational
**Includes:** Brave Search MCP client for secure, context-aware web search and retrieval.  
**Role:** High-throughput MCP **server + client** (unified entry point).  
**Primary Responsibilities:** 
- Execute tools at scale for AI agents
- Expose standardized MCP endpoints
- **Compose and proxy** downstream MCP services within single Python application
- Route MCP requests to specialized services (Qdrant, Docling, Crawl4AI, etc.)

**Port:** 8000 - MCP Gateway

**Integration Points:**
- Frontend: hx-webui-server, application servers
- Backend MCP Services: hx-qmcp-server, hx-docling-mcp-server (when operational), hx-crawl4ai-mcp-server, hx-n8n-mcp-server
- Search: Internal Brave Search MCP client

**Validation Status:** ✅ MCP gateway operational, tool execution verified

---

### hx-n8n-mcp-server (192.168.10.214) – ✅ Operational
**Role:** n8n Model Context Protocol (MCP) Server.  
**Primary Responsibilities:** 
- Bridge between LLMs and n8n workflows
- Allow AI assistants to securely interact with and control n8n workflows through standardized MCP interface
- Enable agents to leverage n8n's visual, low-code automation for real-world task execution
- Workflow triggering and monitoring via MCP

**Data/Paths:** `/srv/n8n-mcp-deployment`

**Deployment Date:** November 11, 2025

**Integration Points:**
- MCP Gateway: hx-fastmcp-server
- Workflow Engine: hx-n8n-server
- Clients: AI agents via MCP protocol

**Validation Status:** ✅ MCP server operational, n8n integration verified

---

### hx-n8n-server (192.168.10.215) – ✅ Operational
**Role:** Workflow Automation and Orchestration Platform.  
**Primary Responsibilities:** 
- Visual workflow builder for integrating apps and automating business processes
- Job scheduling and execution
- CI/CD workflow automation
- Notification and alert management
- Data synchronization across systems

**Data/Paths:** 
- `/srv/n8n` - Workflows, credentials, execution data

**Port:** 5678 - Web UI and API

**Deployment Date:** November 15, 2025

**Integration Points:**
- MCP Interface: hx-n8n-mcp-server
- External APIs: Various third-party integrations
- Database: Uses PostgreSQL for persistence (via hx-postgres-server)

**Validation Status:** ✅ Operational - Login tested, workflows executable

---

### hx-docling-server (192.168.10.216) – ✅ Operational
**Role:** Docling Worker Node.  
**Primary Responsibilities:** 
- Internal and external document retrieval
- Document parsing (PDF, DOCX, etc.)
- Document format conversion
- Text extraction and preprocessing

**Integration Points:**
- MCP Interface: hx-docling-mcp-server (operational)
- Database: hx-postgres-server (document metadata storage)

**Validation Status:** ✅ Worker and MCP interface operational, document processing verified

---

### hx-docling-mcp-server (192.168.10.217) – ✅ Operational
**Role:** Docling MCP Endpoint.  
**Primary Responsibilities:** 
- Expose document parsing and ingestion capabilities to AI agents via MCP
- Provide standardized interface for document processing requests
- Queue and manage document processing jobs

**Integration Points:**
- MCP Gateway: hx-fastmcp-server
- Worker: hx-docling-server
- Clients: AI agents via MCP protocol

**Deployment Date:** 2025-12-11
**Validation Status:** ✅ Operational - MCP interface deployed and validated, document processing verified

---

### hx-crawl4ai-mcp-server (192.168.10.218) – ✅ Operational
**Role:** Crawl4AI MCP Endpoint.  
**Primary Responsibilities:** 
- Provide controlled web scraping capabilities through MCP for AI agents
- Request validation and rate limiting
- Crawl job management and monitoring
- Results formatting and delivery

**Integration Points:**
- MCP Gateway: hx-fastmcp-server
- Worker: hx-crawl4ai-server
- Clients: AI agents via MCP protocol

**Validation Status:** ✅ MCP server operational, crawl requests verified

---

### hx-crawl4ai-server (192.168.10.219) – ✅ Operational
**Role:** Crawl4AI Worker Node.  
**Primary Responsibilities:** 
- Execute web crawling for internal and external sites
- Corpus gathering for knowledge base construction
- Structured data extraction from web pages
- Rate-limited, respectful crawling

**Integration Points:**
- MCP Interface: hx-crawl4ai-mcp-server
- Database: hx-postgres-server (crawled content storage)

**Validation Status:** ✅ Worker operational, crawling functional

---

### hx-literag-server (192.168.10.220) – ✅ Operational
**Role:** LightRAG Server.  
**Primary Responsibilities:** 
- Efficient Retrieval-Augmented Generation (RAG) framework
- Build knowledge graph of relationships between entities and concepts
- Dual-level retrieval:
  - Semantic vector search (via Qdrant)
  - Knowledge graph context
- Generate comprehensive and relevant answers combining both retrieval methods

**Integration Points:**
- Vector DB: hx-qdrant-server (embeddings and semantic search)
- Database: hx-postgres-server (knowledge graph storage)
- Clients: Application servers, AI agents

**Validation Status:** ✅ RAG framework operational, dual-level retrieval verified

---

### hx-mem0-mcp-server (192.168.10.231) – ✅ Operational
**Role:** Mem0 MCP Server - AI Agent Memory Layer.
**Primary Responsibilities:**
- Intelligent, persistent memory for AI agents via MCP protocol
- 8 MCP tools: mem0_add, mem0_search, mem0_get, mem0_get_all, mem0_update, mem0_delete, mem0_history, mem0_reset
- Hybrid isolation model (user_id + agent_id scoping)
- Semantic memory search with automatic memory extraction

**Port:** 8000 - MCP HTTP transport

**Technology Stack:**
- mem0ai v1.0.1 (Memory SDK)
- FastMCP v2.13.3 (MCP Framework)
- Python 3.12.3
- Uvicorn ASGI server

**Data/Paths:**
- `/opt/hx-mem0-mcp-server/` - Application home
- `/opt/hx-mem0-mcp-server/venv/` - Python virtual environment
- `/var/log/hx-mem0-mcp-server/` - Application logs (via journald)

**Service Account:** mem0-svc (uid=999)

**Integration Points:**
- LLM: hx-litellm-server:4000 → hx-ollama1-server (gpt-oss:20b for memory extraction)
- Embeddings: hx-ollama3-server:11434 (bge-m3:567m, 1024 dims)
- Vector Storage: hx-qdrant-server:6333 (mem0 collection)
- Consumers: Claude Code, Open WebUI, LangGraph agents, any MCP client

**Access Points:**
- HTTP API: http://192.168.10.231:8000/mcp
- Health Check: http://192.168.10.231:8000/health
- DNS: http://hx-mem0-mcp-server.hx.dev.local:8000

**Deployment Date:** 2025-12-08
**CAIO Approval:** 2025-12-08 (Jarvis Richardson)

**Documentation:**
- Charter: `nodes/hx-mem0-mcp-server/charter/charter.md`
- Specification: `nodes/hx-mem0-mcp-server/specification/node-spec.md`
- Service Descriptor: `services/operational/hx-mem0-mcp-server/service-descriptor.md`

**Validation Status:** ✅ Operational - 44/44 tests passed (100%), all quality gates passed

---

### hx-coderabbit-server (192.168.10.228) – ❌ Repurposed
**Role:** (DECOMMISSIONED) - Hardware repurposed for hx-mem0-mcp-server.
**Status:** Repurposed - Server hardware at 192.168.10.228 was repurposed to hx-mem0-mcp-server (192.168.10.231) on 2025-12-08.

**Note:** If CodeRabbit MCP Server is needed in the future, a new server will need to be provisioned.

**Repurpose Date:** 2025-12-08
**Repurposed To:** hx-mem0-mcp-server (192.168.10.231)

**Validation Status:** ❌ Decommissioned - Hardware repurposed

---

### hx-shadcn-server (192.168.10.229) – ✅ Operational
**Role:** Shadcn MCP Server - AI-Powered UI Component Discovery and Integration.
**Primary Responsibilities:**
- Bridge between AI agents and component registries (shadcn/ui, radix, etc.)
- Expose 12 MCP tools for component operations (list_registries, list_components, get_component_details, get_component_installation_command, search_components, get_registry_info, set_framework, get_framework, validate_component, check_dependencies, list_categories, get_component_files)
- Enable structured component discovery with multi-registry support
- Component integration automation for frontend development workflows
- Framework-aware operations (Next.js, Vite, Remix, Gatsby, Laravel)

**Technology Stack:**
- Node.js 20.x (LTS)
- @anthropic/shadcn-mcp-server (npm package)
- MCP Protocol (HTTP transport)

**Port:** 8000 - MCP HTTP transport

**Data/Paths:**
- `/opt/hx-shadcn-server/` - Application home
- `/etc/hx-shadcn-server/` - Configuration directory
- Environment configuration via systemd EnvironmentFile

**Service Account:** shadcn-svc (uid=998, gid=998)

**Integration Points:**
- MCP Gateway: hx-fastmcp-server (when integrated)
- Component Registries: shadcn/ui, radix, magic-ui, plate, aceternity
- Frontend Development: hx-dev-server project workflows
- Consumers: Claude Code, AI agents via MCP protocol

**Access Points:**
- HTTP API: http://192.168.10.229:8000/
- Health Check: http://192.168.10.229:8000/health
- DNS: http://hx-shadcn-server.hx.dev.local:8000

**Deployment Date:** 2025-12-09
**CAIO Approval:** 2025-12-09 (Jarvis Richardson)

**Configuration Notes:**
- Environment variables: SHADCN_REGISTRY_URL, SHADCN_CACHE_TTL, SHADCN_GITHUB_TOKEN, SHADCN_LOG_LEVEL
- systemd service with MemoryDenyWriteExecute=false (required for Node.js JIT)
- 1-hour cache TTL for registry data
- Authenticated GitHub API access for rate limit management

**Documentation:**
- Charter: `nodes/hx-shadcn/charter/charter.md`
- Specification: `nodes/hx-shadcn/specification/node-spec.md`
- Service Descriptor: `services/operational/hx-shadcn-server/service-descriptor.md`

**Validation Status:** ✅ Operational - 67 test cases defined, all 26 deployment tasks completed

---

## Application and User-Facing Layers

### hx-webui-server (192.168.10.227) – ✅ Operational
**Role:** Open WebUI for Chat/Agent User Experience.  
**Primary Responsibilities:** 
- Provide full MCP tool access to end users
- User authentication via Domain Controller/Kerberos
- Chat interface for AI agent interactions
- Real-time streaming of agent responses
- Session management and conversation history

**Port:** 3000 - Web UI (behind hx-ssl-server reverse proxy)

**Integration Points:**
- Auth: hx-dc-server (Kerberos SSO)
- LLM Gateway: hx-litellm-server
- Direct LLM: hx-ollama3-server (prompt enhancement bypass)
- MCP Tools: hx-fastmcp-server
  - Docling/Crawl4AI document processing
  - Qdrant via hx-qmcp-server
- Ingress: hx-ssl-server (TLS termination)

**Validation Status:** ✅ Operational - Integrated with LiteLLM, MCP tools accessible

---

### hx-agui-server (192.168.10.221) – ✅ Operational
**Role:** AG-UI (Agent-User Interaction Protocol) Application Server.
**Primary Responsibilities:**
- Implement AG-UI protocol for standardized agent-user interactions (26 event types)
- Server-Sent Events (SSE) real-time streaming
- Real-time agentic chat with streaming responses
- Bi-directional state synchronization (STATE_SNAPSHOT, STATE_DELTA)
- Human-in-the-Loop (HITL) workflows with PostgreSQL checkpoint persistence
- LangGraph agent integration via ag-ui-langgraph

**Technology Stack:**
- Python 3.12.3 runtime
- FastAPI + uvicorn (ASGI server)
- ag-ui-protocol 0.1.10
- ag-ui-langgraph 0.0.15+
- langgraph-checkpoint-postgres 1.0.0+
- React Demo Application (@ag-ui/client)

**Port:** 8420 (AG-UI Protocol API)

**Data/Paths:**
- `/opt/hx-agui-server/` - Application home
- `/opt/hx-agui-server/app/` - Python application
- `/opt/hx-agui-server/static/demo/` - React demo application
- `/var/log/hx-agui-server/` - Application logs

**Service Account:** hx-agui-server (uid=999, gid=988)

**Integration Points:**
- LangGraph Agents: hx-lang-server:8100 (LangGraph agent execution)
- Checkpoints: hx-postgres-server:5432 (database: hx_agui_server, schema: agui_checkpoints)
- SSL Termination: hx-ssl-server (nginx reverse proxy at /agui/)
- Demo UI: https://hx-ssl-server.hx.dev.local/agui/demo/

**Access Points:**
- Direct HTTP: http://hx-agui-server.hx.dev.local:8420/
- SSL via Proxy: https://hx-ssl-server.hx.dev.local/agui/
- Demo Application: https://hx-ssl-server.hx.dev.local/agui/demo/
- Health Check: http://hx-agui-server.hx.dev.local:8420/health

**Deployment Date:** 2025-12-06
**Related Documentation:**
- Specification: `nodes/hx-agui-server/specification/node-spec.md`
- Charter: `nodes/hx-agui-server/charter/charter.md`
- Service Descriptor: `services/operational/hx-agui-server/service-descriptor.md`
- Project Closeout: `nodes/hx-agui-server/specification/status-reports/project-closeout-20251206.md`

**Validation Status:** ✅ All 5 charter success criteria passed, 89 test cases executed

---

### hx-dev-server (192.168.10.222) – ✅ Operational
**Role:** AI/ML Development Server - Docker-based Development Environment.
**Primary Responsibilities:**
- Isolated Docker-based development environment for AI/ML projects
- AD-authenticated developer access via SSH and Portainer
- Pre-configured connectivity to all HX AI services (LiteLLM, Qdrant, PostgreSQL, Redis, FastMCP)
- Wildcard subdomain routing for dev containers (*.dev.hx.dev.local)
- Standardized Dockerfile templates (Python, Node.js) with CA trust and HX environment variables

**Technology Stack:**
- Docker Engine 29.1.2 (data-root on /mnt/docker-data/docker)
- Docker Compose 5.0.0
- Portainer CE (HTTPS on port 9443)
- Ubuntu 24.04 LTS host
- SSSD for AD authentication

**Data/Paths:**
- `/mnt/docker-data/docker` - Docker data root (894GB available)
- `/srv/docker-templates` - Dockerfile templates (Python, Node.js)
- Container networking via bridge networks

**Integration Points:**
- Domain Auth: hx-dc-server (AD via SSSD)
- PKI: hx-ca-server (CA trust mounted into containers)
- Reverse Proxy: hx-ssl-server (*.dev.hx.dev.local wildcard routing)
- LLM: hx-litellm-server:4000 (HTTP)
- Vector DB: hx-qdrant-server:6333 (HTTP/gRPC)
- Database: hx-postgres-server:5432 (TCP)
- Cache: hx-redis-server:6379 (TCP)
- MCP: hx-fastmcp-server:8000 (HTTP)

**Access Points:**
- SSH: `ssh username@192.168.10.222` (AD credentials)
- Portainer: `https://192.168.10.222:9443` (admin credentials in Ansible Vault)
- Dev Containers: `*.dev.hx.dev.local` (via reverse proxy)

**Deployment Date:** 2025-12-06
**CAIO Approval:** 2025-12-06 (Jarvis Richardson)

**Validation Status:** ✅ Operational - All 128 tests passed, all 7 quality gates passed, all 11 success criteria validated

---

### hx-demo-server (192.168.10.223) – ✅ Operational
**Role:** Demo Environment for Stakeholder Presentations.  
**Primary Responsibilities:** 
- Receive promoted containers from hx-dev-server
- Host stable application versions for demonstrations
- Mirror production platform integrations
- Provide controlled environment for stakeholder access

**Data/Paths:** 
- `/srv/apps/demo` - Promoted application containers
- Container registry integration operational

**Integration Points:**
- Same as hx-dev-server (LiteLLM, MCP, LangGraph, etc.)
- Receives promoted builds from dev environment
- Ingress: hx-ssl-server

**Deployment Date:** 2025-12-11
**Validation Status:** ✅ Operational - Container promotion workflow verified, demo environment accessible

---

## Integration, Coordination, and Observability

### hx-cc-server (192.168.10.224) – ✅ Operational
**Role:** Claude Code Systems Integrator & Knowledge Hub.  
**Primary Responsibilities:** 
- Coordinate agent execution runs
- Manage local knowledge repositories
- Infrastructure orchestration via Claude Code
- System integration and coordination

**Authoritative Storage (Updated Paths):**
- `/home/agent0/HX-Infrastructure` - Main infrastructure repository
- `/home/agent0/HX-Infrastructure/hx-knowledge/repos` - Knowledge vault
- `/home/agent0/HX-Infrastructure/hx-knowledge/docs` - Documentation

**Integration Points:**
- All infrastructure nodes (orchestration)
- Knowledge management across services
- Agent coordination

**Validation Status:** ✅ Operational - Claude Code integration functional

---

### hx-metric-server (192.168.10.225) – 🧪 Test & Verify
**Role:** Metrics and Telemetry Collection.  
**Primary Responsibilities:** 
- Centralized logging aggregation
- Metrics collection (Prometheus)
- Long-term retention storage
- Grafana dashboards for visualization
- Alerting and notification

**Stack:**
- Prometheus (metrics collection) - Port 9090
- Grafana (dashboards) - Port 3001
- Alertmanager (notifications) - Port 9093
- Log aggregation (Loki or similar)

**Deployment Date:** 2025-12-11
**Status:** 🧪 Deployed - Undergoing testing and verification
**Note:** Centralized observability infrastructure deployed, quality assurance in progress

**Validation Status:** 🧪 Test & Verify - Quality assurance and validation testing in progress

---

### hx-lang-server (192.168.10.226) – ✅ Operational
**Role:** LangGraph Server.  
**Primary Responsibilities:** 
- Host LangGraph instance for agent workflow orchestration
- State management for multi-agent systems
- Graph-based agent execution
- Used by applications and agents for complex workflow chaining

**Technology Stack:**
- LangGraph (NOT LangChain)
- Python runtime
- Integration with LLM providers via hx-litellm-server

**Integration Points:**
- LLM: hx-litellm-server
- Database: hx-postgres-server (state persistence)
- Cache: hx-redis-server (session state)
- Vector: hx-qdrant-server (RAG integration)
- Applications: hx-dev-server, hx-demo-server, hx-agui-server

**Deployment Date:** 2025-12-11
**Note:** This server runs **LangGraph** (agent orchestration framework), NOT LangChain

**Validation Status:** ✅ Operational - LangGraph deployment complete, agent orchestration verified

---

## Server Status Summary

### Deployment Statistics (Current State)

**Total Servers:** 31 nodes allocated

**Operational Status Breakdown:**
- **Operational (✅):** 28 nodes
  - Identity & Control: 4 nodes
  - Model & Inference: 4 nodes
  - Data Plane: 5 nodes
  - Agentic & Toolchain: 10 nodes (fastmcp, n8n-mcp, n8n, docling-mcp, docling-worker, crawl4ai-mcp, crawl4ai-worker, literag, mem0-mcp, shadcn)
  - Application Layer: 3 nodes (webui, dev-server, demo-server)
  - Integration & Governance: 2 nodes (cc-server, lang-server)

- **Test & Verify (🧪):** 1 node
  - hx-metric-server (observability stack)

- **Repurposed (❌):** 1 node
  - hx-coderabbit-server (hardware repurposed to hx-mem0-mcp-server)

- **Not Counted:** 1 node
  - Gateway (192.168.10.1) - Network infrastructure, not server

**Total Accounted:** 28 + 1 + 1 = 30 active servers + 1(gateway) = 31 total

### Service Category Distribution

| Category | Nodes | Operational | Test & Verify | Repurposed |
|----------|-------|-------------|---------------|-----------|
| Identity & Control | 4 | 4 | 0 | 0 |
| Model & Inference | 4 | 4 | 0 | 0 |
| Data Plane | 5 | 5 | 0 | 0 |
| Agentic & Toolchain | 11 | 10 | 0 | 1 |
| Application Layer | 4 | 4 | 0 | 0 |
| Integration & Governance | 3 | 2 | 1 | 0 |
| **Totals** | **31** | **28** | **1** | **1** |

---

## Traceability to Architecture Documentation

### Alignment with HX-Infrastructure Standards

This node inventory aligns with HX-Infrastructure core documents:
1. **Network Topology** (`network/network-topology.md`) - IP allocations and zones match exactly
2. **Architecture Standards** (`standards/architecture-standards.md`) - Service patterns comply
3. **Documentation Requirements** (`standards/documentation-requirements.md`) - Format follows standards
4. **Testing Requirements** (`standards/testing-requirements.md`) - Validation status reflects test-driven deployment

### Updated Traceability Map

**Frontend/UI Layer:**
- `hx-webui-server` (✅ Operational - Open WebUI)
- `hx-agui-server` (✅ Operational - AG-UI protocol server)
- Custom Next.js apps: `hx-dev-server` (✅ Operational) → `hx-demo-server` (✅ Operational)

**Backend & Integration Services:**
- LLM Gateway: `hx-litellm-server` (✅)
- MCP Gateway: `hx-fastmcp-server` (✅ with Brave Search)
- Workflow: `hx-n8n-mcp-server` (✅), `hx-n8n-server` (✅)
- Document Processing: `hx-docling-server` (✅), `hx-docling-mcp-server` (⬜)
- Web Scraping: `hx-crawl4ai-mcp-server` (✅), `hx-crawl4ai-server` (✅)
- Agent Orchestration: `hx-lang-server` (⬜ - LangGraph)
- RAG Framework: `hx-literag-server` (✅)
- Memory Layer: `hx-mem0-mcp-server` (⬜ Planned - spec approved)

**Data & Model Infrastructure:**
- Relational: `hx-postgres-server` (✅)
- Cache: `hx-redis-server` (✅)
- Vectors: `hx-qdrant-server` (✅), `hx-qdrant-ui-server` (✅), `hx-qmcp-server` (✅)
- Models: `hx-ollama1-server` (✅), `hx-ollama2-server` (✅), `hx-ollama3-server` (✅)

**Platform & Infrastructure:**
- Auth/DNS: `hx-dc-server` (✅)
- Certificates: `hx-ca-server` (✅)
- Ingress: `hx-ssl-server` (✅)
- Config Mgmt: `hx-control-node` (✅)

**DevOps & Governance:**
- Integration: `hx-cc-server` (✅)
- Observability: `hx-metric-server` (⬜)

**Reserved/Future:**
- Code Review: `hx-coderabbit-server` (⚠️)
- Component Library: `hx-shadcn-server` (⚠️)

---

## Appendix A – Host File Reference

**Current as of:** 2025-11-15

Below is the production static hosts file deployed across HX-Infrastructure. This matches the network topology exactly.

```
# =====================================================================
# HX-Infrastructure Project – Static hosts file
# Domain: hx.dev.local
# Generated: 2025-10-21
# Last Updated: 2025-12-06
# =====================================================================

127.0.0.1   localhost
::1         localhost ip6-localhost ip6-loopback
fe00::0     ip6-localnet
ff00::0     ip6-mcastprefix
ff02::1     ip6-allnodes
ff02::2     ip6-allrouters

# --- HX Servers -------------------------------------------------------
192.168.10.200  hx-dc-server.hx.dev.local            hx-dc-server
192.168.10.201  hx-ca-server.hx.dev.local            hx-ca-server
192.168.10.202  hx-ssl-server.hx.dev.local           hx-ssl-server
192.168.10.203  hx-control-node.hx.dev.local         hx-control-node
192.168.10.204  hx-ollama1-server.hx.dev.local       hx-ollama1-server
192.168.10.205  hx-ollama2-server.hx.dev.local       hx-ollama2-server
192.168.10.206  hx-ollama3-server.hx.dev.local       hx-ollama3-server
192.168.10.207  hx-qdrant-server.hx.dev.local        hx-qdrant-server
192.168.10.208  hx-qdrant-ui-server.hx.dev.local     hx-qdrant-ui-server
192.168.10.209  hx-postgres-server.hx.dev.local      hx-postgres-server
192.168.10.210  hx-redis-server.hx.dev.local         hx-redis-server
192.168.10.211  hx-qmcp-server.hx.dev.local          hx-qmcp-server
192.168.10.212  hx-litellm-server.hx.dev.local       hx-litellm-server
192.168.10.213  hx-fastmcp-server.hx.dev.local       hx-fastmcp-server
192.168.10.214  hx-n8n-mcp-server.hx.dev.local       hx-n8n-mcp-server
192.168.10.215  hx-n8n-server.hx.dev.local           hx-n8n-server
192.168.10.216  hx-docling-server.hx.dev.local       hx-docling-server
192.168.10.217  hx-docling-mcp-server.hx.dev.local   hx-docling-mcp-server
192.168.10.218  hx-crawl4ai-mcp-server.hx.dev.local  hx-crawl4ai-mcp-server
192.168.10.219  hx-crawl4ai-server.hx.dev.local      hx-crawl4ai-server
192.168.10.220  hx-literag-server.hx.dev.local       hx-literag-server
192.168.10.221  hx-agui-server.hx.dev.local          hx-agui-server
192.168.10.222  hx-dev-server.hx.dev.local           hx-dev-server
192.168.10.223  hx-demo-server.hx.dev.local          hx-demo-server
192.168.10.224  hx-cc-server.hx.dev.local            hx-cc-server
192.168.10.225  hx-metric-server.hx.dev.local        hx-metric-server
192.168.10.226  hx-lang-server.hx.dev.local          hx-lang-server
192.168.10.227  hx-webui-server.hx.dev.local         hx-webui-server
192.168.10.228  hx-coderabbit-server.hx.dev.local    hx-coderabbit-server
192.168.10.229  hx-shadcn-server.hx.dev.local        hx-shadcn-server
192.168.10.231  hx-mem0-mcp-server.hx.dev.local      hx-mem0-mcp-server
127.0.1.1       hx-control-node.hx.dev.local         hx-control-node
```

---

## Change Log

### Infrastructure Changes

| Date | Change Type | Description | Affected Servers | Changed By |
|------|-------------|-------------|------------------|------------|
| 2025-11-05 | Initial | Platform nodes initial documentation | All 30 servers | Infrastructure Team |
| 2025-11-11 | Deployment | Deployed hx-n8n-mcp-server | hx-n8n-mcp-server (.214) | Infrastructure Team |
| 2025-11-15 | Deployment | Deployed hx-n8n-server, validated operational | hx-n8n-server (.215) | Infrastructure Team |
| 2025-11-15 | Adaptation | Adapted for HX-Infrastructure from Hana-X | All documentation | HX-Infrastructure Team |
| 2025-11-15 | Correction | Corrected operational status for 6 servers | docling-mcp, agui, dev, metric, lang, shadcn | HX-Infrastructure Team |
| 2025-11-15 | Correction | Fixed LangChain → LangGraph for hx-lang-server | hx-lang-server (.226) | HX-Infrastructure Team |
| 2025-11-15 | Enhancement | Added AG-UI documentation section | hx-agui-server (.221) | HX-Infrastructure Team |
| 2025-11-15 | Correction | Fixed server count math (28→21 operational) | Server summary | HX-Infrastructure Team |
| 2025-12-06 | Deployment | hx-dev-server promoted to OPERATIONAL | hx-dev-server (.222) | william-chen |
| 2025-12-08 | Planning | hx-mem0-mcp-server specification APPROVED | hx-mem0-mcp-server (.231) | Agent Zero |
| 2025-12-06 | Update | Updated server counts (22→23 operational, 5→4 planned) | Server summary | william-chen |
| 2025-12-08 | Repurpose | hx-coderabbit-server hardware repurposed | hx-coderabbit-server (.228) → hx-mem0-mcp-server (.231) | william-chen |
| 2025-12-08 | Deployment | hx-mem0-mcp-server promoted to OPERATIONAL | hx-mem0-mcp-server (.231) | Agent Zero |
| 2025-12-08 | Update | Updated server counts (23→24 operational), added Repurposed status | Server summary | Agent Zero |
| 2025-12-09 | Deployment | hx-shadcn-server promoted to OPERATIONAL | hx-shadcn-server (.229) | william-chen |
| 2025-12-09 | Update | Updated server counts (24→25 operational, 4→3 planned) | Server summary | william-chen |
| 2025-12-11 | Deployment | hx-docling-mcp-server promoted to OPERATIONAL | hx-docling-mcp-server (.217) | Agent Zero |
| 2025-12-11 | Deployment | hx-demo-server promoted to OPERATIONAL | hx-demo-server (.223) | william-chen |
| 2025-12-11 | Deployment | hx-lang-server promoted to OPERATIONAL | hx-lang-server (.226) | william-chen |
| 2025-12-11 | Deployment | hx-metric-server deployed for Test & Verify | hx-metric-server (.225) | william-chen |
| 2025-12-11 | Update | Updated server counts (25→28 operational, 3→1 test/verify) | Server summary | Agent Zero |

### Pending Changes

| Planned Date | Change Type | Description | Impact | Priority |
|--------------|-------------|-------------|--------|----------|
| TBD | Deployment | Deploy hx-docling-mcp-server | Complete Docling MCP integration | High |
| TBD | Deployment | Deploy hx-agui-server (AG-UI protocol) | Add AG-UI agent interaction capability | High |
| TBD | Deployment | Deploy hx-metric-server observability stack | Centralized monitoring | High |
| TBD | Deployment | Deploy hx-lang-server (LangGraph) | Agent orchestration framework | High |
| TBD | Configuration | Complete hx-demo-server setup | Finish stakeholder demo environment | Medium |
| TBD | Deployment | Deploy hx-coderabbit-server | Code review automation | Medium |

---

## Document Maintenance

### Update Triggers

This document MUST be updated when:
- ✅ New servers deployed to infrastructure
- ✅ Server status changes (operational, in-progress, planned)
- ✅ IP address changes occur
- ✅ Service roles or responsibilities change
- ✅ Integration points are modified
- ✅ New capabilities are added to servers
- ✅ Servers are decommissioned

### Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-11-05 | Initial platform nodes documentation | Infrastructure Team |
| 2.0 | 2025-11-15 | Adapted for HX-Infrastructure, corrected status, added AG-UI, fixed LangGraph | HX-Infrastructure Team |
| 2.1 | 2025-12-06 | hx-dev-server promoted to OPERATIONAL (CAIO approved), updated server counts | william-chen |
| 2.2 | 2025-12-09 | hx-shadcn-server promoted to OPERATIONAL, updated server counts (25 operational, 3 planned) | william-chen |
| 2.3 | 2025-12-11 | All servers promoted to OPERATIONAL except metrics (Test & Verify): hx-docling-mcp-server, hx-demo-server, hx-lang-server, hx-metric-server; updated counts (28 operational, 1 test/verify) | Agent Zero |

### Related Documents

**HX-Infrastructure Core Documents:**
- `constitution.md` - Project principles and philosophy
- `README.md` - Repository overview and navigation
- `action-plan-v2-updated.md` - Project roadmap and status

**Network and Infrastructure:**
- `network/network-topology.md` - Network architecture and IP allocations (v1.1.1)
- `network/port-mapping.md` - Service port assignments (when created)
- `nodes/<node-name>/node-spec.md` - Individual node specifications (when created)

**Standards:**
- `standards/naming-conventions.md` - Naming standards
- `standards/architecture-standards.md` - Architecture guidelines
- `standards/documentation-requirements.md` - Documentation standards
- `standards/testing-requirements.md` - Testing and validation requirements
- `standards/deployment-requirements.md` - Deployment procedures

**Agent Documentation:**
- `hx-agents/hx-agent-inventory.md` - 32 agents (5 Core Team SMEs + 27 Technology SMEs) and capabilities
- `hx-agents/hx-orchestration-guide.md` - Multi-agent workflows
- `CLAUDE.md` - Agent Zero orchestration instructions

**External References:**
- AG-UI Protocol Documentation: https://ag-ui.com/
- AG-UI GitHub Repository: https://github.com/ag-ui-protocol/ag-ui

---

**Document Information:**
- **Version**: 2.2
- **Status**: ACTIVE - Authoritative Infrastructure Baseline
- **Maintained By**: HX-Infrastructure Team
- **Review Frequency**: Weekly or upon infrastructure changes
- **Last Review**: 2025-12-09
- **Next Review**: 2025-12-16 (or upon significant change)

---

*This platform nodes inventory represents the current state of HX-Infrastructure's server deployment. It serves as the authoritative reference for server capabilities, operational status, and integration points. All infrastructure changes must be reflected in this document within 24 hours of implementation.*
