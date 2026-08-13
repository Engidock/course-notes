# Golang Project Mastery

> **👋 Hey Fresher — Read This First!**

> Go (also called Golang) is the language that powers some of the biggest systems in the world — Docker, Kubernetes, Terraform, and Prometheus are all written in Go. Companies love Go because it is **fast, simple, and built for concurrency** (doing many things at the same time). You don't need to know everything before starting — we explain every concept step by step using a real company story. By the end, you will have 5 working Go programs that solve real company problems, and you will understand concurrency — the single most valuable Go skill in any interview.

> **Company in this project:** SwiftMetrics — a SaaS startup in Pune building a developer productivity analytics platform. 40 engineers, processing millions of events per day. You just joined as a Junior Go Developer. Your lead is Ananya. Let's build.

#### What You Will Build in This Project

You will build **5 production-quality Go programs** — ranging from a REST API to a concurrent file processor to a CLI tool — that SwiftMetrics actually needs for their platform. Each program introduces real Go concepts in a real business context.

REST API Server, CSV File Processor, Concurrent Worker Pool, CLI Tool, gRPC Service, goroutines, channels, structs & interfaces

> **🌐 Program 1 — REST API Server**

> Build a production HTTP server with routing, JSON request/response, middleware, error handling, and graceful shutdown using Go's standard library.

> Read millions of event rows from CSV files, parse and validate each record, aggregate metrics per team, and write summary reports — with proper error handling.

> Process thousands of API health-check jobs simultaneously using goroutines and channels — Go's answer to threading. See why Go beats Python 10x on concurrency.

> Build a command-line tool (like kubectl or terraform) that accepts flags, reads config files, calls an API, and formats output as tables or JSON.

> Define a Protocol Buffer schema, implement a gRPC server that streams live metrics, and write a client that receives and displays them in real time.

**Scene 1 — SwiftMetrics Engineering Office, Pune | Your First Morning**

> **Ananya** _Senior Go Engineer — SwiftMetrics Platform Team_
> 
> Welcome to SwiftMetrics. We process about 8 million developer events per day — commits, builds, deployments, PR reviews. Our entire backend is Go. Not because it's trendy — because we need the performance. Our Python prototype was maxing out at 80,000 events per minute. The Go rewrite handles 4 million per minute on the same hardware. That is the difference Go makes.

> **Rahul (You)** _Junior Go Developer — Day 1 at SwiftMetrics_
> 
> I've used Python and Java in college. I know Go has goroutines for concurrency, but I've never built anything real with it. Where do I start?

> **Ananya** _Senior Go Engineer_
> 
> Start with Go's fundamentals — they are simpler than Java and stricter than Python. Go has no classes, no inheritance, no exceptions. It has structs, interfaces, goroutines, and channels. Learn these four things deeply and you can build anything. Your first task is Program 1: a REST API for our metrics ingestion endpoint. No frameworks — just net/http from the standard library.

> **Vikas** _Go Architect — SwiftMetrics Infrastructure_
> 
> And learn to love error handling. In Go, errors are values returned from functions — not exceptions you throw and catch. Every function that can fail returns (result, error). You check the error every single time. It is verbose, but it means you can never accidentally ignore a failure. That discipline is what makes Go code reliable.

### 1. Go Foundations — Before You Write a Single Program

Before building the five programs, every fresher needs to understand Go's core concepts. Go is intentionally simple — it has very few keywords and no hidden magic. Everything is explicit.

> **Why Go? The Simple Explanation for Freshers**

> Go was created at Google in 2009 by the engineers who built Unix. They needed a language that compiled fast (like Python), ran fast (like C), and handled thousands of simultaneous tasks easily (unlike either). Go achieves this with three design choices: (1) static typing that compiles to a single binary — no runtime needed, (2) goroutines — thousands of lightweight threads for ₹1KB of memory each, and (3) explicit error handling instead of exceptions. The result: you can deploy a Go binary to any Linux server and it just runs, handling thousands of requests per second with a single process.

```
SwiftMetrics Go Project — Workspace Structure
===============================================

  swiftmetrics/
  ├── go.mod                        ← Module definition (like package.json in Node)
  ├── go.sum                        ← Dependency checksums (auto-generated)
  ├── cmd/
  │   ├── api/
  │   │   └── main.go              ← Program 1: REST API entry point
  │   ├── processor/
  │   │   └── main.go              ← Program 2: CSV Processor entry point
  │   ├── worker/
  │   │   └── main.go              ← Program 3: Worker Pool entry point
  │   ├── cli/
  │   │   └── main.go              ← Program 4: CLI Tool entry point
  │   └── grpc/
  │       └── main.go              ← Program 5: gRPC Service entry point
  ├── internal/
  │   ├── models/
  │   │   └── event.go             ← Shared data structures (structs)
  │   ├── api/
  │   │   ├── handlers.go          ← HTTP handler functions
  │   │   └── middleware.go        ← Logging, auth middleware
  │   └── worker/
  │       └── pool.go              ← Worker pool logic
  ├── proto/
  │   └── metrics.proto            ← gRPC service definition
  └── data/
      └── events.csv               ← Sample data for Program 2
```

1. Setting Up Go and Your First Module
Install Go from golang.org. Then initialise your module — this is Go's dependency management system (like pip for Python or npm for Node).

```
# Install Go: download from https://golang.org/dl/
# Verify installation:
go version   # go version go1.22.0 linux/amd64
# Create project and initialise module
mkdir swiftmetrics && cd swiftmetrics
go mod init github.com/swiftmetrics/platform
# go.mod is now created — it defines your module name and Go version
# Think of it like package.json in Node.js or requirements.txt in Python
# Install external packages (we'll use these later)
go get google.golang.org/grpc          # for Program 5
go get google.golang.org/protobuf      # for Program 5
# Build any program
go build ./cmd/api/                    # compiles to a binary
# Run directly without building
go run ./cmd/api/main.go

# Run all tests
go test ./...

# Format code (always before committing!)
go fmt ./...
```

#### Go Core Concepts — The Four Things That Matter

2. Structs — Go's Version of Classes (But Simpler)
Go has no classes. Instead it uses structs — named collections of fields. Methods are attached to structs separately. This is simpler and more explicit than class inheritance.

```
// internal/models/event.go
// Structs define the shape of your data
package models
import "time"
// Event represents a developer activity event from any team tool
type Event struct {
    ID        string // unique event identifier
    TeamID    string // which team triggered this event
    Type      string // "commit", "build", "deploy", "review"
    UserEmail string // who triggered it
    Timestamp time.Time // when it happened
    Duration  int64 // how long it took in milliseconds
    Success   bool // did it succeed?
}

// TeamSummary holds aggregated metrics for one team
type TeamSummary struct {
    TeamID      string
    TotalEvents int
    Successes   int
    Failures    int
    AvgDuration float64
}

// Method on the TeamSummary struct — like a class method but attached externally
func (ts *TeamSummary) SuccessRate() float64 {
    if ts.TotalEvents == 0 {
        return 0
    }
    return float64(ts.Successes) / float64(ts.TotalEvents) * 100
}

// Creating a struct instance:
event := Event{
    ID:        "evt-001",
    TeamID:    "team-alpha",
    Type:      "build",
    UserEmail: "rahul@swiftmetrics.io",
    Duration:  2400,
    Success:   true,
}
```

3. Interfaces — Define Behaviour, Not Inheritance
Interfaces in Go define a set of methods. Any type that has those methods automatically satisfies the interface — no explicit "implements" needed. This is called implicit satisfaction.

```
// Interfaces define behaviour — what a type CAN DO
type EventStore interface {
    Save(event Event) error
    FindByTeam(teamID string) ([]Event, error)
    Count() (int, error)
}

// InMemoryStore satisfies EventStore — without saying "implements"
type InMemoryStore struct {
    events []Event
}

func (s *InMemoryStore) Save(event Event) error {
    s.events = append(s.events, event)
    return nil
}

func (s *InMemoryStore) FindByTeam(teamID string) ([]Event, error) {
    var result []Event
for _, e := range s.events {
        if e.TeamID == teamID {
            result = append(result, e)
        }
    }
    return result, nil
}

func (s *InMemoryStore) Count() (int, error) {
    return len(s.events), nil
}

// Now InMemoryStore can be passed wherever EventStore is expected
// Later, you can swap to PostgresStore without changing any calling code
var store EventStore = &InMemoryStore{}
store.Save(event)   // works!
```

4. Error Handling — Go's Most Important Discipline
Go has no exceptions. Functions that can fail return an error as the last return value. You must check it every time. This makes all failure paths explicit and impossible to accidentally ignore.

```
// Go error handling — the most important pattern to master
// Functions return (value, error) — two return values
func readCSVFile(path string) ([]Event, error) {
    f, err := os.Open(path)
    if err != nil {
        // fmt.Errorf wraps the original error with more context
return nil, fmt.Errorf("readCSVFile: cannot open %s: %w", path, err)
    }
    defer f.Close()  // defer: runs when the function returns — guaranteed cleanup
// ... read and parse the file ...
return events, nil // nil error means success
}

// Calling the function — MUST check the error
events, err := readCSVFile("data/events.csv")
if err != nil {
    log.Fatalf("Failed to read events: %v", err)
    // log.Fatalf prints the error and exits the program
}
// Only use events here — after confirming err is nil
fmt.Printf("Loaded %d events\n", len(events))

// Custom error types for more context
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed: field '%s': %s", e.Field, e.Message)
}
```

> **💡 Fresher Tip — What is defer?**

> `defer` is one of Go's most useful keywords. It schedules a function call to run when the surrounding function returns — no matter how it returns (normal return, early return, panic). It's most commonly used to close files, release database connections, and unlock mutexes. Without defer, if your function has 5 return points, you'd have to write `f.Close()` before each one. With defer, you write it once after opening and it always runs. Always use `defer f.Close()` immediately after `os.Open()`.

5. Goroutines & Channels — Go's Concurrency Superpower
A goroutine is a lightweight thread that Go manages — you can run 100,000 of them with less memory than 100 OS threads. Channels are typed pipes through which goroutines communicate safely.

```
// Goroutines — the "go" keyword starts a function in a new goroutine
// Without goroutines — sequential, slow
for _, url := range urls {
    result := checkHealth(url)  // waits for each one to finish before starting next
    fmt.Println(result)
}

// With goroutines — concurrent, fast
results := make(chan string, len(urls))  // buffered channel — holds len(urls) values
for _, url := range urls {
    go func(u string) {         // "go" keyword starts a goroutine
        results <- checkHealth(u) // <- sends result into channel
    }(url)
}

// Collect all results from the channel
for i := 0; i < len(urls); i++ {
    fmt.Println(<-results)  // <-results receives one value from channel
}
// All URLs checked simultaneously — 10x faster than sequential
// WaitGroup — coordinate: "wait until all goroutines finish"
var wg sync.WaitGroup
for _, job := range jobs {
    wg.Add(1)           // tell WaitGroup: one more goroutine starting
go func(j Job) {
        defer wg.Done()   // tell WaitGroup: this goroutine is done
processJob(j)
    }(job)
}
wg.Wait()  // block here until all goroutines call Done()
```

### 2. Program 1 — REST API Server

**Business Problem:** SwiftMetrics needs an HTTP endpoint that receives developer event data from client SDKs installed on companies' CI/CD tools. The API must accept JSON, validate the incoming data, store it, and return a response — all in under 5 milliseconds. It must handle 10,000 requests per second without crashing.

**Scene 2 — SwiftMetrics Architecture Meeting | The API Design**

> **Ananya** _Senior Go Engineer_
> 
> Rahul, your first task: build the events ingestion API. POST /api/v1/events — accepts JSON with the event data. GET /api/v1/events/{teamId} — returns all events for a team. Add a request logging middleware so every request prints its method, path, and duration. No external frameworks — use Go's standard net/http library. This forces you to understand what frameworks actually do under the hood.

> **Vikas** _Go Architect_
> 
> And always implement graceful shutdown. When we deploy a new version, Kubernetes sends SIGTERM to the process. A properly written Go server catches SIGTERM, stops accepting new connections, finishes in-flight requests within 30 seconds, and exits cleanly. If you don't handle this, you get data corruption and 502 errors during every deployment.

```
Program 1 — REST API Architecture
=====================================

  Client SDK                HTTP Server (net/http)         In-Memory Store
  ──────────                ─────────────────────         ───────────────
  POST /api/v1/events ────► RequestLogger middleware
    {JSON body}                   │
                              Handler: IngestEvent
                                  │
                             ① Decode JSON body
                             ② Validate fields
                             ③ Save to store
                             ④ Return 201 JSON
                                  │
  ◄─── 201 {"id":"evt-001"} ──────┘

  GET /api/v1/events/{teamId} ──► Handler: GetTeamEvents
                                       │
                                  ① Parse teamId from URL
                                  ② Query store
                                  ③ Return 200 JSON array
  ◄─── 200 [{...},{...}] ─────────────┘

  OS SIGTERM ──────────────────────► Graceful Shutdown
                                       │
                                  ① Stop accepting new requests
                                  ② Wait up to 30s for active requests
                                  ③ Close server
                                  ④ Exit 0
```

```
// cmd/api/main.go
// Program 1: REST API Server for SwiftMetrics event ingestion
// Runs on port 8080, handles JSON events, graceful shutdown
package main
import (
    "context"
"encoding/json"
"fmt"
"log"
"net/http"
"os"
"os/signal"
"strings"
"sync"
"syscall"
"time"
)

// ── DATA STRUCTURES ──────────────────────────────────────────────────────────
type Event struct {
    ID        string `json:"id"` // json tags control JSON field names
    TeamID    string `json:"team_id"`
    Type      string `json:"type"`
    UserEmail string `json:"user_email"`
    Duration  int64 `json:"duration_ms"`
    Success   bool `json:"success"`
    CreatedAt string `json:"created_at"`
}

type APIResponse struct {
    Success bool `json:"success"`
    Message string `json:"message,omitempty"` // omitempty: skip if empty
    Data    interface{} `json:"data,omitempty"` // any type
}

// ── IN-MEMORY STORE (thread-safe) ────────────────────────────────────────────
type EventStore struct {
    mu     sync.RWMutex // protects events from concurrent read/write race conditions
    events []Event
}

func (s *EventStore) Add(e Event) {
    s.mu.Lock()         // exclusive write lock — no concurrent reads or writes
defer s.mu.Unlock() // always unlock when done
    s.events = append(s.events, e)
}

func (s *EventStore) GetByTeam(teamID string) []Event {
    s.mu.RLock()         // shared read lock — multiple readers allowed simultaneously
defer s.mu.RUnlock()
    var result []Event
for _, e := range s.events {
        if e.TeamID == teamID {
            result = append(result, e)
        }
    }
    return result
}

// ── HANDLERS ─────────────────────────────────────────────────────────────────
type Server struct {
    store *EventStore
}

// writeJSON is a helper — sends a JSON response with the given status code
func writeJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

// IngestEvent handles POST /api/v1/events
func (s *Server) IngestEvent(w http.ResponseWriter, r *http.Request) {
    // Step 1: Only accept POST method
if r.Method != http.MethodPost {
        writeJSON(w, http.StatusMethodNotAllowed, APIResponse{
            Success: false, Message: "method not allowed",
        })
        return
    }

    // Step 2: Decode the JSON body into an Event struct
var event Event
if err := json.NewDecoder(r.Body).Decode(&event); err != nil {
        writeJSON(w, http.StatusBadRequest, APIResponse{
            Success: false, Message: "invalid JSON: " + err.Error(),
        })
        return
    }

    // Step 3: Validate required fields
if event.TeamID == "" || event.Type == "" || event.UserEmail == "" {
        writeJSON(w, http.StatusBadRequest, APIResponse{
            Success: false, Message: "team_id, type, and user_email are required",
        })
        return
    }

    // Step 4: Set server-generated fields
    event.ID = fmt.Sprintf("evt-%d", time.Now().UnixNano())
    event.CreatedAt = time.Now().Format(time.RFC3339)

    // Step 5: Save and respond
    s.store.Add(event)
    writeJSON(w, http.StatusCreated, APIResponse{
        Success: true, Message: "event recorded", Data: event,
    })
}

// GetTeamEvents handles GET /api/v1/events/{teamId}
func (s *Server) GetTeamEvents(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        writeJSON(w, http.StatusMethodNotAllowed, APIResponse{Success: false})
        return
    }

    // Extract teamId from URL path: "/api/v1/events/team-alpha" → "team-alpha"
    parts := strings.Split(r.URL.Path, "/")
    teamID := parts[len(parts)-1]
    if teamID == "" {
        writeJSON(w, http.StatusBadRequest, APIResponse{
            Success: false, Message: "teamId is required in URL",
        })
        return
    }

    events := s.store.GetByTeam(teamID)
    writeJSON(w, http.StatusOK, APIResponse{
        Success: true,
        Data:    events,
    })
}

// ── MIDDLEWARE ────────────────────────────────────────────────────────────────
// RequestLogger wraps a handler and logs every request's method, path, and duration
func RequestLogger(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next(w, r)  // call the actual handler
        log.Printf("[%s] %s %s — %v", r.Method, r.URL.Path, r.RemoteAddr, time.Since(start))
    }
}

// ── MAIN — SERVER SETUP AND GRACEFUL SHUTDOWN ─────────────────────────────────
func main() {
    store := &EventStore{}
    srv := &Server{store: store}

    mux := http.NewServeMux()
    mux.HandleFunc("/api/v1/events",         RequestLogger(srv.IngestEvent))
    mux.HandleFunc("/api/v1/events/",        RequestLogger(srv.GetTeamEvents))
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
    })

    httpServer := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
    }

    // Start server in a goroutine so we can listen for OS signals below
go func() {
        log.Printf("SwiftMetrics API starting on :8080")
        if err := httpServer.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // ── GRACEFUL SHUTDOWN ────────────────────────────────────────────────────
    // signal.NotifyContext creates a context that is cancelled when SIGINT/SIGTERM arrives
    // This is how Kubernetes tells our process to stop
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    <-ctx.Done()  // block here until signal received

    log.Println("Shutdown signal received — draining connections...")
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := httpServer.Shutdown(shutdownCtx); err != nil {
        log.Printf("Shutdown error: %v", err)
    }
    log.Println("Server stopped cleanly.")
    os.Exit(0)
}
```

```
SwiftMetrics API starting on :8080
[POST] /api/v1/events 127.0.0.1:54231 — 142µs
[GET]  /api/v1/events/team-alpha 127.0.0.1:54232 — 83µs
[GET]  /health 127.0.0.1:54233 — 12µs

# Test with curl:
# curl -X POST http://localhost:8080/api/v1/events \
#   -H "Content-Type: application/json" \
#   -d '{"team_id":"team-alpha","type":"build","user_email":"rahul@co.com","duration_ms":2400,"success":true}'
# Response: {"success":true,"message":"event recorded","data":{...}}
```

### 3. Program 2 — CSV Event Processor

**Business Problem:** At the end of each day, SwiftMetrics receives CSV exports from enterprise clients who can't use the real-time SDK. These files have up to 2 million rows. The processor must parse every row, validate fields, skip malformed rows (logging the skip reason), aggregate metrics per team, and write a summary report — all in under 30 seconds.

**Scene 3 — SwiftMetrics Data Team | The Nightly Import**

> **Divya** _Data Engineering Lead — SwiftMetrics_
> 
> Every night at midnight, three enterprise clients send us CSV files. TechGiant's file is 1.8 million rows. The Python script we had before took 14 minutes to process it. Kubernetes killed it after 10 minutes timeout. We lost a night of data twice this month. Go should handle this in under 30 seconds. Rahul, this is your second program — read the CSV, validate rows, compute per-team metrics, write a JSON summary.

> **Ananya** _Senior Go Engineer_
> 
> Key point: never load the entire CSV into memory. Use encoding/csv's Reader to stream one row at a time. A 1.8 million row file at ~200 bytes per row is 360MB — loading it all at once crashes the container. Stream it row by row, process each one, and accumulate only the summary. That's the Go way — process data as a stream, not a batch loaded into RAM.

```
// cmd/processor/main.go
// Program 2: CSV Event Processor — streams 2M rows, aggregates per-team metrics
package main
import (
    "encoding/csv"
"encoding/json"
"fmt"
"io"
"log"
"os"
"strconv"
"time"
)

type EventRow struct {
    TeamID    string
    EventType string
    Duration  int64
    Success   bool
}

type TeamMetrics struct {
    TeamID       string `json:"team_id"`
    TotalEvents  int `json:"total_events"`
    Successes    int `json:"successes"`
    Failures     int `json:"failures"`
    TotalDuration int64 `json:"total_duration_ms"`
    AvgDuration  float64 `json:"avg_duration_ms"`
    SuccessRate  float64 `json:"success_rate_pct"`
}

type ProcessResult struct {
    ProcessedAt    string `json:"processed_at"`
    TotalRows      int `json:"total_rows"`
    ValidRows      int `json:"valid_rows"`
    SkippedRows    int `json:"skipped_rows"`
    ProcessingTime string `json:"processing_time"`
    TeamMetrics    []TeamMetrics `json:"team_metrics"`
}

// parseRow validates and converts a CSV row ([]string) into an EventRow struct
func parseRow(record []string, lineNum int) (*EventRow, error) {
    // CSV columns: team_id, event_type, duration_ms, success
if len(record) < 4 {
        return nil, fmt.Errorf("line %d: expected 4 columns, got %d", lineNum, len(record))
    }

    teamID := strings.TrimSpace(record[0])
    eventType := strings.TrimSpace(record[1])
    if teamID == "" || eventType == "" {
        return nil, fmt.Errorf("line %d: team_id and event_type cannot be empty", lineNum)
    }

    // strconv.ParseInt converts "2400" (string) to 2400 (int64)
    // 10 = base 10 (decimal), 64 = 64-bit integer
    duration, err := strconv.ParseInt(strings.TrimSpace(record[2]), 10, 64)
    if err != nil {
        return nil, fmt.Errorf("line %d: invalid duration '%s'", lineNum, record[2])
    }

    // strconv.ParseBool converts "true"/"false" (string) to bool
    success, err := strconv.ParseBool(strings.TrimSpace(record[3]))
    if err != nil {
        return nil, fmt.Errorf("line %d: invalid success value '%s'", lineNum, record[3])
    }

    return &EventRow{
        TeamID: teamID, EventType: eventType,
        Duration: duration, Success: success,
    }, nil
}

func processCSV(filePath string) (*ProcessResult, error) {
    start := time.Now()

    f, err := os.Open(filePath)
    if err != nil {
        return nil, fmt.Errorf("cannot open file: %w", err)
    }
    defer f.Close()

    reader := csv.NewReader(f)  // creates a streaming CSV reader
// Map from teamID → accumulating metrics
    // make(map[K]V) creates an empty map (like dict in Python)
    teamData := make(map[string]*TeamMetrics)

    totalRows, validRows, skippedRows := 0, 0, 0
    lineNum := 0

    for {
        record, err := reader.Read()  // reads ONE row at a time — memory efficient!
if err == io.EOF {
            break // io.EOF means we've read the entire file — normal, not an error
        }
        if err != nil {
            log.Printf("CSV read error at line %d: %v", lineNum, err)
            skippedRows++
            continue
        }

        lineNum++
        totalRows++

        if lineNum == 1 {
            continue // skip header row
        }

        row, err := parseRow(record, lineNum)
        if err != nil {
            log.Printf("SKIP: %v", err)
            skippedRows++
            continue
        }

        // Get or create TeamMetrics entry for this team
        metrics, exists := teamData[row.TeamID]
        if !exists {
            metrics = &TeamMetrics{TeamID: row.TeamID}
            teamData[row.TeamID] = metrics
        }

        metrics.TotalEvents++
        metrics.TotalDuration += row.Duration
        if row.Success {
            metrics.Successes++
        } else {
            metrics.Failures++
        }
        validRows++
    }

    // Compute derived fields (averages, rates) after accumulation is done
    teamList := make([]TeamMetrics, 0, len(teamData))
    for _, m := range teamData {
        if m.TotalEvents > 0 {
            m.AvgDuration = float64(m.TotalDuration) / float64(m.TotalEvents)
            m.SuccessRate = float64(m.Successes) / float64(m.TotalEvents) * 100
        }
        teamList = append(teamList, *m)
    }

    return &ProcessResult{
        ProcessedAt:    time.Now().Format(time.RFC3339),
        TotalRows:      totalRows,
        ValidRows:      validRows,
        SkippedRows:    skippedRows,
        ProcessingTime: time.Since(start).String(),
        TeamMetrics:    teamList,
    }, nil
}

func main() {
    filePath := "data/events.csv"
if len(os.Args) > 1 {
        filePath = os.Args[1]  // os.Args[0] is the program name, [1] is first argument
    }

    log.Printf("Processing: %s", filePath)
    result, err := processCSV(filePath)
    if err != nil {
        log.Fatalf("Processing failed: %v", err)
    }

    // Write result to stdout as pretty-printed JSON
    encoder := json.NewEncoder(os.Stdout)
    encoder.SetIndent("", "  ")  // 2-space indentation for readability
    encoder.Encode(result)

    log.Printf("Done: %d valid rows, %d skipped, took %s",
        result.ValidRows, result.SkippedRows, result.ProcessingTime)
}
```

```
{
  "processed_at": "2026-01-15T09:00:01Z",
  "total_rows": 1800000,
  "valid_rows": 1798234,
  "skipped_rows": 1766,
  "processing_time": "4.823s",
  "team_metrics": [
    {
      "team_id": "team-alpha",
      "total_events": 324819,
      "successes": 312441,
      "failures": 12378,
      "total_duration_ms": 779565600,
      "avg_duration_ms": 2400.1,
      "success_rate_pct": 96.19
    }
  ]
}
Done: 1798234 valid rows, 1766 skipped, took 4.823s
```

### 4. Program 3 — Concurrent Worker Pool

**Business Problem:** SwiftMetrics needs to check the health of 500 customer API endpoints every minute to detect outages and include their availability in the analytics dashboard. Checking them sequentially takes over 8 minutes (500 × 1s timeout). Using goroutines for every request simultaneously would open 500 connections at once and overwhelm the network. The solution: a worker pool — a fixed number of goroutines processing jobs from a shared queue.

**Scene 4 — Engineering Standup | The Concurrency Problem**

> **Ananya** _Senior Go Engineer_
> 
> This is where Go really shines over Python. Python's threading is blocked by the GIL for CPU-bound work. Go goroutines are truly parallel. But more importantly — a worker pool is the right pattern. You don't want 500 goroutines firing simultaneously. You want 50 workers pulling from a jobs channel. The channel acts as a queue. Workers consume jobs as fast as they can. This gives you parallelism AND backpressure. Rahul, implement this with goroutines, a jobs channel, and a results channel.

```
Program 3 — Worker Pool Pattern
==================================

  main goroutine                Jobs Channel           Results Channel
  ──────────────                ────────────           ───────────────
  Loads 500 jobs ──────────►  [job][job][job][...]  
                                     │
                         ┌───────────┼───────────┐
                         ▼           ▼           ▼
                    Worker 1    Worker 2    Worker 3  ... Worker 50
                    goroutine   goroutine   goroutine
                         │           │           │
                    checks URL  checks URL  checks URL
                         │           │           │
                         └───────────┼───────────┘
                                     ▼
                               results channel ──► main collects all 500
                               [result][result][...]    results and writes report

  Total time: ~10 seconds (vs 8 minutes sequential)
  Goroutines active: 50 workers + 1 main = 51 total
```

```
// cmd/worker/main.go
// Program 3: Concurrent Worker Pool for health-checking 500 API endpoints
package main
import (
    "fmt"
"log"
"net/http"
"sync"
"time"
)

const (
    NumWorkers = 50              // number of goroutines in the pool
    HTTPTimeout = 5 * time.Second // per-request timeout
)

type HealthJob struct {
    CustomerID string
    URL        string
}

type HealthResult struct {
    CustomerID   string
    URL          string
    StatusCode   int
    ResponseTime time.Duration
    IsUp         bool
    Error        string
}

// checkHealth sends an HTTP GET and returns the result.
// This is the work each goroutine does for one job.
func checkHealth(job HealthJob) HealthResult {
    result := HealthResult{CustomerID: job.CustomerID, URL: job.URL}
    start := time.Now()

    client := &http.Client{Timeout: HTTPTimeout}
    resp, err := client.Get(job.URL)
    result.ResponseTime = time.Since(start)

    if err != nil {
        result.Error = err.Error()
        result.IsUp = false
return result
    }
    defer resp.Body.Close()

    result.StatusCode = resp.StatusCode
    result.IsUp = resp.StatusCode >= 200 && resp.StatusCode < 300
    return result
}

// worker is the function each goroutine runs.
// It reads jobs from the jobs channel and sends results to the results channel.
// It exits when the jobs channel is closed.
func worker(id int, jobs <-chan HealthJob, results chan<- HealthResult, wg *sync.WaitGroup) {
    defer wg.Done()  // signal WaitGroup when this worker exits
// range over a channel: reads until the channel is closed
for job := range jobs {
        result := checkHealth(job)
        results <- result  // send result to results channel
        log.Printf("Worker %d: %s — up=%v (%v)", id, job.CustomerID, result.IsUp, result.ResponseTime)
    }
}

func runHealthCheck(jobs []HealthJob) []HealthResult {
    // Buffered channel for jobs — holds all jobs upfront
    // Workers pull from it at their own pace
    jobCh := make(chan HealthJob, len(jobs))

    // Buffered results channel — holds up to len(jobs) results
    resultCh := make(chan HealthResult, len(jobs))

    var wg sync.WaitGroup
// Step 1: Start the worker pool — NumWorkers goroutines
for i := 1; i <= NumWorkers; i++ {
        wg.Add(1)
        go worker(i, jobCh, resultCh, &wg)
    }

    // Step 2: Feed all jobs into the channel
for _, job := range jobs {
        jobCh <- job
    }
    close(jobCh)  // close the jobs channel — this signals workers to stop after draining
// Step 3: Wait for all workers to finish, then close results channel
go func() {
        wg.Wait()
        close(resultCh)
    }()

    // Step 4: Collect all results from the results channel
var results []HealthResult
for r := range resultCh {  // reads until resultCh is closed
        results = append(results, r)
    }
    return results
}

func main() {
    // Build 500 sample jobs (in production, load from database)
    jobs := make([]HealthJob, 0, 500)
    for i := 1; i <= 500; i++ {
        jobs = append(jobs, HealthJob{
            CustomerID: fmt.Sprintf("customer-%03d", i),
            URL:        fmt.Sprintf("https://api.customer%d.com/health", i),
        })
    }

    start := time.Now()
    log.Printf("Starting health check for %d endpoints with %d workers", len(jobs), NumWorkers)

    results := runHealthCheck(jobs)

    // Summarise results
    up, down := 0, 0
    for _, r := range results {
        if r.IsUp { up++ } else { down++ }
    }
    log.Printf("Done in %v — UP: %d, DOWN: %d", time.Since(start), up, down)
}
```

> **💡 Fresher Tip — Buffered vs Unbuffered Channels**

> An **unbuffered channel** (`make(chan int)`) blocks the sender until a receiver is ready. An **buffered channel** (`make(chan int, 100)`) lets the sender put up to 100 values in without waiting. In our worker pool, we use a buffered jobs channel (`make(chan HealthJob, len(jobs))`) so we can load ALL jobs into the channel instantly — workers then pull from it at their own pace. The results channel is also buffered so workers never have to wait for the collector. Wrong channel types cause **deadlocks** — a program that freezes because every goroutine is waiting for another. Go's runtime detects deadlocks and panics: "all goroutines are asleep — deadlock!"

### 5. Program 4 — CLI Analytics Tool

**Business Problem:** SwiftMetrics engineers need a command-line tool to query the metrics API from their terminal — like how DevOps engineers use `kubectl get pods` or `terraform show`. The tool must support flags for team ID, output format (table or JSON), and date range. It should call the REST API and display the results neatly.

**Scene 5 — Engineering Slack | "I Need a CLI"**

> **Nithya** _DevOps Engineer — SwiftMetrics Platform_
> 
> I'm tired of writing curl commands to check team metrics. Can someone build a proper CLI tool? I want to run something like: swift metrics --team team-alpha --format table and get a nicely formatted output. And swift events --team team-alpha --since 2026-01-01 for event history. Something I can pipe into grep and awk.

> **Ananya** _Senior Go Engineer_
> 
> Rahul — Program 4. Use Go's flag package for argument parsing. The flag package is built-in, simple, and produces proper --help output automatically. Call our REST API with net/http, decode the JSON response, and print as either a formatted table (using text/tabwriter) or as JSON. One binary, zero dependencies, works everywhere.

```
// cmd/cli/main.go
// Program 4: CLI Analytics Tool — query SwiftMetrics API from the terminal
// Usage: swift --team team-alpha --format table
//        swift --team team-alpha --format json
package main
import (
    "encoding/json"
"flag"
"fmt"
"io"
"log"
"net/http"
"os"
"text/tabwriter"
"time"
)

type Config struct {
    APIBase string
    Timeout time.Duration
}

type Event struct {
    ID        string `json:"id"`
    TeamID    string `json:"team_id"`
    Type      string `json:"type"`
    UserEmail string `json:"user_email"`
    Duration  int64 `json:"duration_ms"`
    Success   bool `json:"success"`
    CreatedAt string `json:"created_at"`
}

type APIResponse struct {
    Success bool `json:"success"`
    Data    json.RawMessage `json:"data"` // RawMessage defers JSON parsing
}

func fetchTeamEvents(cfg Config, teamID string) ([]Event, error) {
    url := fmt.Sprintf("%s/api/v1/events/%s", cfg.APIBase, teamID)

    client := &http.Client{Timeout: cfg.Timeout}
    resp, err := client.Get(url)
    if err != nil {
        return nil, fmt.Errorf("API request failed: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        body, _ := io.ReadAll(resp.Body)
        return nil, fmt.Errorf("API error %d: %s", resp.StatusCode, body)
    }

    var apiResp APIResponse
if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return nil, fmt.Errorf("decode failed: %w", err)
    }

    var events []Event
if err := json.Unmarshal(apiResp.Data, &events); err != nil {
        return nil, fmt.Errorf("unmarshal events: %w", err)
    }
    return events, nil
}

// printTable uses text/tabwriter to print aligned columns — like the output of kubectl
func printTable(events []Event) {
    // tabwriter aligns tab-separated columns automatically
    // minwidth=0, tabwidth=0, padding=3, flags=0
    w := tabwriter.NewWriter(os.Stdout, 0, 0, 3, ' ', 0)
    // \t is the tab character — tabwriter uses it to align columns
    fmt.Fprintln(w, "ID\tTEAM\tTYPE\tUSER\tDURATION\tSTATUS\tCREATED")
    fmt.Fprintln(w, "──\t────\t────\t────\t────────\t──────\t───────")
    for _, e := range events {
        status := "✓ OK"
if !e.Success { status = "✗ FAIL" }
        fmt.Fprintf(w, "%s\t%s\t%s\t%s\t%dms\t%s\t%s\n",
            e.ID, e.TeamID, e.Type, e.UserEmail, e.Duration, status, e.CreatedAt)
    }
    w.Flush()  // must call Flush() to actually write the aligned output
}

func main() {
    // flag.String defines a --team flag, default "", description shown in --help
    teamFlag   := flag.String("team",   "",      "team ID to query (required)")
    formatFlag := flag.String("format", "table", "output format: table or json")
    apiFlag    := flag.String("api",    "http://localhost:8080", "API base URL")

    flag.Parse()  // must call Parse() to process the actual command-line arguments
if *teamFlag == "" {
        fmt.Fprintln(os.Stderr, "Error: --team is required")
        flag.Usage()  // prints the auto-generated help text
        os.Exit(1)
    }

    cfg := Config{APIBase: *apiFlag, Timeout: 10 * time.Second}
    events, err := fetchTeamEvents(cfg, *teamFlag)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("Events for team '%s': %d results\n\n", *teamFlag, len(events))

    switch *formatFlag {
    case "table":
        printTable(events)
    case "json":
        enc := json.NewEncoder(os.Stdout)
        enc.SetIndent("", "  ")
        enc.Encode(events)
    default:
        fmt.Fprintf(os.Stderr, "Unknown format: %s (use table or json)\n", *formatFlag)
        os.Exit(1)
    }
}
```

```
$ swift --team team-alpha --format table
Events for team 'team-alpha': 3 results

ID             TEAM         TYPE    USER                    DURATION   STATUS   CREATED
──             ────         ────    ────                    ────────   ──────   ───────
evt-171234001  team-alpha   build   rahul@swiftmetrics.io   2400ms     ✓ OK     2026-01-15T09:01:00Z
evt-171234002  team-alpha   deploy  ananya@swiftmetrics.io  8100ms     ✓ OK     2026-01-15T09:05:00Z
evt-171234003  team-alpha   review  vikram@swiftmetrics.io  450ms      ✗ FAIL   2026-01-15T09:12:00Z

$ swift --help
Usage of swift:
  -api string
        API base URL (default "http://localhost:8080")
  -format string
        output format: table or json (default "table")
  -team string
        team ID to query (required)
```

### 6. Program 5 — gRPC Metrics Streaming Service

**Business Problem:** SwiftMetrics' enterprise dashboard needs live metrics streamed to it in real time — not HTTP polling every 5 seconds. gRPC (Google Remote Procedure Call) is the industry standard for this. It uses Protocol Buffers (binary format — 6x smaller than JSON) and supports server-streaming so the server can push data to the client continuously over one connection.

**Scene 6 — Client Demo Prep | "We Need Real-Time"**

> **Ananya** _Senior Go Engineer_
> 
> The demo is next week and the CTO of TechGiant asked one question: "Is this real-time?" Polling every 5 seconds is not real-time. gRPC server streaming is. The client opens one connection, and our server pushes a metrics snapshot every second for as long as the connection is open. Rahul, define the proto schema, implement the server, and write a client that prints the streamed metrics. This is the architecture Kubernetes itself uses for its watch API.

> **Vikas** _Go Architect_
> 
> Understand proto files first. A .proto file is the contract between server and client — like an API spec but strongly typed and language-agnostic. protoc compiles it into Go code. You never write the gRPC boilerplate — protoc generates it. You only implement the business logic methods that protoc creates as interfaces.

```
Program 5 — gRPC Streaming Architecture
==========================================

  metrics.proto (contract)
  ─────────────────────────
  service MetricsService {
    rpc StreamMetrics(StreamRequest)
        returns (stream MetricsSnapshot);
  }
         │
         ▼ protoc compiler generates Go code
         │
  ┌──────┴────────────────────────────────────────┐
  │  Server (cmd/grpc/server.go)                   │
  │  Implements MetricsServiceServer interface      │
  │  StreamMetrics():                               │
  │    loop {                                       │
  │      snapshot = collectCurrentMetrics()         │
  │      stream.Send(snapshot)    ← push to client  │
  │      time.Sleep(1 second)                       │
  │    }                                            │
  └──────────────────────────────────────────────── ┘
         │  gRPC over HTTP/2 (binary, compressed)
         ▼
  ┌──────┴────────────────────────────────────────┐
  │  Client (cmd/grpc/client.go)                   │
  │  stream = client.StreamMetrics(request)         │
  │  loop {                                         │
  │    snapshot, err = stream.Recv()  ← receive     │
  │    print(snapshot)                              │
  │  }                                              │
  └──────────────────────────────────────────────── ┘

  Result: Dashboard updates every second, no polling
```

```
// proto/metrics.proto
// Step 1: Define the service contract using Protocol Buffers syntax
// Compile with: protoc --go_out=. --go-grpc_out=. proto/metrics.proto

syntax = "proto3";
package metrics;
option go_package = "github.com/swiftmetrics/platform/proto/metrics";

// Request message — what the client sends to start the stream
message StreamRequest {
    string team_id = 1;        // field number 1 (used in binary encoding)
    int32  interval_seconds = 2;
}

// The snapshot pushed to the client every interval
message MetricsSnapshot {
    string  team_id         = 1;
    int64   timestamp       = 2;
    int64   events_per_min  = 3;
    double  success_rate    = 4;
    double  avg_duration_ms = 5;
    int32   active_users    = 6;
}

// Service definition — one RPC method with server-side streaming
service MetricsService {
    // "stream MetricsSnapshot" = server sends multiple messages (stream)
// (no stream before StreamRequest) = client sends one request only
    rpc StreamMetrics(StreamRequest) returns (stream MetricsSnapshot);
}
```

```
// cmd/grpc/server.go
// Program 5 — gRPC Server: implements MetricsService and streams live metrics
package main
import (
    "log"
"math/rand"
"net"
"time"
"google.golang.org/grpc"
    pb "github.com/swiftmetrics/platform/proto/metrics" // pb = alias for generated code
)

// metricsServer implements pb.MetricsServiceServer (generated by protoc)
type metricsServer struct {
    pb.UnimplementedMetricsServiceServer // embed this for forward compatibility
}

// StreamMetrics is called when a client connects and calls StreamMetrics.
// The stream parameter lets us push messages to the client.
func (s *metricsServer) StreamMetrics(
    req *pb.StreamRequest,
    stream pb.MetricsService_StreamMetricsServer,
) error {

    log.Printf("Client connected: team=%s, interval=%ds", req.TeamId, req.IntervalSeconds)
    interval := time.Duration(req.IntervalSeconds) * time.Second
if interval == 0 {
        interval = time.Second // default to 1 second if not specified
    }

    // stream.Context() is cancelled when client disconnects — we use that to stop
for {
        select {
        case <-stream.Context().Done():
            log.Printf("Client disconnected: team=%s", req.TeamId)
            return nil
default:
            // Collect current metrics for this team (simulated here)
            snapshot := collectMetrics(req.TeamId)

            // stream.Send() pushes the message to the client
if err := stream.Send(snapshot); err != nil {
                return err  // client disconnected mid-stream
            }
            time.Sleep(interval)
        }
    }
}

// collectMetrics simulates fetching live metrics from a database or cache
func collectMetrics(teamID string) *pb.MetricsSnapshot {
    return &pb.MetricsSnapshot{
        TeamId:        teamID,
        Timestamp:     time.Now().Unix(),
        EventsPerMin:  int64(rand.Intn(5000) + 1000),
        SuccessRate:   94.0 + rand.Float64()*5.0,
        AvgDurationMs: 800.0 + rand.Float64()*400.0,
        ActiveUsers:   int32(rand.Intn(200) + 50),
    }
}

func main() {
    // Start TCP listener on port 50051 (gRPC convention)
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("Failed to listen: %v", err)
    }

    grpcServer := grpc.NewServer()
    pb.RegisterMetricsServiceServer(grpcServer, &metricsServer{})

    log.Printf("gRPC MetricsService listening on :50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("Serve failed: %v", err)
    }
}
```

```
// cmd/grpc/client.go
// gRPC Client: connects and receives live metrics stream
package main
import (
    "context"
"fmt"
"io"
"log"
"time"
"google.golang.org/grpc"
"google.golang.org/grpc/credentials/insecure"
    pb "github.com/swiftmetrics/platform/proto/metrics"
)

func main() {
    // Dial the gRPC server — establishes the connection
    conn, err := grpc.Dial("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),  // no TLS for dev
    )
    if err != nil {
        log.Fatalf("Connection failed: %v", err)
    }
    defer conn.Close()

    client := pb.NewMetricsServiceClient(conn)

    // Create a context with timeout — stream auto-closes after 30 seconds
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Call StreamMetrics — returns a stream object
    stream, err := client.StreamMetrics(ctx, &pb.StreamRequest{
        TeamId:          "team-alpha",
        IntervalSeconds: 1,
    })
    if err != nil {
        log.Fatalf("StreamMetrics failed: %v", err)
    }

    fmt.Println("📊 SwiftMetrics Live Dashboard — team-alpha")
    fmt.Println("──────────────────────────────────────────")

    // Receive loop: stream.Recv() blocks until server sends a message
for {
        snapshot, err := stream.Recv()
        if err == io.EOF {
            fmt.Println("Stream ended.")
            return
        }
        if err != nil {
            log.Fatalf("Recv error: %v", err)
        }

        t := time.Unix(snapshot.Timestamp, 0).Format("15:04:05")
        fmt.Printf("[%s] Events/min: %d  |  Success: %.1f%%  |  Avg: %.0fms  |  Users: %d\n",
            t, snapshot.EventsPerMin, snapshot.SuccessRate,
            snapshot.AvgDurationMs, snapshot.ActiveUsers)
    }
}
```

```
📊 SwiftMetrics Live Dashboard — team-alpha
──────────────────────────────────────────
[09:01:00] Events/min: 3412  |  Success: 97.3%  |  Avg: 934ms  |  Users: 142
[09:01:01] Events/min: 2891  |  Success: 96.8%  |  Avg: 1012ms |  Users: 138
[09:01:02] Events/min: 4108  |  Success: 98.1%  |  Avg: 867ms  |  Users: 155
[09:01:03] Events/min: 3756  |  Success: 95.2%  |  Avg: 978ms  |  Users: 147
[09:01:04] Events/min: 2234  |  Success: 97.7%  |  Avg: 823ms  |  Users: 131
^C  (Ctrl+C to stop)
Stream ended.
```

### 7. Essential Go Commands & Tools Reference

These are the Go toolchain commands you will use every day in a professional Go project. Unlike Python or Node, Go ships with almost everything built in — formatter, test runner, benchmark runner, documentation generator.

Command

What It Does

When You Use It

go run ./cmd/api/

Compile and run in one step — no binary saved

Development — fast iteration

go build ./...

Compile all packages, save binaries

Before deploying or testing the binary

go test ./...

Run all tests in all packages

Before every commit, in CI/CD pipeline

go test -v ./...

Run tests with verbose output (shows each test name)

Debugging failing tests

go test -bench=.

Run benchmark functions

Measuring performance of critical code

go test -cover ./...

Show test coverage percentage per package

Measuring how much code is tested

go fmt ./...

Format all .go files to standard style (no arguments)

Before every commit — non-negotiable

go vet ./...

Static analysis — catches common bugs fmt misses

Before every commit, in CI/CD

go mod init

Initialise a new module (creates go.mod)

Starting a new project

go get package

Download and add a dependency to go.mod

Adding a new external library

go mod tidy

Remove unused dependencies, add missing ones

After changing imports, before committing

go mod vendor

Copy all dependencies to vendor/ folder

Projects that need offline builds

go generate ./...

Run //go:generate directives (e.g. protoc, mockgen)

Regenerating proto code, mocks

go doc fmt.Println

Show documentation for any function or package

Looking up how to use a function

GOOS=linux GOARCH=amd64 go build

Cross-compile for a different OS/architecture

Building Linux binary from macOS/Windows

### 8. Debugging & Testing Go Programs

Go has a built-in testing framework — no external libraries needed. The convention is to create `_test.go` files alongside your code. Tests are functions named `TestXxx(t *testing.T)`. This section shows you how to write and debug real Go tests.

```
// internal/models/event_test.go
// Unit tests for the Event models
// Go test files have the _test.go suffix — they are ONLY compiled during testing
package models
import "testing"
// Test functions must start with "Test" and accept *testing.T
func TestTeamSummarySuccessRate(t *testing.T) {
    // Table-driven tests: define multiple cases in a slice, run them all in a loop
    // This is the standard Go testing pattern — used everywhere including standard library
    tests := []struct {
        name     string
        total    int
        success  int
        expected float64
    }{
        {"no events",     0,   0,   0.0},
        {"all success",  100, 100, 100.0},
        {"half success", 100,  50,  50.0},
        {"96.2% rate",   500, 481,  96.2},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            summary := &TeamSummary{
                TotalEvents: tc.total,
                Successes:   tc.success,
            }
            got := summary.SuccessRate()
            if got != tc.expected {
                // t.Errorf marks test as failed but continues running
                t.Errorf("SuccessRate() = %.1f, want %.1f", got, tc.expected)
            }
        })
    }
}

// Test the EventStore thread safety
func TestEventStoreConcurrency(t *testing.T) {
    store := &InMemoryStore{}
    var wg sync.WaitGroup
// Launch 100 goroutines all writing simultaneously — tests for race conditions
for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            store.Save(Event{ID: fmt.Sprintf("evt-%d", n), TeamID: "team-a"})
        }(i)
    }
    wg.Wait()

    count, _ := store.Count()
    if count != 100 {
        t.Errorf("expected 100 events, got %d", count)
    }
}

// Run tests:        go test ./internal/models/
// Run with race detector: go test -race ./internal/models/
// The -race flag detects concurrent map/slice access bugs at runtime
```

**Common Go Mistakes Freshers Make & How to Fix Them**

- **🔴 Ignoring the error return value** — **Wrong:** `f, _ := os.Open("file.csv")`  
**Right:** `f, err := os.Open(...); if err != nil { return err }`  
The _ discards the error silently. If the file doesn't exist, f is nil and the next line panics with a cryptic nil pointer error.

- **🔴 Forgetting defer f.Close()** — **Wrong:** Opening a file and returning without closing.  
**Right:** Write `defer f.Close()` immediately after every successful `os.Open()`. Forgetting this causes file descriptor leaks — the server eventually runs out of open file handles.

- **🔴 Capturing loop variable in goroutine** — **Wrong:** `for _, url := range urls { go func() { checkHealth(url) }() }`  
**Right:** `go func(u string) { checkHealth(u) }(url)`  
Without passing url as a parameter, all goroutines see the last value of url when they finally run.

- **🔴 Concurrent map writes without mutex** — **Wrong:** Writing to a `map` from multiple goroutines without locking.  
**Right:** Use `sync.RWMutex` and lock before every read/write. Run `go test -race` — Go's race detector catches this before production.

- **🔴 Not closing channels causing goroutine leaks** — **Wrong:** Starting goroutines that block forever on a channel that never closes.  
**Right:** Always `close(ch)` after all sends are done. Goroutines ranging over the channel will then exit cleanly. Use `go tool pprof` to detect goroutine leaks.

- **🔴 Using panic instead of returning errors** — **Wrong:** `panic("something went wrong")` in library or handler code.  
**Right:** Return `error` values — let the caller decide what to do. Panic is only appropriate for programmer errors (e.g. nil pointer that should never happen) not for expected failures like file not found or invalid input.

### 9. Interview Questions — Golang

These are the questions you will face in Go developer interviews. The answers reference real concepts from the programs you built in this project.

##### Interview Q&A — Fresher Level (0–1 Year Go Experience)

**Q: Q1. What is a goroutine and how is it different from a thread?**

A: A goroutine is a lightweight function that runs concurrently with other goroutines in the same process. Unlike OS threads (which cost ~1MB of memory each), goroutines start with only ~8KB of stack that grows dynamically as needed — making it practical to run 100,000 goroutines simultaneously. Go's runtime scheduler multiplexes goroutines onto a much smaller number of OS threads using the M:N threading model. You start a goroutine with the `go` keyword: `go checkHealth(url)`. In our worker pool (Program 3), we ran 50 goroutines simultaneously checking 500 endpoints — something that would be impractical with 50 OS threads due to memory and context-switching overhead.

**Q: Q2. What is a channel in Go and when do you use it?**

A: A channel is a typed conduit through which goroutines send and receive values safely — like a thread-safe queue. Channels are created with `make(chan Type, bufferSize)`. An unbuffered channel `make(chan int)` blocks the sender until a receiver is ready. A buffered channel `make(chan int, 100)` lets the sender enqueue up to 100 values without blocking. You use channels when goroutines need to communicate results or coordinate work — as in Program 3 where 50 worker goroutines sent health check results back to the main goroutine via a results channel. The general rule: use channels to communicate between goroutines; use mutexes to protect shared data from concurrent access.

**Q: Q3. How does Go handle errors? What is the difference from exceptions in Java/Python?**

A: Go has no exceptions. Functions that can fail return an `error` as the last return value. The caller must explicitly check `if err != nil` before using the result. This makes all error paths visible in the code — you can never accidentally swallow an error. In Java/Python, a thrown exception propagates up the call stack silently unless caught — you can call five functions and only catch at the top, potentially missing intermediate cleanup. In Go, every function in the chain explicitly handles or propagates its errors. The idiom `return nil, fmt.Errorf("operation failed: %w", err)` wraps the original error with context. The `%w` verb allows the caller to unwrap and inspect the original error type with `errors.As()`.

**Q: Q4. What is an interface in Go and how does it differ from interfaces in Java?**

A: A Go interface is a set of method signatures. Any type that has all those methods automatically satisfies the interface — there is no `implements` keyword. This is called implicit (or structural) satisfaction, also known as duck typing with static type checking. In Java, you must explicitly declare `class Dog implements Animal`. In Go, if `Dog` has a `Speak()` method and `Animal` interface requires `Speak()`, Dog automatically satisfies Animal. This makes Go interfaces very flexible — you can define an interface for a type in an external library without modifying that library. In Program 1, our EventStore interface let us swap InMemoryStore for a PostgresStore without changing any handler code.

**Q: Q5. What is a mutex and when do you use it in Go?**

A: A mutex (mutual exclusion lock) is a synchronisation primitive that ensures only one goroutine can access a shared resource at a time. `sync.Mutex` has Lock() and Unlock() methods. `sync.RWMutex` is more efficient for read-heavy workloads: it allows multiple goroutines to read simultaneously (`RLock()`), but only one to write (`Lock()`). In Program 1's EventStore, we used RWMutex because events are read far more often than written — multiple HTTP handlers can read the event list simultaneously, improving throughput. Always use `defer mu.Unlock()` immediately after `mu.Lock()` — this guarantees the unlock happens even if the function returns early or panics. Never access a map or slice from multiple goroutines without mutex protection — Go's race detector (go test -race) will catch this.

**Q: Q6. What does defer do and why is it important?**

A: `defer` schedules a function call to execute when the surrounding function returns — no matter how it returns (normal, early return, or panic). Deferred calls are executed in LIFO (last-in, first-out) order. The most important use is guaranteed resource cleanup: write `defer f.Close()` immediately after `os.Open()` — the file will always be closed no matter how the function exits. Without defer, you'd need to call Close() before every return statement. In Program 2's CSV processor, `defer f.Close()` ensured the file was always closed after reading, even if parsing failed partway through. Defer also captures variables by reference, which is sometimes a subtle gotcha — the deferred function sees the variable's value at the time it executes, not at the time defer was called.

**Quiz: Quiz 1 — What is wrong with this goroutine code? for _, url := range urls { go func() { fmt.Println(url) }() }**

- A) You cannot use range with goroutines
- B) All goroutines will print the same value — the last value of url — because they share the loop variable by reference
- C) go func() requires a return value
- D) fmt.Println is not safe to use in goroutines

> **Answer/explanation:** ✅ Answer: B. The loop variable `url` is shared across all iterations. By the time the goroutines actually execute, the loop has often finished and `url` holds its last value. All goroutines print the same URL. The fix: pass url as a parameter — `go func(u string) { fmt.Println(u) }(url)`. This creates a copy for each goroutine. Run `go vet ./...` — it detects this exact pattern and warns you.

**Quiz: Quiz 2 — In Program 3's worker pool, why do we close(jobCh) after sending all jobs?**

- A) To free memory used by the channel
- B) Because closed channels cause goroutines to panic if they try to send
- C) To signal workers to stop — workers using "for job := range jobCh" exit when the channel is closed and empty
- D) Go requires all channels to be closed before program exit

> **Answer/explanation:** ✅ Answer: C. When you use `for job := range ch` inside a goroutine, the goroutine blocks waiting for the next value. When the channel is closed AND empty, `range` terminates — just like ranging over a slice that has been fully iterated. If you never close the channel, workers block forever waiting for more jobs — this is a goroutine leak. Closing channels is the idiomatic Go way to signal "no more work." Important: only the sender should close a channel; receiving from a closed channel is safe (returns zero values) but sending to a closed channel panics.

**Quiz: Quiz 3 — What does sync.RWMutex provide that sync.Mutex does not?**

- A) RWMutex is faster than Mutex for all operations
- B) RWMutex allows multiple goroutines to read simultaneously but only one to write — improving throughput for read-heavy workloads
- C) RWMutex prevents deadlocks automatically
- D) RWMutex works across multiple processes

> **Answer/explanation:** ✅ Answer: B. A regular Mutex is exclusive: when any goroutine holds the lock (whether reading or writing), all others wait. RWMutex distinguishes between reads and writes: multiple goroutines can call RLock() simultaneously and read concurrently. When a goroutine calls Lock() (for writing), all readers finish and exit, then the writer gets exclusive access. For our EventStore in Program 1 — which handles thousands of GET requests per second vs occasional POST requests — RWMutex dramatically improves read throughput since most requests don't contend with each other. Use Mutex when the ratio of reads to writes is roughly equal; use RWMutex when reads dominate.

> **Golang Project — Core Takeaways for Freshers**

> - Go is intentionally simple — no classes, no inheritance, no generics complexity (until Go 1.18+). Master structs, interfaces, goroutines, channels, and error handling, and you can build any production system.
> - Errors are values, not exceptions — always check if err != nil immediately after every function call that can fail. Never use _ to discard an error in production code.
> - defer is your cleanup contract — write defer f.Close() immediately after os.Open(), defer mu.Unlock() immediately after mu.Lock(), defer cancel() immediately after context.WithTimeout(). Never let cleanup be conditional.
> - Goroutines are cheap but not free — a goroutine leak is a real production problem. Always ensure goroutines have a clear exit condition: a closed channel, a cancelled context, or a timeout. Run go test -race on every PR.
> - The worker pool pattern (jobs channel → fixed goroutines → results channel) is the most important concurrency pattern in Go — it gives you parallelism with controlled resource usage. Learn it deeply.
> - Always use sync.RWMutex to protect shared maps and slices from concurrent access — never access them from multiple goroutines without locking. The race detector (go test -race) catches this before production.
> - gRPC + Protocol Buffers is the industry standard for internal microservice communication — smaller payloads than JSON, strongly typed contracts, and streaming built in. Learn proto syntax early — it appears in every Go backend role.
> - Go tooling is non-negotiable: go fmt (format), go vet (lint), go test -race (concurrency bugs), go mod tidy (clean dependencies). Run all four in your CI/CD pipeline. These are not optional — they are the Go way.

##### Go Code Standards — SwiftMetrics Engineering Rules

- Package names are short, lowercase, single words: models, api, worker — never models_package, API, or Worker
- Exported names (used outside the package) start with uppercase: EventStore, IngestEvent. Unexported (package-private) names start with lowercase: writeJSON, parseRow
- Error messages are lowercase without punctuation: "failed to open file" not "Failed to open file." — they get concatenated into longer chains: "processCSV: readCSVFile: failed to open file: ..."
- Accept interfaces, return structs — function parameters should be interface types (flexible), return values should be concrete struct types (explicit). This is the Go API design philosophy.
- Keep functions short — if a function needs a comment block to explain what it does, it probably needs to be split into smaller named functions. Go encourages many small, well-named functions over long single functions.
- Use context.Context as the first parameter of any function that does I/O (HTTP, database, file) — it enables timeouts, cancellation, and deadline propagation throughout the call chain
- Never use init() functions for complex setup — use explicit constructors like NewServer() or NewEventStore(). init() runs automatically before main() and makes code hard to test and reason about.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Extend Program 1:** Add rate limiting to the IngestEvent handler — allow maximum 100 requests per second per team. Use a `sync.Map` to store per-team `rate.Limiter` from the `golang.org/x/time/rate` package. Return HTTP 429 Too Many Requests when the limit is exceeded. This is a real production requirement for any public API.
2. **Extend Program 2:** Add parallel CSV processing — split the file into chunks by seeking to specific byte offsets and process each chunk in a goroutine. Merge the per-goroutine team metric maps at the end. Benchmark the improvement with `go test -bench=.` comparing sequential vs parallel processing on a 500,000-row file.
3. **Extend Program 3:** Add circuit breaker logic — if a customer's endpoint fails 3 checks in a row, mark it as "circuit open" and skip the next 5 checks before trying again. Store the state in a struct with a mutex. This prevents hammering a dead server with retries — a real production pattern used in Netflix's Hystrix and Go-Resilience.
4. **Extend Program 4:** Add a `swift push` subcommand that reads events from a local JSON file and POSTs each one to the API. Use `os.Args[1]` to distinguish between `swift metrics` and `swift push` commands — this is how CLI tools like `git commit`, `git push` work with subcommands.
5. **Bonus Challenge:** Add a health check endpoint to the gRPC server using gRPC's standard health protocol (`google.golang.org/grpc/health`) so Kubernetes can check if the gRPC server is ready to receive connections. Write a client that uses `grpc_health_v1.NewHealthClient` to check the server status before connecting to the metrics stream.

### Golang Project Complete 🎉

You have designed and built five production-quality Go programs — a REST API server, a streaming CSV processor, a concurrent worker pool, a CLI analytics tool, and a gRPC streaming service — the same types of systems that power SwiftMetrics, and that companies like Uber, Dropbox, Cloudflare, and Docker build every day in Go.

> **Ananya**
> 
> "Rahul, when you arrived you said you'd used Python and Java but never Go. In two weeks you've built a REST API with graceful shutdown, processed 1.8 million CSV rows in under 5 seconds, run 50 concurrent goroutines to health-check 500 endpoints, built a CLI tool, and implemented a gRPC streaming service. That is not junior work — that is what a mid-level Go engineer delivers. The Go mindset clicked for you: errors are values, goroutines are cheap, defer is your cleanup, and keep it simple."

> **Vikas**
> 
> "The worker pool you built in Program 3 is already in production — monitoring 500 customer APIs every 60 seconds. Response time dropped from 8 minutes to 11 seconds. That performance difference directly affects how quickly our dashboard detects outages. When a Go program is right, it is noticeably, measurably right. You wrote a right program."

> **Divya**
> 
> "And the CSV processor? 1.8 million rows in 4.8 seconds. The Python equivalent took 14 minutes. The enterprise client called us this morning and asked why their nightly import now completes before they finish their first coffee. Your Go program is the answer."

> **Next: Advanced Go — Microservices, Observability & Production Patterns**

> - Database integration — connect to PostgreSQL with pgx/v5, use database/sql transactions, implement repository pattern
> - Middleware chains — build authentication, rate limiting, and request tracing middleware using context propagation
> - Prometheus metrics — instrument Go services with counters, histograms, and gauges using the official Go client
> - Structured logging — use slog (Go 1.21 standard library) for JSON-structured logs compatible with Grafana Loki
> - Context propagation — pass request IDs, user info, and deadlines through the entire call chain using context.WithValue
> - Go generics — write type-safe data structures and algorithms using type parameters (Go 1.18+)
> - Kafka consumer in Go — process event streams with confluent-kafka-go, handle offset commits and consumer group rebalancing
