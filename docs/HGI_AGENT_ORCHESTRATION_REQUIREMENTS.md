# HGI Agent Orchestration Requirements

**Document Type:** Architecture Specification  
**Version:** 1.0.0-draft  
**Date:** 2026-05-19  
**Status:** DRAFT - Internal Review  

---

## 1. Executive Summary

This document defines the architectural requirements for **HGI-native agent orchestration** — a local-first, secure, and interoperable agent layer for the HGI ecosystem.

**Core Principles:**
1. **Local-first:** All agent computation runs on user hardware by default
2. **HGI-native:** Deep integration with HGI-EDGE-RUNTIME and hgi-local-node
3. **No forced cloud:** Cloud providers are optional, not required
4. **Worker-compatible:** Integrates with RedVecinalMX, MarshantaMX, VistaDev
5. **Permission-first:** Explicit capability grants with audit trails
6. **Privacy-preserving:** Minimal data collection, user-controlled telemetry

---

## 2. System Context

### 2.1 HGI Ecosystem Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER WORKSTATION                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │   VistaDev IDE  │  │  MOLIE Agent    │  │ hgi-local-node  │  │
│  │   (TypeScript)  │  │    (Electron)   │  │    (Rust)       │  │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  │
│           │                    │                    │          │
│           └────────────────────┼────────────────────┘          │
│                                │                                │
│                    ┌───────────▼───────────┐                   │
│                    │   HGI-EDGE-RUNTIME    │                   │
│                    │      (Rust/WASM)      │                   │
│                    └───────────┬───────────┘                   │
│                                │                                │
└────────────────────────────────┼────────────────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
     ┌──────▼──────┐    ┌───────▼───────┐   ┌───────▼────────┐
     │RedVecinalMX │    │  MarshantaMX  │   │  External APIs │
     │ (Workers)   │    │  (Community)  │   │ (Optional)     │
     └─────────────┘    └───────────────┘   └────────────────┘
```

### 2.2 Agent Orchestration Layer Position

```
┌─────────────────────────────────────────┐
│         APPLICATION LAYER              │
│  • VistaDev IDE (Agent Mode)            │
│  • MOLIE Agent Mode                       │
│  • RedVecinalMX Automation                │
│  • MarshantaMX Workflows                  │
└─────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│      AGENT ORCHESTRATION LAYER           │
│  • Task decomposition                     │
│  • Sub-agent coordination                 │
│  • Tool registry & dispatch               │
│  • Permission orchestration               │
│  • Audit logging                          │
└─────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│        HGI-EDGE-RUNTIME                  │
│  • Capability enforcement               │
│  • WASM sandbox                         │
│  • Inter-node messaging                 │
│  • Local-first policy                   │
└─────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│         WORKER LAYER                     │
│  • RedVecinalMX worker nodes            │
│  • MarshantaMX community nodes          │
│  • hgi-local-node instances             │
└─────────────────────────────────────────┘
```

---

## 3. Functional Requirements

### 3.1 Agent Lifecycle

| ID | Requirement | Priority |
|----|-------------|----------|
| AG-001 | Agent MUST be spawnable with explicit capability grant | P0 |
| AG-002 | Agent MUST have configurable turn/iteration limits | P0 |
| AG-003 | Agent MUST support graceful shutdown/cancellation | P0 |
| AG-004 | Agent MUST persist state for resume after interruption | P1 |
| AG-005 | Agent MUST support checkpointing for long-running tasks | P1 |
| AG-006 | Agent MUST support hot-swappable LLM providers | P1 |
| AG-007 | Agent MUST support sub-agent spawning with attenuated capabilities | P0 |

### 3.2 Tool System

| ID | Requirement | Priority |
|----|-------------|----------|
| TOOL-001 | Tool registry MUST support dynamic registration/deregistration | P0 |
| TOOL-002 | Tool registry MUST support versioning and schema evolution | P1 |
| TOOL-003 | Tool execution MUST be sandboxed (WASM or capability-based) | P0 |
| TOOL-004 | Tool execution pipeline MUST have ≥5 stages: validate, permission, pre-hook, execute, post-hook | P0 |
| TOOL-005 | Read-only tools MUST execute in parallel | P1 |
| TOOL-006 | Write-capable tools MUST execute serially | P0 |
| TOOL-007 | Tool results >10KB MUST be stored with reference returned | P2 |
| TOOL-008 | Tool registry MUST support OpenAI-compatible function schema | P1 |
| TOOL-009 | Tool registry MUST support HGI-native tool protocol | P0 |

### 3.3 Planning System

| ID | Requirement | Priority |
|----|-------------|----------|
| PLAN-001 | System MUST support task decomposition into sub-tasks | P1 |
| PLAN-002 | System MUST support dependency graph between tasks | P2 |
| PLAN-003 | System MUST support plan execution with checkpoint/resume | P1 |
| PLAN-004 | System MUST support read-only exploration mode | P1 |
| PLAN-005 | System MUST support replanning when tasks fail | P2 |

### 3.4 Memory System

| ID | Requirement | Priority |
|----|-------------|----------|
| MEM-001 | Memory MUST persist across sessions | P1 |
| MEM-002 | Memory MUST support semantic search (vector store) | P2 |
| MEM-003 | Memory MUST support explicit save/load operations | P1 |
| MEM-004 | Memory MUST support project-level and user-level scopes | P1 |
| MEM-005 | Memory MUST have configurable retention/decay policy | P3 |
| MEM-006 | Memory MUST be exportable/importable (portable) | P2 |

### 3.5 LLM Provider Support

| ID | Requirement | Priority |
|----|-------------|----------|
| LLM-001 | System MUST support local inference (llama.cpp, Ollama) | P0 |
| LLM-002 | System MUST support OpenAI-compatible endpoints | P1 |
| LLM-003 | System MUST support Anthropic API | P2 |
| LLM-004 | System MUST support hot-swapping providers mid-session | P2 |
| LLM-005 | System MUST detect provider capabilities (context size, etc.) | P1 |
| LLM-006 | System MUST gracefully degrade if provider unavailable | P1 |
| LLM-007 | System MUST support streaming responses | P1 |

---

## 4. Integration Requirements

### 4.1 HGI-EDGE-RUNTIME Integration

| ID | Requirement | Priority |
|----|-------------|----------|
| EDGE-001 | Agent orchestration MUST use HGI-EDGE for capability enforcement | P0 |
| EDGE-002 | Agent orchestration MUST use WASM sandbox from HGI-EDGE | P0 |
| EDGE-003 | Agent orchestration MUST emit events to HGI-EDGE audit log | P0 |
| EDGE-004 | Agent orchestration MUST respect HGI-EDGE local-first policy | P0 |
| EDGE-005 | Agent orchestration MUST support HGI-EDGE inter-node messaging | P1 |

### 4.2 hgi-local-node Integration

| ID | Requirement | Priority |
|----|-------------|----------|
| NODE-001 | Agent MUST be runnable as hgi-local-node module | P0 |
| NODE-002 | Agent MUST expose HTTP API for external control | P1 |
| NODE-003 | Agent MUST support node-to-node agent delegation | P2 |
| NODE-004 | Agent MUST integrate with hgi-local-node config system | P1 |

### 4.3 RedVecinalMX Integration

| ID | Requirement | Priority |
|----|-------------|----------|
| RV-001 | Agent MUST support RedVecinal worker node as tool provider | P1 |
| RV-002 | Agent MUST support delegating tasks to RedVecinal workers | P2 |
| RV-003 | Agent MUST support RedVecinal workflow definitions | P2 |
| RV-004 | Agent MUST emit events to RedVecinal audit stream | P1 |

### 4.4 MarshantaMX Integration

| ID | Requirement | Priority |
|----|-------------|----------|
| MM-001 | Agent MUST support Marshanta community node protocols | P2 |
| MM-002 | Agent MUST support Marshanta identity/permission system | P2 |
| MM-003 | Agent MUST support Marshanta resource sharing contracts | P3 |

### 4.5 MOLIE Integration

| ID | Requirement | Priority |
|----|-------------|----------|
| MOLIE-001 | Agent MUST expose LSP-compatible interface | P1 |
| MOLIE-002 | Agent MUST support MOLIE Agent Mode UI integration | P1 |
| MOLIE-003 | Agent MUST support MOLIE tool calling format | P1 |

### 4.6 MCP (Model Context Protocol) Integration

| ID | Requirement | Priority |
|----|-------------|----------|
| MCP-001 | Agent MUST support MCP client (stdio transport) | P2 |
| MCP-002 | Agent MUST support MCP server auto-discovery | P3 |
| MCP-003 | Agent MUST support dynamic MCP tool registration | P2 |

---

## 5. Permission & Security Requirements

### 5.1 Capability System

| ID | Requirement | Priority |
|----|-------------|----------|
| CAP-001 | System MUST implement capability-based security | P0 |
| CAP-002 | Capabilities MUST be attenuatable (delegate subset) | P1 |
| CAP-003 | Capabilities MUST have expiration/scope constraints | P2 |
| CAP-004 | System MUST support capability revocation | P1 |

#### Core Capabilities (Required)

```rust
// File operations
capability FileRead { path: PathPattern }
capability FileWrite { path: PathPattern, operation: Create|Modify|Delete }

// Process execution
capability ProcessExec { 
    binary: String, 
    args: Vec<String>,
    timeout_ms: u64 
}

// Network operations
capability NetworkEgress { 
    host: HostPattern, 
    port: PortRange,
    protocol: Tcp|Udp|Http|Https 
}

// Agent operations
capability AgentSpawn { 
    agent_type: String,
    max_turns: u32,
    allowed_tools: Vec<String>
}

// VCS operations
capability VcsRead { repo: PathPattern }
capability VcsMutate { repo: PathPattern, operation: Commit|Push|Merge|Rebase }

// Memory operations
capability MemoryRead { namespace: String }
capability MemoryWrite { namespace: String }
```

### 5.2 Permission Levels

| Level | Behavior | Use Case |
|-------|----------|----------|
| `AutoAllow` | Execute without prompt | Read-only safe operations |
| `AskOnce` | Prompt user once per operation | Single destructive operation |
| `AskSession` | Prompt once, allow for session | Repeated similar operations |
| `Deny` | Reject with explanation | Blocked operations |
| `RequireApproval` | Require external approval (e.g., human in loop) | Critical operations |

### 5.3 Sandboxing Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| SB-001 | Tool execution MUST be sandboxed | P0 |
| SB-002 | Sandbox MUST prevent filesystem escape | P0 |
| SB-003 | Sandbox MUST prevent network egress without capability | P0 |
| SB-004 | Sandbox MUST prevent privilege escalation | P0 |
| SB-005 | Sandbox MUST support resource limits (CPU, memory, time) | P1 |
| SB-006 | Sandbox MUST be implemented via WASM (preferred) or containers | P0 |

### 5.4 Secret Management

| ID | Requirement | Priority |
|----|-------------|----------|
| SEC-001 | System MUST scan tool inputs for secrets | P1 |
| SEC-002 | System MUST warn before writing secrets to files | P1 |
| SEC-003 | System MUST support secret redaction in logs | P1 |
| SEC-004 | System MUST integrate with HGI secret vault | P2 |

---

## 6. Audit & Compliance Requirements

### 6.1 Audit Logging

| ID | Requirement | Priority |
|----|-------------|----------|
| AUDIT-001 | ALL tool executions MUST be logged | P0 |
| AUDIT-002 | ALL permission decisions MUST be logged | P0 |
| AUDIT-003 | ALL capability grants MUST be logged | P0 |
| AUDIT-004 | ALL agent spawns MUST be logged | P0 |
| AUDIT-005 | Logs MUST include timestamp, actor, action, result | P0 |
| AUDIT-006 | Logs MUST be tamper-evident (signed) | P2 |
| AUDIT-007 | Logs MUST be exportable (JSON/CSV) | P1 |
| AUDIT-008 | Logs MUST support HGI-EDGE audit stream | P0 |

### 6.2 Audit Log Format

```json
{
  "timestamp": "2026-05-19T14:32:01Z",
  "sequence": 42,
  "actor": {
    "type": "agent",
    "id": "agent-uuid",
    "parent_id": "parent-uuid",
    "session_id": "session-uuid"
  },
  "action": {
    "type": "tool_execution",
    "tool": "FileWrite",
    "input_hash": "sha256:abc123...",
    "capabilities": ["FileWriteCap:/workspace/foo.txt"]
  },
  "result": {
    "status": "success",
    "duration_ms": 45,
    "output_hash": "sha256:def456..."
  },
  "permissions": {
    "decision": "AutoAllow",
    "rule_matched": "FileWriteCap in workspace"
  }
}
```

### 6.3 Human Approval Gates

| ID | Requirement | Priority |
|----|-------------|----------|
| APPR-001 | System MUST support configurable approval gates | P1 |
| APPR-002 | Gates MUST support file write approval | P1 |
| APPR-003 | Gates MUST support shell command approval | P1 |
| APPR-004 | Gates MUST support network egress approval | P1 |
| APPR-005 | Gates MUST support agent spawn approval | P1 |
| APPR-006 | Gates MUST support external webhook approval | P2 |
| APPR-007 | Gates MUST support multi-party approval (M-of-N) | P3 |

### 6.4 Data Retention

| ID | Requirement | Priority |
|----|-------------|----------|
| RET-001 | User MUST be able to configure retention policy | P2 |
| RET-002 | System MUST support automatic log rotation | P1 |
| RET-003 | System MUST support complete data deletion | P1 |
| RET-004 | System MUST NOT transmit logs to external servers without consent | P0 |

---

## 7. Worker & Daemon Compatibility

### 7.1 Worker Node Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| WORKER-001 | Agent MUST support deployment on worker nodes | P1 |
| WORKER-002 | Agent on worker MUST operate with limited capabilities | P0 |
| WORKER-003 | Agent on worker MUST delegate to orchestrator for escalations | P1 |
| WORKER-004 | Agent on worker MUST support offline operation (with sync) | P2 |

### 7.2 Daemon Mode

| ID | Requirement | Priority |
|----|-------------|----------|
| DAEMON-001 | Agent MUST support daemon/background mode | P2 |
| DAEMON-002 | Daemon MUST support task queue consumption | P2 |
| DAEMON-003 | Daemon MUST support graceful shutdown | P1 |
| DAEMON-004 | Daemon MUST support health check endpoint | P1 |
| DAEMON-005 | Daemon MUST support log rotation | P1 |

---

## 8. Deployment Requirements

### 8.1 Platform Support

| Platform | Support Level | Notes |
|----------|---------------|-------|
| Linux (x64) | ✅ Required | Primary target |
| Linux (ARM64) | ✅ Required | Edge devices |
| Windows (x64) | ✅ Required | VistaDev primary platform |
| Windows (ARM64) | ⚠️ Desired | Modern laptops |
| macOS (x64/ARM64) | ⚠️ Desired | Developer machines |
| WebAssembly | ⚠️ Future | Browser-based agents |

### 8.2 Resource Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **Agent runtime only** | 512 MB RAM | 1 GB RAM |
| **With local LLM (7B)** | 8 GB RAM | 16 GB RAM |
| **With local LLM (13B)** | 16 GB RAM | 32 GB RAM |
| **Disk** | 1 GB | 5 GB |
| **CPU** | 2 cores | 4+ cores |

### 8.3 Installation Methods

| Method | Priority | Notes |
|--------|----------|-------|
| Cargo install | P1 | `cargo install hgi-agent` |
| Homebrew | P2 | macOS/Linux |
| Windows MSI | P1 | VistaDev bundle |
| Docker | P2 | For server deployment |
| hgi-local-node plugin | P0 | Integrated with node |

### 8.4 Configuration

| ID | Requirement | Priority |
|----|-------------|----------|
| CFG-001 | Configuration MUST support JSON/YAML/TOML | P1 |
| CFG-002 | Configuration MUST support user and project scopes | P1 |
| CFG-003 | Configuration MUST support environment variable overrides | P1 |
| CFG-004 | Configuration MUST support CLI flag overrides | P1 |
| CFG-005 | Configuration MUST validate on load | P1 |
| CFG-006 | Configuration changes MUST NOT require restart (hot reload) | P2 |

### 8.5 Environment Variables

```bash
# Core
HGI_AGENT_ENDPOINT=http://localhost:8787
HGI_AGENT_LOG_LEVEL=info
HGI_AGENT_DATA_DIR=~/.hgi/agent

# LLM
HGI_LLM_PROVIDER=local
HGI_LLM_ENDPOINT=http://localhost:11434
HGI_LLM_MODEL=qwen2.5-coder:7b
HGI_LLM_API_KEY=                    # For cloud providers

# Permissions
HGI_PERMISSION_MODE=interactive     # interactive|auto|locked
HGI_APPROVAL_GATE=file_write,shell  # Comma-separated gates

# Integrations
HGI_EDGE_ENDPOINT=ws://localhost:8788
HGI_NODE_ID=                        # For worker mode
HGI_REDACTED_VECINAL_ENDPOINT=      # For RV integration
```

---

## 9. Use Case Scenarios

### 9.1 MOLIE Agent Mode

**Scenario:** User in VistaDev IDE wants AI assistance with coding task.

**Flow:**
1. User triggers MOLIE Agent Mode
2. hgi-local-node spawns agent with `MOLIE` agent type
3. Agent gets capabilities: `FileRead`, `FileEdit`, `LSP`, `Bash (safe only)`
4. Agent executes tool loop with user approval gates
5. All actions logged to HGI-EDGE audit stream
6. User can approve/deny at each step via IDE UI

### 9.2 RedVecinalMX Automation

**Scenario:** Community worker node runs automated workflows.

**Flow:**
1. Workflow definition loaded from RedVecinal orchestrator
2. Agent spawns with workflow-specific capabilities
3. Agent executes tasks against local resources
4. Results reported back to orchestrator
5. Full audit trail available to community admins

### 9.3 MarshantaMX Community Agent

**Scenario:** Marshanta node offers shared agent service.

**Flow:**
1. User authenticates with Marshanta identity
2. Agent spawns with capabilities based on user's role
3. Agent respects Marshanta resource sharing contracts
4. Usage tracked for fair resource allocation
5. Community governance can audit all agent actions

### 9.4 VistaDev Internal Automation

**Scenario:** CI/CD pipeline uses HGI agent for code review.

**Flow:**
1. Pipeline triggers agent in daemon mode
2. Agent loads project-specific playbook
3. Agent performs read-only analysis (Plan mode)
4. Agent generates review report
5. Report submitted as PR comment

### 9.5 HGI App Automation

**Scenario:** HGI-powered application uses agent for user tasks.

**Flow:**
1. Application embeds hgi-local-node
2. Node spawns agent with app-specific tool set
3. Agent operates within app sandbox
4. User controls data sharing
5. No data leaves device without explicit consent

---

## 10. Non-Functional Requirements

### 10.1 Performance

| ID | Requirement | Target |
|----|-------------|--------|
| PERF-001 | Tool execution latency | <100ms for read-only |
| PERF-002 | Agent spawn time | <500ms |
| PERF-003 | Context checkpoint | <1s |
| PERF-004 | Audit log write | <10ms |
| PERF-005 | Concurrent agents | ≥10 per node |

### 10.2 Reliability

| ID | Requirement | Target |
|----|-------------|--------|
| REL-001 | Agent crash recovery | 99.9% success |
| REL-002 | State persistence | No data loss on crash |
| REL-003 | Graceful degradation | Works with limited capabilities |
| REL-004 | Error recovery | Automatic retry with backoff |

### 10.3 Usability

| ID | Requirement | Priority |
|----|-------------|----------|
| UX-001 | Error messages MUST be actionable | P0 |
| UX-002 | Permission prompts MUST show context | P0 |
| UX-003 | Audit logs MUST be human-readable | P1 |
| UX-004 | Configuration MUST have sensible defaults | P1 |
| UX-005 | Documentation MUST include examples | P1 |

### 10.4 Privacy

| ID | Requirement | Priority |
|----|-------------|----------|
| PRIV-001 | NO telemetry without explicit opt-in | P0 |
| PRIV-002 | User MUST be able to export all data | P1 |
| PRIV-003 | User MUST be able to delete all data | P1 |
| PRIV-004 | Data MUST stay local by default | P0 |
| PRIV-005 | Cloud providers MUST be explicitly configured | P0 |

---

## 11. API Specification (Draft)

### 11.1 Core Types

```rust
// Agent types
enum AgentType {
    Interactive,    // User-facing agent
    Worker,         // Background worker
    SubAgent,       // Spawned by another agent
    Daemon,         // Long-running service
}

// Agent configuration
struct AgentConfig {
    agent_type: AgentType,
    max_turns: u32,
    timeout_ms: u64,
    allowed_tools: Vec<String>,
    capabilities: Vec<Capability>,
    llm_config: LlmConfig,
}

// Tool definition
struct Tool {
    name: String,
    description: String,
    input_schema: JsonSchema,
    is_read_only: bool,
    is_concurrency_safe: bool,
    required_capabilities: Vec<CapabilityDef>,
}

// Tool execution result
enum ToolResult {
    Success { content: String, artifacts: Vec<Artifact> },
    Error { message: String, recoverable: bool },
}
```

### 11.2 Core Interfaces

```rust
// Agent orchestration
trait AgentOrchestrator {
    fn spawn(&self, config: AgentConfig) -> Result<AgentHandle, SpawnError>;
    fn list(&self) -> Vec<AgentInfo>;
    fn terminate(&self, handle: AgentHandle) -> Result<(), TerminateError>;
}

// Tool registry
trait ToolRegistry {
    fn register(&mut self, tool: Tool) -> Result<(), RegisterError>;
    fn unregister(&mut self, name: &str) -> Result<(), UnregisterError>;
    fn resolve(&self, name: &str) -> Option<&Tool>;
    fn list(&self) -> Vec<&Tool>;
}

// Permission engine
trait PermissionEngine {
    fn check(&self, tool: &str, input: &JsonValue, capabilities: &[Capability]) 
        -> PermissionDecision;
    fn grant(&mut self, capability: Capability) -> Result<(), GrantError>;
    fn revoke(&mut self, capability: &Capability) -> Result<(), RevokeError>;
}

// Audit logger
trait AuditLogger {
    fn log(&self, entry: AuditEntry) -> Result<(), LogError>;
    fn query(&self, filter: AuditFilter) -> Vec<AuditEntry>;
    fn export(&self, format: ExportFormat) -> Result<Vec<u8>, ExportError>;
}
```

---

## 12. Implementation Roadmap

### Phase 1: Core Runtime (Q3 2026)
- [ ] Tool registry with WASM sandbox
- [ ] Capability-based permission engine
- [ ] Basic agent loop with turn limits
- [ ] Local LLM integration (Ollama, llama.cpp)
- [ ] HGI-EDGE integration

### Phase 2: Integration (Q4 2026)
- [ ] hgi-local-node module
- [ ] LSP protocol support
- [ ] MOLIE Agent Mode integration
- [ ] MCP client support
- [ ] Audit logging to HGI-EDGE

### Phase 3: Advanced Features (Q1 2027)
- [ ] Sub-agent orchestration
- [ ] Playbook/workflow system
- [ ] Memory system with vector search
- [ ] Human approval gates
- [ ] RedVecinalMX integration

### Phase 4: Scale & Polish (Q2 2027)
- [ ] Windows native support
- [ ] Daemon mode
- [ ] Performance optimization
- [ ] Documentation & examples
- [ ] Security audit

---

## 13. References

### External Specifications
- [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [WASM Component Model](https://component-model.bytecodealliance.org/)

### HGI Internal Documents
- `HGI-EDGE-RUNTIME Architecture`
- `hgi-local-node Module System`
- `RedVecinalMX Worker Protocol`
- `MarshantaMX Community Contracts`

### Related Audits
- `HGI_OPENMONOAGENT_AUDIT.md` (this repo)

---

**End of Document**
