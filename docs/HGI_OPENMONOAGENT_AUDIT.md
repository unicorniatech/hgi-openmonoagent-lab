# HGI OpenMonoAgent Audit

**Date:** 2026-05-19  
**Auditor:** HGI AI Systems  
**Source:** https://github.com/unicorniatech/hgi-openmonoagent-lab  
**Version Audited:** Main branch (beta)

---

## Executive Summary

OpenMonoAgent is a **local-first, Dockerized coding agent** built on .NET 10 with llama.cpp inference. It demonstrates excellent architectural patterns for agent orchestration but has **critical platform and license constraints** that prevent direct adoption as HGI core.

**HGI Fit Score: 6/10** — Strong architectural inspiration, significant compatibility barriers.

**Recommendation:** `INSPIRATION ONLY` — Do not adopt directly. Extract patterns for HGI-native agent layer.

---

## 1. Repository Structure

### Primary Technology Stack
| Component | Technology |
|-----------|------------|
| **Language** | C# 12 (.NET 10 Preview) |
| **Runtime** | .NET 10 SDK (preview) |
| **Container** | Docker + Docker Compose |
| **Inference** | llama.cpp (bundled) |
| **LSP** | OmniSharp, typescript-language-server, pylsp, gopls, rust-analyzer |
| **MCP** | Client via stdio JSON-RPC |
| **UI** | Terminal UI (Spectre.Console) + Classic CLI |

### Architecture Split
```
┌────────────────────────────────────────┐
│  OpenMono CLI (.NET 10)                │
│  ├── 20 built-in tools                 │
│  ├── MCP client (dynamic tools)        │
│  ├── LSP client (code intelligence)    │
│  └── Sub-agent orchestration           │
└────────────────────────────────────────┘
           │ HTTP :7474 (OpenAI-compatible)
           ▼
┌────────────────────────────────────────┐
│  llama-server (llama.cpp)              │
│  ├── Qwen3.6-27B (GPU)                 │
│  └── Qwen3.6-35B-A3B (CPU/MoE)         │
└────────────────────────────────────────┘
```

### Directory Structure
```
src/OpenMono.Cli/
├── Agents/           # Sub-agent definitions
├── Commands/         # 14 slash commands
├── Config/           # settings.json loader
├── History/          # File snapshots for /undo
├── Hooks/            # Pre/post tool hooks
├── Llm/              # Provider registry (4 providers)
├── Lsp/              # Language server client
├── Mcp/              # MCP client
├── Memory/           # Cross-session memory (YAML)
├── Permissions/      # Capability-based permission engine
├── Playbooks/        # YAML workflow engine
├── Session/          # Conversation loop, context management
├── Tools/            # 20 built-in tools + registry
├── Tui/              # Terminal UI renderer
└── Utils/            # Git, process helpers
```

---

## 2. License Analysis

### License: GNU AGPL-3.0 (Affero GPL v3)

**Critical Finding:** AGPL-3.0 is a **strong copyleft** license with network interaction provisions.

| Right | Status | Notes |
|-------|--------|-------|
| **Commercial use** | ✅ Allowed | Must comply with source distribution requirements |
| **Modification** | ✅ Allowed | Must release modifications under AGPL-3.0 |
| **Redistribution** | ✅ Allowed | Must include source and license |
| **SaaS/Network use** | ⚠️ **COPYLEFT TRIGGERED** | §13 requires offering source to all network users |
| **Proprietary integration** | ❌ **PROHIBITED** | Linking with HGI core would infect with AGPL |
| **Closed-source fork** | ❌ **PROHIBITED** | All distributed/modified versions must be AGPL |

### HGI License Risk Assessment

**SEVERE RISK:**
- Direct integration with HGI-EDGE-RUNTIME (proprietary) would violate AGPL
- Cannot be embedded in VistaDev products without full source release
- Network use clause means any HGI service using OpenMono must offer source

**Mitigation:**
- Use only as **reference/inspiration**, not as a component
- Clean-room reimplementation of patterns with MIT/Apache license
- Keep as separate, standalone AGPL tool only (if at all)

---

## 3. Agent Architecture Audit

### 3.1 Agent Creation Model

**Sub-agent System:**
```csharp
// 5 specialist sub-agents with locked tool sets
var agents = new Dictionary<string, AgentDefinition>
{
    ["general-purpose"] = new(25, ["*"]),           // 25 turns, all tools
    ["Explore"] = new(15, ["FileRead", "Glob", "Grep", "MCP*"]),
    ["Plan"] = new(10, ["FileRead", "TodoWrite"]),  // read-only + planning
    ["Coder"] = new(30, ["FileRead", "FileWrite", "FileEdit", "Bash", ...]),
    ["Verify"] = new(20, ["FileRead", "Roslyn", "LSP", "Bash", "MCP*"])
};
```

**Pattern Analysis:**
- ✅ Tool-locked sandboxes per agent type
- ✅ Turn limits prevent runaway execution
- ✅ Isolated sessions (separate `SessionState`)
- ✅ Parent permission engine reused (good for audit trail)
- ❌ No formal agent-to-agent messaging protocol
- ❌ No agent registry/discovery

### 3.2 Tool System

**20 Built-in Tools:**

| Tool | Category | Permission Level |
|------|----------|------------------|
| `FileRead` | File | AutoAllow |
| `FileWrite` | File | Ask (destructive) |
| `FileEdit` | File | Ask (destructive) |
| `ApplyPatch` | File | Ask (destructive) |
| `Glob` | Discovery | AutoAllow |
| `Grep` | Discovery | AutoAllow |
| `ListDirectory` | Discovery | AutoAllow |
| `Bash` | Execution | Ask (destructive commands denied) |
| `LSP` | Code Intel | AutoAllow |
| `Roslyn` | Code Intel | AutoAllow |
| `WebFetch` | Network | Ask |
| `WebSearch` | Network | Ask |
| `Agent` | Orchestration | Ask |
| `TodoWrite` | Planning | AutoAllow |
| `PlanModeTool` | Planning | AutoAllow |
| `Playbook` | Workflow | Ask |
| `MemorySave` | Memory | AutoAllow |
| `AskUser` | UI | AutoAllow |
| `ToolSearch` | Discovery | AutoAllow |

**Tool Pipeline (12 steps):**
```
1. Parse JSON arguments
2. Schema validation
3. Path sanity check (workspace escape detection)
4. Plan mode guard
5. Capability check → PermissionEngine
6. Result cache lookup (read-only)
7. Pre-tool hook
8. Execute
9. Post-tool hook
10. Artifact store (>10KB)
11. Cache write
12. File cache invalidation
```

**Strengths:**
- ✅ 12-step pipeline with clear separation of concerns
- ✅ Capability-based permission system
- ✅ Read-only tools run in parallel
- ✅ Writable tools run serially
- ✅ Built-in sanity checks (destructive command detection)
- ✅ Path guards prevent workspace escape

**Weaknesses:**
- ❌ No formal tool versioning/schema evolution
- ❌ No tool result streaming for long operations
- ❌ Limited tool composition/chaining primitives

### 3.3 Planning System

**Two Planning Modes:**

1. **Plan Mode** (`/plan`) — Read-only exploration
   - Restricts to `FileRead`, `Glob`, `Grep`, `MCP`
   - Safe architecture exploration before any writes

2. **Todo Tool** — Task decomposition
   - Agent writes structured todo list
   - No enforcement/completion tracking
   - Just a structured scratchpad

**Analysis:**
- ✅ Plan mode prevents premature writes
- ❌ No formal plan execution engine
- ❌ No dependency graph between tasks
- ❌ No automatic plan revision

### 3.4 Memory System

**Implementation:** YAML frontmatter files in `~/.openmono/memory/`

```yaml
---
name: project-context
description: High-level architecture decisions
type: project
---

Content here...
```

**Features:**
- Cross-session persistence
- YAML metadata with markdown content
- Auto-indexed in `MEMORY.md`
- Injected into system prompt

**Analysis:**
- ✅ Simple, human-readable format
- ✅ Version control friendly
- ❌ No vector/semantic search
- ❌ No automatic memory consolidation
- ❌ No memory decay/retention policy

### 3.5 LLM Provider Support

**4 Providers:**

| Provider | Status | Models |
|----------|--------|--------|
| `local` (llama.cpp) | ✅ **FULLY SUPPORTED** | Any GGUF |
| `ollama` | ⚠️ WIP | llama3, qwen2.5-coder |
| `openai` | ⚠️ WIP | gpt-4o, o1, o3-mini |
| `anthropic` | ⚠️ WIP | claude-opus-4-7 |

**Architecture:**
```csharp
interface ILlmClient
{
    IAsyncEnumerable<StreamChunk> StreamChatAsync(
        IReadOnlyList<Message> messages,
        JsonElement? tools,
        LlmOptions options,
        CancellationToken ct);
}
```

**Analysis:**
- ✅ Clean provider abstraction
- ✅ Streaming support
- ✅ Hot-swappable mid-session (`/model` command)
- ⚠️ Cloud providers marked as WIP/untested
- ✅ OpenAI-compatible endpoint support (generic client)

---

## 4. HGI Compatibility Audit

### 4.1 Local-First Compatibility

| Aspect | Status | Notes |
|--------|--------|-------|
| **Fully offline capable** | ✅ Yes | llama.cpp runs entirely local |
| **No cloud dependencies** | ✅ Yes | Optional cloud providers, not required |
| **Local model support** | ✅ Excellent | GGUF, Ollama, llama.cpp |
| **Data residency** | ✅ Local only | Everything in `~/.openmono/` |

### 4.2 Platform Compatibility

| Platform | Status | Notes |
|----------|--------|-------|
| **Linux** | ✅ Primary target | Ubuntu 26.04 LTS recommended |
| **Docker** | ✅ Required | All execution sandboxed |
| **Windows** | ⚠️ Unlikely | Bash scripts, Docker-dependent |
| **macOS** | ⚠️ Partial | Docker Desktop/Colima support |
| **HGI-EDGE-RUNTIME** | ❌ **INCOMPATIBLE** | .NET 10 vs Rust runtime |
| **hgi-local-node** | ❌ **INCOMPATIBLE** | Different architecture |

### 4.3 Integration Compatibility

| System | Compatibility | Analysis |
|--------|---------------|----------|
| **HGI-EDGE-RUNTIME** | ❌ None | Different tech stack (.NET vs Rust) |
| **hgi-local-node** | ❌ None | No shared protocol |
| **RedVecinalMX** | ⚠️ Manual | Via MCP or HTTP API only |
| **MarshantaMX** | ⚠️ Manual | Via MCP or HTTP API only |
| **MOLIE Agent Mode** | ⚠️ Partial | Could integrate via LSP/MCP |

### 4.4 Tool Calling Compatibility

**Schema:** OpenAI-compatible function calling
```json
{
  "type": "function",
  "function": {
    "name": "FileRead",
    "description": "Read file contents",
    "parameters": { ... }
  }
}
```

**HGI Assessment:**
- ✅ Standard OpenAI tool schema (compatible with HGI tool registry)
- ✅ Streaming tool call support
- ✅ Parallel tool execution for read-only tools
- ❌ No HGI-specific tool discovery protocol

### 4.5 Security & Sandboxing

| Feature | Implementation | Rating |
|---------|----------------|--------|
| **Containerization** | Docker (required) | ✅ Strong |
| **Workspace isolation** | Bind-mount at `/workspace` | ✅ Good |
| **Path traversal guard** | `PathGuard.Validate()` | ✅ Good |
| **Destructive command detection** | `SanityCheck.IsDestructiveCommand()` | ✅ Good |
| **Permission system** | Capability-based | ✅ Strong |
| **Secret scanning** | `SecretScanner.Scan()` | ✅ Good |
| **Code execution sandbox** | Docker only | ⚠️ Linux-only |
| **Network egress control** | Capability check | ⚠️ Ask-based |

**Security Weaknesses:**
- ❌ No formal capability attenuation
- ❌ No proof-carrying code for tools
- ❌ Shell command parsing is heuristic-based
- ❌ No formal verification of sandbox escape

### 4.6 Telemetry & Privacy

| Aspect | Status |
|--------|--------|
| **Phone home** | ❌ None detected |
| **Usage telemetry** | ❌ None |
| **Crash reporting** | ❌ None |
| **Model telemetry** | ❌ None |
| **Opt-in tracing** | ⚠️ Planned (OpenTelemetry) |

**Privacy Score: EXCELLENT**
- Zero telemetry by default
- No external connections without explicit configuration
- All data stays in `~/.openmono/`

---

## 5. Deployment Audit

### 5.1 Install Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|---------------|
| **OS** | Ubuntu 25.10 | Ubuntu 26.04 LTS |
| **GPU VRAM** | 12 GB | 24 GB |
| **CPU RAM** | 24 GB | 32 GB |
| **Disk** | 20 GB | 25 GB |
| **Docker** | Required | Required |

### 5.2 Installation Commands

```bash
# One-liner install
curl -fsSL https://raw.githubusercontent.com/StartupHakk/OpenMonoAgent.ai/main/get-openmono.sh | bash

# Manual setup
~/openmono.ai/openmono setup          # Auto-detect
~/openmono.ai/openmono setup --gpu    # Force GPU
~/openmono.ai/openmono setup --cpu    # Force CPU
```

### 5.3 Docker Services

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `llama-server` | `ghcr.io/ggml-org/llama.cpp:server` | 7474 | Inference |
| `agent` | Built from Dockerfile.agent | N/A | CLI in container |

### 5.4 Environment Variables

```bash
# Core LLM settings
OPENMONO_ENDPOINT=http://localhost:7474
OPENMONO_MODEL=qwen3.6-27b
OPENMONO_API_KEY=                      # For cloud providers

# Paths
OPENMONO_WORKSPACE=/workspace
OPENMONO_HOST_WORKSPACE=.
OPENMONO_DATA_DIR=~/.openmono

# Dual-box tunnel (optional)
FRP_VM_ADDRESS=
FRP_AUTH_TOKEN=
```

### 5.5 Port Requirements

| Port | Service | Direction | Notes |
|------|---------|-----------|-------|
| 7474 | llama-server | Localhost | Can bind to 0.0.0.0 for dual-box |
| 22 | SSH | Outbound | For dual-box tunnel setup |

### 5.6 Cloud Dependencies

| Dependency | Required | Purpose | Offline Alternative |
|------------|----------|---------|---------------------|
| **HuggingFace** | ⚠️ One-time | Model download | Manual GGUF download |
| **Docker Hub** | ⚠️ One-time | Base images | Pre-pull images |
| **GitHub** | ⚠️ One-time | Install script | Manual clone |
| **Cloud LLM** | ❌ No | Optional | Local llama.cpp |

**Conclusion:** Can run fully offline after initial setup.

### 5.7 Windows Compatibility

**Status: NOT SUPPORTED**

- Requires Linux Docker containers
- Bash-based CLI wrapper
- No native Windows build
- WSL2 might work but untested

### 5.8 Mini PC / Edge Node Compatibility

| Hardware | Feasibility | Notes |
|----------|-------------|-------|
| **Intel NUC 13+** | ✅ Good | CPU mode with 24GB+ RAM |
| **Raspberry Pi 5** | ❌ No | Insufficient RAM (need 24GB) |
| **Mini PC 16GB** | ⚠️ Marginal | May swap heavily |
| **Edge GPU (Jetson)** | ❌ Unknown | No CUDA support documented |

---

## 6. Risk Assessment

### 6.1 Code Execution Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| **Bash tool injection** | HIGH | Parser + destructive pattern detection |
| **Docker escape** | MEDIUM | User namespacing, no root in container |
| **Path traversal** | MEDIUM | `PathGuard.Validate()` on all file ops |
| **Interpreter abuse** | MEDIUM | Blocks `python -c`, `node -e`, etc. |
| **Fork bomb** | LOW | Pattern detection for `:(){:|:&};:` |
| **Disk fill** | MEDIUM | No quota enforcement detected |

### 6.2 Prompt Injection Risks

| Vector | Status | Notes |
|--------|--------|-------|
| **System prompt leak** | ⚠️ Possible | No prompt hardening detected |
| **Tool name confusion** | ⚠️ Possible | No formal tool namespacing |
| **Context window stuffing** | ⚠️ Possible | 192K context is large attack surface |

### 6.3 Shell Access Risks

**Assessment:** Bash tool is powerful but gated:
- Permission engine requires explicit approval
- Destructive commands auto-denied
- Protected paths blocked (`/etc`, `/usr`, credential dirs)
- `sudo`, `chmod`, `chown` binaries blocked

**Remaining Risk:** Escalation via less obvious destructive commands.

### 6.4 File Write Risks

| Protection | Implementation | Effectiveness |
|------------|----------------|---------------|
| **Workspace boundary** | `PathGuard.Validate()` | Good |
| **Credential directory block** | Hardcoded paths | Good |
| **System path block** | `/etc`, `/usr`, etc. | Good |
| **Secret scanning** | `SecretScanner.Scan()` | Good (warns on secrets) |
| **Undo history** | `FileHistory.Record*()` | Good (for recovery) |

### 6.5 Dependency Risks

| Dependency | Risk Level | Notes |
|------------|------------|-------|
| **.NET 10 Preview** | MEDIUM | Preview SDK may have instability |
| **llama.cpp** | LOW | Stable, widely used |
| **Qwen3.6** | LOW | Open-weights, well-tested |
| **Docker** | LOW | Industry standard |
| **Spectre.Console** | LOW | Mature .NET library |
| **YamlDotNet** | LOW | Mature YAML library |

### 6.6 Telemetry Risks

**Assessment: MINIMAL**
- No telemetry detected in codebase
- No external API calls without configuration
- All data stays local

### 6.7 Cloud Lock-in Risk

**Assessment: NONE**
- Fully functional without any cloud service
- Cloud providers are optional add-ons
- Models are downloadable GGUF files

### 6.8 Maintenance Risk

| Factor | Assessment |
|--------|------------|
| **Project age** | Very new (2026) |
| **Activity** | High (beta, rapid iteration) |
| **Bus factor** | Unknown (StartupHakk = small team?) |
| **License stability** | AGPL-3.0 (unlikely to change) |
| **.NET 10 maturity** | Preview SDK (risk) |

---

## 7. Lessons for HGI

### 7.1 Patterns to Extract

#### 7.1.1 Agent Loop Pattern
```csharp
// From ConversationLoop.cs
for (var turn = 0; turn < maxTurns; turn++)
{
    // 1. Stream LLM response
    // 2. Accumulate tool calls
    // 3. Execute tools (parallel for read-only)
    // 4. Add results to context
    // 5. Continue until no tool calls
}
```

**HGI Application:**
- Implement in `hgi-local-node` agent runtime
- Add turn limits and doom loop detection
- Implement context window checkpoints at 65%/80%

#### 7.1.2 Tool Registry Pattern
```csharp
interface ITool
{
    string Name { get; }
    string Description { get; }
    JsonElement InputSchema { get; }
    bool IsReadOnly { get; }
    bool IsConcurrencySafe { get; }
    Task<ToolResult> ExecuteAsync(...);
    IReadOnlyList<Capability> RequiredCapabilities(JsonElement input);
}
```

**HGI Application:**
- Use for HGI tool registry
- Add WASM sandbox for tool execution
- Support dynamic tool registration via HGI protocol

#### 7.1.3 Capability-Based Permissions
```csharp
public abstract record Capability
{
    public abstract string Summary { get; }
}
public sealed record FileWriteCap(string Path, string Operation) : Capability;
public sealed record ProcessExecCap(string Binary, IReadOnlyList<string> Args) : Capability;
```

**HGI Application:**
- Implement in HGI-EDGE-RUNTIME permission system
- Add capability attenuation for worker nodes
- Support capability delegation across nodes

#### 7.1.4 Sub-Agent Pattern
```csharp
var agents = new Dictionary<string, AgentDefinition>
{
    ["Explore"] = new(15, ["FileRead", "Glob", "Grep"]),  // read-only
    ["Coder"] = new(30, ["FileRead", "FileWrite", "Bash"]) // full access
};
```

**HGI Application:**
- Implement worker specialization in HGI node network
- Tool-locked sandboxes per worker type
- Turn budgets prevent runaway execution

#### 7.1.5 12-Step Tool Pipeline
```
Parse → Validate → Sanity Check → Plan Mode Guard → 
Capability Check → Cache → Pre-Hook → Execute → 
Post-Hook → Artifact Store → Cache Write → Invalidation
```

**HGI Application:**
- Implement in HGI-EDGE-RUNTIME tool executor
- Add HGI-specific gates (audit log, human approval)
- Support hooks for RedVecinal/Marshanta integration

#### 7.1.6 Playbook/Workflow Pattern
```yaml
---
name: commit
version: 1.0.0
parameters:
  message:
    type: String
    required: true
steps:
  - name: validate
    action: exec
    tool: Bash
    input:
      command: "npm test"
  - name: commit
    action: exec
    tool: Bash
    input:
      command: "git commit -m '{{parameters.message}}'"
---
System prompt here...
```

**HGI Application:**
- Use for RedVecinalMX workflow automation
- Support HGI-native playbook format (JSON/YAML)
- Add checkpoint/resume for long workflows

### 7.2 Anti-Patterns to Avoid

| OpenMono Pattern | HGI Alternative |
|------------------|-----------------|
| Docker-only sandbox | WASM + capability-based sandbox |
| .NET 10 preview | Stable Rust toolchain |
| AGPL-3.0 license | MIT/Apache-2.0 |
| Ubuntu-only | Cross-platform (Windows/Linux/macOS) |
| 24GB RAM requirement | Efficient quantized models |
| llama.cpp bundled | Pluggable inference (Ollama, local, cloud) |

---

## 8. Final Recommendation

### 8.1 Adoption Decision: REJECT FOR CORE

**Do NOT adopt OpenMonoAgent as HGI core component because:**

1. **License incompatibility:** AGPL-3.0 would infect HGI-EDGE-RUNTIME
2. **Platform incompatibility:** Linux/Docker only, no Windows support
3. **Tech stack mismatch:** .NET vs Rust architecture
4. **Resource requirements:** 24GB RAM minimum is too high for edge nodes
5. **Maintenance risk:** Beta software, .NET 10 preview, new project

### 8.2 Inspiration Value: HIGH

**Strong patterns to extract and reimplement:**

1. ✅ Capability-based permission system
2. ✅ 12-step tool execution pipeline
3. ✅ Sub-agent specialization with tool locking
4. ✅ Conversation loop with doom loop detection
5. ✅ Context window checkpointing
6. ✅ YAML playbook workflows
7. ✅ MCP client integration
8. ✅ Local-first, privacy-preserving design

### 8.3 Recommended Next Steps

| Priority | Action | Owner |
|----------|--------|-------|
| **P0** | Document HGI Agent Orchestration Requirements | HGI Architecture |
| **P1** | Design capability-based permission system for HGI-EDGE | HGI Security |
| **P1** | Prototype tool registry in Rust (WASM-compatible) | HGI Runtime |
| **P2** | Design sub-agent orchestration protocol | HGI Protocol |
| **P2** | Implement context checkpointing in hgi-local-node | HGI Node |
| **P3** | Add MCP client support to HGI-EDGE | HGI Integration |
| **P3** | Design HGI-native playbook format | HGI UX |

---

## Appendix A: File Inventory

### Core Source Files (Most Relevant)

| File | Purpose | Lines |
|------|---------|-------|
| `src/OpenMono.Cli/Program.cs` | Entry point, DI wiring | ~1000 |
| `src/OpenMono.Cli/Session/ConversationLoop.cs` | Main agent loop | ~400 |
| `src/OpenMono.Cli/Tools/ToolDispatcher.cs` | Tool execution pipeline | ~350 |
| `src/OpenMono.Cli/Permissions/PermissionEngine.cs` | Capability-based permissions | ~370 |
| `src/OpenMono.Cli/Tools/AgentTool.cs` | Sub-agent orchestration | ~200 |
| `src/OpenMono.Cli/Tools/BashTool.cs` | Shell execution | ~260 |
| `src/OpenMono.Cli/Tools/SanityCheck.cs` | Security validation | ~180 |
| `src/OpenMono.Cli/Llm/ProviderRegistry.cs` | LLM provider abstraction | ~125 |

### Documentation Files

| File | Purpose |
|------|---------|
| `docs/ARCHITECTURE.md` | System architecture |
| `docs/CONFIG.md` | Configuration reference |
| `docs/MODELS.md` | Model selection guide |
| `docs/PLAYBOOKS.md` | Workflow documentation |
| `docs/SETUP.md` | Installation guide |

---

**End of Audit**
