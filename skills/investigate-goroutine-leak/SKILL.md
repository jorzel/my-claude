---
description: Investigate goroutine leaks in Go services using pprof analysis
---

# Investigate Go Goroutine Leaks

Investigate goroutine leaks in a Go service using pprof analysis.

## Prerequisites
- Service must have pprof endpoints enabled
- Port forwarding set up if running in Kubernetes

## Investigation Steps

### 1. Enable pprof endpoint
Check if pprof is enabled. Common env var: `PROFILER_ENDPOINT_ENABLED=true`

### 2. Access pprof endpoint
```bash
# For Kubernetes
kubectl port-forward <pod-name> 8888:<ops-port>

# Common paths (try these)
curl -s "http://localhost:8888/debug/pprof/goroutine?debug=1"
curl -s "http://localhost:8888/-/debug/pprof/goroutine?debug=1"
```

### 3. Get goroutine counts (baseline)
```bash
# Summary - shows count per stack trace
curl -s "http://localhost:<port>/-/debug/pprof/goroutine?debug=1" | grep -E "^[0-9]+ @" | sort -rn | head -20

# Total count
curl -s "http://localhost:<port>/-/debug/pprof/goroutine?debug=1" | head -1
```

### 4. Identify leaking goroutines
Look for high counts in the output. Common leak patterns:
- `grpc/internal/grpcsync.(*CallbackSerializer).run` - gRPC connection leak
- `net/http.(*persistConn).readLoop` - HTTP client connection leak
- `time.Sleep` or channel operations - goroutines stuck waiting

### 5. Get detailed stack traces
```bash
# Full stack trace for specific goroutine pattern
curl -s "http://localhost:<port>/-/debug/pprof/goroutine?debug=2" | grep -A 30 "<pattern>" | head -50

# Find parent goroutine that created the leak
curl -s "http://localhost:<port>/-/debug/pprof/goroutine?debug=2" | grep -B 5 "created by.*<pattern>"
```

### 6. Compare over time
Wait 30-60 minutes and repeat step 3. Growing counts indicate active leaks.

## Common Causes & Fixes

### gRPC Connection Pool Leak
**Symptom:** `CallbackSerializer.run` goroutines growing
**Cause:** Connections not closed when cache entries expire
**Fix:** Add `OnEvicted` callback to close connections:
```go
pool := cache.New(ttl, cleanupInterval)
pool.OnEvicted(func(_ string, value interface{}) {
    if executor, ok := value.(PooledExecutor); ok {
        executor.Close()
    }
})
```

### HTTP Client Leak
**Symptom:** `persistConn.readLoop` goroutines growing
**Cause:** Response body not closed
**Fix:** Always close response body:
```go
resp, err := client.Do(req)
if err != nil { return err }
defer resp.Body.Close()
```

### Channel/Context Leak
**Symptom:** Goroutines stuck on channel operations
**Cause:** Goroutine waiting on channel that's never closed/written to
**Fix:** Use context cancellation or ensure channels are properly closed

### gRPC Stream Leak
**Symptom:** gRPC transport goroutines growing
**Cause:** gRPC connection not closed on error paths
**Fix:** Close connection in all error paths:
```go
conn, err := grpc.NewClient(addr, opts...)
if err != nil { return err }

result, err := useConnection(conn)
if err != nil {
    conn.Close()  // Don't forget error paths!
    return err
}
```

## Verification
After deploying fix, monitor goroutine count - should stabilize instead of growing.
