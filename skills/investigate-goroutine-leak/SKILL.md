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
kubectl port-forward <pod-name> 8888:<ops-port> -n <namespace>

# Common paths (try these)
curl -s "http://localhost:8888/debug/pprof/goroutine?debug=1"
curl -s "http://localhost:8888/-/debug/pprof/goroutine?debug=1"
```

### 3. Get goroutine counts (baseline)
```bash
# Total count
curl -s "http://localhost:<port>/-/debug/pprof/goroutine?debug=1" | head -1

# Summary - shows count per stack trace
curl -s "http://localhost:<port>/-/debug/pprof/goroutine?debug=1" | grep -E "^[0-9]+ @" | sort -rn | head -20
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

## Advanced Analysis

### Goroutine Ratio Analysis
For gRPC leaks, compare related goroutine counts:
```bash
# Save full dump (ALWAYS use debug=2 for counting - debug=1 shows hex addresses, not names)
curl -s "http://localhost:<port>/-/debug/pprof/goroutine?debug=2" > /tmp/goroutines.txt

# Count actual goroutines by function (use .run or specific function to avoid double-counting)
grep -c "CallbackSerializer.run" /tmp/goroutines.txt
grep -c "loopyWriter" /tmp/goroutines.txt
grep -c "http2Client.*reader" /tmp/goroutines.txt
```

**IMPORTANT:** Always use `debug=2` for counting goroutines by type. The `debug=1` output shows aggregated stacks with hex addresses, not function names - grepping for function names will give incorrect results.

**Expected ratio:** Each active gRPC connection should have ~1 CallbackSerializer, 1 loopyWriter, 1 reader.
**Leak indicator:** High CallbackSerializer count with low loopyWriter/reader count = orphaned goroutines.

### Goroutine Age Analysis
Check how long goroutines have been running:
```bash
# Find goroutines with their ages
grep -E "goroutine [0-9]+ \[" /tmp/goroutines.txt | head -30
```

Output like `goroutine 12345 [select, 41 minutes]:` shows goroutine stuck for 41 minutes.
- Scattered ages (1min, 5min, 20min, 41min) = continuous leak during runtime
- All same age = leak at startup

### Find Leak Creation Source
```bash
# Find which parent goroutines created the leaks
grep "created by" /tmp/goroutines.txt | sort | uniq -c | sort -rn | head -10
```

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

### gRPC Error Path Leak
**Symptom:** Slow steady growth of gRPC goroutines
**Cause:** gRPC connection not closed on error paths
**Fix:** Close connection in ALL error paths:
```go
conn, err := grpc.NewClient(addr, opts...)
if err != nil { return err }

result, err := useConnection(conn)
if err != nil {
    conn.Close()  // Don't forget error paths!
    return err
}
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

### gRPC Stream Not Closed Before Connection
**Symptom:** CallbackSerializer goroutines leak even when `conn.Close()` is called
**Cause:** Bidirectional streams (e.g., `ProxyCDP`, streaming RPCs) not properly closed before connection
**Fix:** Always call `CloseSend()` on the stream before closing the connection:
```go
func (b *grpcBackend) Close() error {
    _ = b.stream.CloseSend()  // Signal stream completion first
    return b.conn.Close()
}
```

### gRPC Library-Level Leak
**Symptom:** CallbackSerializer leaks persist even after closing connections and streams properly
**Evidence:** Transport goroutines (reader, loopyWriter) exit but CallbackSerializer persists
**Cause:** Known gRPC-go issue where `Close()` returns before internal goroutines terminate

**Diagnosis:**
```bash
# If this ratio is way off (e.g., 60 CallbackSerializer vs 6 loopyWriter), it's a library issue
grep -c "CallbackSerializer" /tmp/goroutines.txt  # Should roughly match...
grep -c "loopyWriter" /tmp/goroutines.txt         # ...this count
```

**Fix:** Upgrade gRPC to latest version:
```bash
# Check current version
grep "google.golang.org/grpc" go.mod

# Check for fixes in newer versions
# Search: https://github.com/grpc/grpc-go/issues?q=goroutine+leak

# Upgrade (may require Go version bump)
go get google.golang.org/grpc@latest
go mod tidy
```

**Relevant gRPC issues:**
- [#8655](https://github.com/grpc/grpc-go/issues/8655) - Goroutines outlive Close()
- [#8746](https://github.com/grpc/grpc-go/pull/8746) - Wait for goroutines on close
- [#6413](https://github.com/grpc/grpc-go/issues/6413) - CallbackSerializer leak report

### gRPC DNS Resolver Leak (Common since gRPC v1.63)
**Symptom:** CallbackSerializer goroutines leak even with proper connection closing
**Cause:** Since gRPC v1.63, `grpc.NewClient()` defaults to DNS resolver which spawns CallbackSerializer goroutines for DNS watching. When connecting to IPs directly, these goroutines don't clean up properly on `Close()`.

**Diagnosis:**
```bash
# Check if connecting to IPs (not hostnames)
grep -r "grpc.NewClient\|grpc.Dial" --include="*.go" .

# If targets are IPs like "10.0.0.5:4444", DNS resolver is unnecessary
```

**Fix:** Use `passthrough` resolver to bypass DNS:
```go
// Before - uses DNS resolver (leaks)
conn, err := grpc.NewClient(addr, opts...)

// After - passthrough resolver (no leak)
conn, err := grpc.NewClient("passthrough:///"+addr, opts...)
```

**CRITICAL:** When applying this fix, search for ALL `grpc.NewClient` and `grpc.Dial` calls in the codebase:
```bash
grep -rn "grpc.NewClient\|grpc.Dial" --include="*.go" .
```
Apply passthrough to ALL locations that connect to IPs, not just the one you're investigating.

## Verification
After deploying fix:
```bash
# Baseline - total count
curl -s "http://localhost:<port>/-/debug/pprof/goroutine?debug=1" | head -1

# IMPORTANT: Use debug=2 for counting specific goroutine types (debug=1 uses hex addresses)
curl -s "http://localhost:<port>/-/debug/pprof/goroutine?debug=2" > /tmp/goroutines.txt
grep -c "CallbackSerializer.run" /tmp/goroutines.txt

# Check dead parent goroutines (leaked children)
for parent in $(grep "created by.*in goroutine" /tmp/goroutines.txt | sed 's/.*in goroutine //' | sort -u); do
    grep -q "^goroutine $parent " /tmp/goroutines.txt || echo "DEAD: $parent"
done | wc -l

# Wait 10-30 min, repeat and compare counts
```

**Success criteria:**
- Total goroutine count stabilizes (some fluctuation OK, but no steady linear growth)
- Specific leak pattern (e.g., CallbackSerializer) stays proportional to active connections
- Dead parent count stays at 0 or very low (only startup goroutines)
- No steady linear growth over time
