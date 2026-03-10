# Task: Architecture Refactor — Session Manager

> Create session management layer: player sessions, state persistence, multi-instance orchestration.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P2 — Cloud-Native |
| **Estimated Complexity** | Medium |
| **Estimated Effort** | 2 weeks |
| **Dependencies** | Container system |
| **Blocks** | Streaming gateway, Azure deployment |
| **Phase** | Phase 4 — Cloud-Native |

---

## Objective

Build a session management service that orchestrates multiple concurrent Quake engine containers, managing lifecycle (create/destroy), health monitoring, resource allocation, and session routing.

---

## Architecture

```
┌──────────────┐     ┌─────────────────────────────────────┐
│   Clients    │     │        Session Manager               │
│  (Browser/   │     │                                      │
│   Gateway)   │     │  REST API:                           │
│              │────▶│  POST   /api/sessions                │
│              │     │  GET    /api/sessions                 │
│              │     │  GET    /api/sessions/{id}            │
│              │     │  DELETE /api/sessions/{id}            │
│              │     │  GET    /api/health                   │
│              │     │                                      │
│              │     │  Session Registry (in-memory + Redis) │
│              │     │  Container Orchestrator (K8s API)     │
│              │     └──────────────┬────────────────────────┘
│              │                    │
│              │     ┌──────────────▼────────────────────────┐
│              │     │     Kubernetes / Container Runtime     │
│              │     │  ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│              │     │  │Engine 1 │ │Engine 2 │ │Engine N │ │
│              │     │  │Session A│ │Session B│ │Session C│ │
│              │     │  └─────────┘ └─────────┘ └─────────┘ │
│              │     └───────────────────────────────────────┘
└──────────────┘
```

---

## API Design

### POST /api/sessions — Create Session

```json
Request:
{
    "map": "start",
    "maxPlayers": 8,
    "gameMode": "deathmatch",
    "config": {
        "QUAKE_MEMORY": "32",
        "QUAKE_GAMEDIR": "id1"
    }
}

Response (201 Created):
{
    "id": "session-abc123",
    "status": "starting",
    "endpoint": "10.0.1.5:26000",
    "healthUrl": "http://10.0.1.5:8080/healthz",
    "createdAt": "2024-01-15T10:30:00Z"
}
```

### GET /api/sessions — List Sessions

```json
Response (200 OK):
{
    "sessions": [
        {
            "id": "session-abc123",
            "status": "running",
            "players": 3,
            "maxPlayers": 8,
            "map": "dm4",
            "uptime": "2h15m",
            "endpoint": "10.0.1.5:26000"
        }
    ],
    "total": 1
}
```

### GET /api/sessions/{id} — Get Session Details

### DELETE /api/sessions/{id} — Terminate Session

Sends SIGTERM to the container, waits for graceful shutdown.

---

## Implementation

### Technology Choice

| Option | Language | Pros | Cons |
|--------|----------|------|------|
| **Go** | Go | Fast, K8s native client, lightweight | New language |
| **Python (FastAPI)** | Python | Rapid development, easy deployment | Runtime overhead |
| **Node.js** | JavaScript | Good WebSocket integration | Not ideal for system programming |

**Recommended**: Go — native Kubernetes client library, low resource usage, compiled binary.

### Session State Machine

```
                 POST /sessions
                      │
                      ▼
              ┌──────────────┐
              │  CREATING    │
              │ (Pod pending)│
              └──────┬───────┘
                     │ Pod running + health OK
                     ▼
              ┌──────────────┐
              │   RUNNING    │
              │ (Active game)│
              └──────┬───────┘
                     │ DELETE or timeout
                     ▼
              ┌──────────────┐
              │  STOPPING    │
              │ (Draining)   │
              └──────┬───────┘
                     │ Pod terminated
                     ▼
              ┌──────────────┐
              │  TERMINATED  │
              │ (Cleaned up) │
              └──────────────┘
```

### Kubernetes Integration

```go
// Create engine pod
func (sm *SessionManager) CreateSession(config SessionConfig) (*Session, error) {
    pod := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "quake-session-" + config.ID,
            Namespace: "quake",
            Labels: map[string]string{
                "app":     "quake-engine",
                "session": config.ID,
            },
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{{
                Name:  "quake-engine",
                Image: "quake-engine:latest",
                Env: []corev1.EnvVar{
                    {Name: "QUAKE_PORT", Value: "26000"},
                    {Name: "QUAKE_MAP", Value: config.Map},
                    {Name: "QUAKE_MAXPLAYERS", Value: fmt.Sprintf("%d", config.MaxPlayers)},
                },
                Ports: []corev1.ContainerPort{
                    {ContainerPort: 26000, Protocol: "UDP"},
                    {ContainerPort: 8080, Protocol: "TCP"},
                },
                LivenessProbe: &corev1.Probe{
                    ProbeHandler: corev1.ProbeHandler{
                        HTTPGet: &corev1.HTTPGetAction{
                            Path: "/healthz",
                            Port: intstr.FromInt(8080),
                        },
                    },
                },
            }},
        },
    }
    // Create pod via Kubernetes API
    return sm.k8sClient.CoreV1().Pods("quake").Create(ctx, pod, metav1.CreateOptions{})
}
```

---

## Files to Create

| File | Description |
|------|-------------|
| `services/session-manager/main.go` | Entry point |
| `services/session-manager/api/handlers.go` | REST API handlers |
| `services/session-manager/api/routes.go` | Route definitions |
| `services/session-manager/session/manager.go` | Session lifecycle management |
| `services/session-manager/session/state.go` | Session state machine |
| `services/session-manager/k8s/client.go` | Kubernetes client wrapper |
| `services/session-manager/Dockerfile` | Container build |
| `services/session-manager/go.mod` | Go module definition |

---

## Testing Requirements

| Test | Description |
|------|-------------|
| `test_create_session` | POST /api/sessions creates a pod |
| `test_list_sessions` | GET /api/sessions returns active sessions |
| `test_get_session` | GET /api/sessions/{id} returns session details |
| `test_delete_session` | DELETE /api/sessions/{id} terminates the pod |
| `test_session_health` | Session manager detects unhealthy sessions |
| `test_session_timeout` | Idle sessions are terminated after timeout |
| `test_concurrent_sessions` | 10+ sessions created simultaneously |
| `test_session_cleanup` | Terminated session pods are cleaned up |

---

## Acceptance Criteria

- [ ] REST API creates game sessions (responds within 10 seconds)
- [ ] REST API lists active sessions with status and player count
- [ ] REST API terminates sessions cleanly
- [ ] Failed/crashed sessions are detected and cleaned up
- [ ] Idle sessions auto-terminate after configurable timeout (default 5 min)
- [ ] Supports 10+ concurrent sessions
- [ ] Session manager itself has health endpoint
- [ ] Session manager runs as a Kubernetes Deployment
- [ ] Session state persists across session manager restarts (optional: Redis)
