# Task: Performance Benchmarks — Latency & Scalability

> Performance benchmarks: frame latency, concurrent sessions, network throughput, scaling behavior.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P2 — Testing |
| **Estimated Complexity** | Medium |
| **Estimated Effort** | 1 week |
| **Dependencies** | Container system, Streaming gateway, Frame encoding |
| **Blocks** | Production capacity planning, SLA definition |
| **Phase** | Phase 4 — Cloud-Native (Testing) |

---

## Objective

Establish performance baselines and validate scalability characteristics of the cloud-native Quake streaming platform. Benchmarks cover rendering, encoding, networking, and multi-session scaling to determine production capacity and inform infrastructure sizing.

---

## Benchmark Categories

### BENCH-1: Engine Rendering Performance

Measure core engine rendering throughput in headless mode.

| Metric | Method | Target |
|--------|--------|--------|
| **Frames per second (FPS)** | `timedemo demo1` | Establish baseline |
| **Frame render time (ms)** | Instrumented `R_RenderView()` at `r_main.c:1049` | ≤ 10ms at 640×480 |
| **BSP traversal time** | Instrumented `R_RecursiveWorldNode()` in `r_bsp.c` | ≤ 2ms |
| **Entity rendering time** | Instrumented `R_DrawEntitiesOnList()` in `r_main.c` | ≤ 3ms |

**Test procedure**:
```bash
# Run timedemo benchmark
./quake-headless -timedemo demo1 -width 640 -height 480

# Expected output:
# XX.X seconds, XXX.X fps
```

**Comparison matrix**:

| Configuration | Expected FPS | Notes |
|--------------|-------------|-------|
| With assembly (x86) | Baseline | Original performance |
| Without assembly (C only) | 50-80% of baseline | C fallback overhead |
| With SIMD intrinsics | 80-95% of baseline | SSE2 optimized |
| 320×240 resolution | ~2x baseline | Quarter pixels |
| 1280×720 resolution | ~25% of baseline | 4x pixels |

### BENCH-2: Frame Encoding Performance

| Metric | Method | Target |
|--------|--------|--------|
| **Palette → RGB conversion** | Timed extraction from `vid.buffer` | ≤ 1ms at 640×480 |
| **RGB → YUV420 conversion** | Timed color space conversion | ≤ 1ms (scalar), ≤ 0.3ms (SIMD) |
| **H.264 encode time** | Timed libx264 `ultrafast` encode | ≤ 10ms at 640×480 |
| **Total encode pipeline** | Extract + convert + encode | ≤ 12ms at 640×480 |
| **Encode throughput** | Max FPS at each resolution | ≥ 60 FPS at 640×480 |

**Resolution matrix**:

| Resolution | Target Encode Time | Min FPS |
|-----------|-------------------|---------|
| 320×240 | ≤ 5ms | 60 |
| 640×480 | ≤ 10ms | 60 |
| 800×600 | ≤ 16ms | 30 |
| 1280×720 | ≤ 16ms (hardware) | 30 |

### BENCH-3: Network Performance

| Metric | Method | Target |
|--------|--------|--------|
| **WebSocket frame delivery** | Time from encode complete to client receive | ≤ 5ms (localhost) |
| **Input injection latency** | Time from WebSocket receive to `IN_InjectRemoteInput()` | ≤ 1ms |
| **Bandwidth per session** | Video + audio stream bandwidth | ≤ 5 Mbps at 640×480 |
| **Protocol overhead** | WebSocket framing overhead | ≤ 5% of payload |

**Network protocol analysis**:

| Stream | Bandwidth (30 FPS) | Bandwidth (60 FPS) |
|--------|-------------------|-------------------|
| Video (H.264 ultrafast) | 1-3 Mbps | 2-5 Mbps |
| Audio (PCM 44.1kHz stereo) | 1.4 Mbps | 1.4 Mbps |
| Audio (Opus 128kbps) | 128 Kbps | 128 Kbps |
| Input (JSON) | < 10 Kbps | < 20 Kbps |
| **Total (Opus audio)** | **~1.5-3.5 Mbps** | **~2.5-5.5 Mbps** |

### BENCH-4: End-to-End Latency

| Stage | Metric | Target | Measurement Point |
|-------|--------|--------|-------------------|
| Render | Frame render time | ≤ 10ms | `R_RenderView()` exit at `r_main.c:1049` |
| Extract | Buffer copy time | ≤ 1ms | `VID_GetFramebuffer()` return |
| Convert | Color space conversion | ≤ 1ms | Pre-encode timing |
| Encode | Video encoding | ≤ 10ms | Post-encode timing |
| Network | WebSocket send/receive | ≤ 10ms | Client-side timestamp |
| Decode | Client H.264 decode | ≤ 5ms | Post-decode timestamp |
| Display | Canvas render | ≤ 1ms | `requestAnimationFrame` timing |
| **Total** | **End-to-end** | **≤ 38ms** | Input → display measurement |

**Latency measurement method**:
```
Client sends input with timestamp T0
Engine receives input at T1 (T1-T0 = network + parse)
Engine renders frame containing response at T2
Gateway encodes frame at T3
Client receives frame at T4
Client displays frame at T5

E2E latency = T5 - T0
Measurable: T5 - T0 via NTP-synchronized clocks or round-trip
```

### BENCH-5: Concurrent Session Scaling

| Metric | Method | Target |
|--------|--------|--------|
| **Max sessions per node** | Scale up sessions until CPU saturated | ≥ 4 on 4-core node |
| **Session creation time** | Measure Pod startup time | ≤ 10 seconds |
| **Per-session CPU** | Monitor container CPU usage | ≤ 1 core |
| **Per-session memory** | Monitor container memory | ≤ 128 MB |
| **Per-session bandwidth** | Monitor network I/O | ≤ 5 Mbps |

**Scaling test procedure**:
```
1. Start with 1 session, measure baseline metrics
2. Add sessions one at a time
3. At each level, measure:
   - FPS per session
   - Latency per session (P50, P95, P99)
   - CPU utilization
   - Memory usage
4. Continue until FPS drops below 30 or latency exceeds 50ms
5. Record max sustainable sessions
```

**Expected scaling behavior**:

| Node Size | vCPUs | Expected Sessions | Total Bandwidth |
|-----------|-------|-------------------|-----------------|
| Standard_B2s | 2 | 1-2 | 10 Mbps |
| Standard_D4s_v3 | 4 | 3-4 | 20 Mbps |
| Standard_D8s_v3 | 8 | 6-8 | 40 Mbps |
| Standard_D16s_v3 | 16 | 12-16 | 80 Mbps |

### BENCH-6: Auto-Scaling Response Time

| Metric | Method | Target |
|--------|--------|--------|
| **Scale-up time** | Time from demand increase to pod ready | ≤ 30 seconds |
| **Scale-down time** | Time from demand decrease to pod terminated | ≤ 5 minutes |
| **HPA reaction** | Time from CPU threshold to scaling decision | ≤ 15 seconds |

---

## Benchmark Implementation

### Rendering Benchmark

```c
/* bench_render.c — Rendering performance benchmark */
#include "quakedef.h"
#include <time.h>

void Bench_Render(void) {
    struct timespec start, end;
    int num_frames = 1000;
    double total_ms = 0;

    for (int i = 0; i < num_frames; i++) {
        clock_gettime(CLOCK_MONOTONIC, &start);
        R_RenderView();
        clock_gettime(CLOCK_MONOTONIC, &end);

        double ms = (end.tv_sec - start.tv_sec) * 1000.0 +
                     (end.tv_nsec - start.tv_nsec) / 1000000.0;
        total_ms += ms;
    }

    double avg_ms = total_ms / num_frames;
    double fps = 1000.0 / avg_ms;
    Con_Printf("Benchmark: %d frames, avg %.2f ms/frame, %.1f FPS\n",
               num_frames, avg_ms, fps);
}
```

### Encoding Benchmark

```c
/* bench_encode.c — Encoding performance benchmark */
void Bench_Encode(void) {
    int width, height;
    const pixel_t *fb = VID_GetFramebuffer(&width, &height);

    uint8_t *rgb = malloc(width * height * 3);
    uint8_t *yuv = malloc(width * height * 3 / 2);

    /* Benchmark palette → RGB */
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < 1000; i++)
        extract_frame_software(rgb, &width, &height);
    clock_gettime(CLOCK_MONOTONIC, &end);
    double extract_ms = elapsed_ms(start, end) / 1000;

    /* Benchmark RGB → YUV */
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < 1000; i++)
        rgb_to_yuv420(rgb, width, height, yuv, yuv + width*height, yuv + width*height*5/4);
    clock_gettime(CLOCK_MONOTONIC, &end);
    double convert_ms = elapsed_ms(start, end) / 1000;

    Con_Printf("Extract: %.2f ms, Convert: %.2f ms\n", extract_ms, convert_ms);

    free(rgb);
    free(yuv);
}
```

### Scaling Benchmark Script

```bash
#!/bin/bash
# bench_scaling.sh — Concurrent session scaling benchmark

MAX_SESSIONS=20
RESULTS_FILE="scaling_results.csv"
echo "sessions,avg_fps,p95_latency_ms,cpu_percent,memory_mb" > $RESULTS_FILE

for n in $(seq 1 $MAX_SESSIONS); do
    echo "Testing with $n sessions..."

    # Create sessions
    for i in $(seq 1 $n); do
        curl -s -X POST http://localhost:8090/api/sessions \
            -d '{"map":"start"}' > /dev/null
    done

    # Wait for all ready
    sleep 15

    # Measure metrics (30 second sample)
    avg_fps=$(measure_fps $n 30)
    p95_lat=$(measure_latency $n 30)
    cpu=$(kubectl top pods -n quake --no-headers | awk '{sum+=$2} END {print sum}')
    mem=$(kubectl top pods -n quake --no-headers | awk '{sum+=$3} END {print sum}')

    echo "$n,$avg_fps,$p95_lat,$cpu,$mem" >> $RESULTS_FILE

    # Check if degraded
    if (( $(echo "$avg_fps < 30" | bc -l) )); then
        echo "FPS dropped below 30 at $n sessions. Stopping."
        break
    fi

    # Cleanup
    for i in $(seq 1 $n); do
        curl -s -X DELETE http://localhost:8090/api/sessions/session-$i > /dev/null
    done
    sleep 10
done
```

---

## Reporting

### Performance Report Format

```markdown
## Performance Benchmark Report — [Date]

### Environment
- Node: Standard_D4s_v3 (4 vCPU, 16 GB RAM)
- OS: Ubuntu 22.04
- Engine: quake-headless v1.0.0
- Encoder: libx264 (ultrafast, zerolatency)

### Results Summary
| Metric | Result | Target | Status |
|--------|--------|--------|--------|
| Render FPS (640×480) | XX FPS | Baseline | ✅/❌ |
| Encode time (640×480) | XX ms | ≤ 10ms | ✅/❌ |
| E2E latency (P95) | XX ms | ≤ 50ms | ✅/❌ |
| Max sessions/node | XX | ≥ 4 | ✅/❌ |
| Per-session bandwidth | X.X Mbps | ≤ 5 Mbps | ✅/❌ |

### Scaling Curve
[Chart: Sessions vs FPS, Sessions vs Latency]
```

---

## Acceptance Criteria

- [ ] Rendering baseline established (FPS at 320×240, 640×480, 1280×720)
- [ ] Encoding latency ≤ 10ms at 640×480 (H.264 ultrafast)
- [ ] End-to-end latency P95 ≤ 50ms
- [ ] Concurrent session scaling tested up to node saturation
- [ ] Per-session resource usage documented (CPU, memory, bandwidth)
- [ ] Auto-scaling response time ≤ 30 seconds
- [ ] Performance regression detection in CI (timedemo comparison)
- [ ] Benchmarks are reproducible and automated
- [ ] Results documented in standard report format
- [ ] Capacity planning recommendations provided
