# Go Best Practices

Comprehensive guide for idiomatic Go development. Reference when writing, reviewing, or optimizing Go code.

## When to Apply

- Writing new Go code or reviewing existing code
- Designing package structure and interfaces
- Implementing concurrent code
- Handling errors and debugging
- Optimizing performance
- Writing tests and benchmarks

## Core Philosophy

Go favors simplicity, clarity, and explicit behavior over cleverness.

### Go Proverbs (Rob Pike)
- Don't communicate by sharing memory; share memory by communicating
- Concurrency is not parallelism
- Channels orchestrate; mutexes serialize
- The bigger the interface, the weaker the abstraction
- Make the zero value useful
- `interface{}` says nothing
- Gofmt's style is no one's favorite, yet gofmt is everyone's favorite
- A little copying is better than a little dependency
- Clear is better than clever
- Errors are values
- Don't just check errors, handle them gracefully
- Don't panic

## Package Design

### Naming Conventions
```go
// Package names: short, lowercase, no underscores
package httputil  // Good
package http_util // Bad

// Exported names: CamelCase, descriptive
func ReadAll(r io.Reader) ([]byte, error)  // Good
func Read_all(r io.Reader) ([]byte, error) // Bad

// Avoid stuttering
httputil.Client      // Good
httputil.HTTPClient  // Bad (stutters with package name)
```

### Interface Design
```go
// Small, focused interfaces (1-3 methods)
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Compose interfaces when needed
type ReadWriter interface {
    Reader
    Writer
}

// Accept interfaces, return structs
func Process(r io.Reader) (*Result, error) {
    // Implementation
}
```

### Dependency Injection
```go
// Use interfaces for dependencies (testable)
type Service struct {
    db     Database
    logger Logger
}

func NewService(db Database, logger Logger) *Service {
    return &Service{db: db, logger: logger}
}

// Use functional options for configuration
type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080, timeout: 30 * time.Second} // defaults
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

## Error Handling

### Error Wrapping (Go 1.13+)
```go
// Wrap errors with context
if err != nil {
    return fmt.Errorf("failed to process user %d: %w", userID, err)
}

// Check wrapped errors
if errors.Is(err, sql.ErrNoRows) {
    return nil, ErrNotFound
}

// Extract error types
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    log.Printf("path: %s", pathErr.Path)
}
```

### Custom Error Types
```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

// Sentinel errors for known conditions
var (
    ErrNotFound     = errors.New("resource not found")
    ErrUnauthorized = errors.New("unauthorized")
)
```

### Error Handling Patterns
```go
// Handle errors at the appropriate level
func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err // Don't add context here, caller knows the path
    }
    defer f.Close()
    
    data, err := io.ReadAll(f)
    if err != nil {
        return fmt.Errorf("reading file: %w", err)
    }
    
    return process(data)
}

// Only panic for programming errors
func MustParse(s string) *Config {
    c, err := Parse(s)
    if err != nil {
        panic(fmt.Sprintf("invalid config: %v", err))
    }
    return c
}
```

## Concurrency

### Goroutine Management
```go
// Always use context for cancellation
func worker(ctx context.Context, jobs <-chan Job) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case job, ok := <-jobs:
            if !ok {
                return nil
            }
            if err := process(job); err != nil {
                return err
            }
        }
    }
}

// Use errgroup for coordinated goroutines
func processAll(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    
    for _, item := range items {
        item := item // capture loop variable
        g.Go(func() error {
            return process(ctx, item)
        })
    }
    
    return g.Wait()
}
```

### Channel Patterns
```go
// Fan-out: distribute work to multiple workers
func fanOut(jobs <-chan Job, numWorkers int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                process(job)
            }
        }()
    }
    wg.Wait()
}

// Fan-in: merge multiple channels into one
func fanIn(channels ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup
    
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan Result) {
            defer wg.Done()
            for r := range c {
                out <- r
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

// Worker pool with bounded concurrency
func workerPool(ctx context.Context, jobs <-chan Job, results chan<- Result, maxWorkers int) {
    sem := make(chan struct{}, maxWorkers)
    var wg sync.WaitGroup
    
    for job := range jobs {
        sem <- struct{}{} // acquire
        wg.Add(1)
        go func(j Job) {
            defer func() {
                <-sem // release
                wg.Done()
            }()
            results <- process(j)
        }(job)
    }
    
    wg.Wait()
    close(results)
}
```

### Synchronization
```go
// Use sync.Mutex for state protection
type Counter struct {
    mu    sync.RWMutex
    value int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.value
}

// Use sync.Once for one-time initialization
var (
    instance *DB
    once     sync.Once
)

func GetDB() *DB {
    once.Do(func() {
        instance = newDB()
    })
    return instance
}

// Use sync.Pool for object reuse
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data []byte) {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    // use buf
}
```

## Testing

### Table-Driven Tests
```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 1, 2, 3},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d", tt.a, tt.b, got, tt.expected)
            }
        })
    }
}
```

### Testing with Interfaces
```go
// Define interface for dependencies
type UserStore interface {
    GetUser(id int) (*User, error)
}

// Mock implementation for tests
type mockUserStore struct {
    users map[int]*User
}

func (m *mockUserStore) GetUser(id int) (*User, error) {
    u, ok := m.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return u, nil
}

func TestService_GetUser(t *testing.T) {
    store := &mockUserStore{
        users: map[int]*User{1: {ID: 1, Name: "Alice"}},
    }
    svc := NewService(store)
    
    user, err := svc.GetUser(1)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("got name %q; want Alice", user.Name)
    }
}
```

### Benchmarks
```go
func BenchmarkProcess(b *testing.B) {
    data := generateTestData()
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        Process(data)
    }
}

func BenchmarkProcess_Parallel(b *testing.B) {
    data := generateTestData()
    
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            Process(data)
        }
    })
}
```

## Performance

### Memory Optimization
```go
// Pre-allocate slices when size is known
items := make([]Item, 0, expectedSize)

// Use strings.Builder for string concatenation
var b strings.Builder
b.Grow(estimatedSize)
for _, s := range parts {
    b.WriteString(s)
}
result := b.String()

// Avoid unnecessary allocations
// Bad: creates new slice on each append
func collect(items []Item, newItems ...Item) []Item {
    return append(items, newItems...)
}

// Good: modify in place when capacity allows
func collect(items *[]Item, newItems ...Item) {
    *items = append(*items, newItems...)
}
```

### Profiling
```go
// CPU profiling
import "runtime/pprof"

f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()

// Memory profiling
import "runtime/pprof"

f, _ := os.Create("mem.prof")
pprof.WriteHeapProfile(f)
f.Close()

// HTTP profiling endpoint
import _ "net/http/pprof"

go http.ListenAndServe(":6060", nil)
// Then: go tool pprof http://localhost:6060/debug/pprof/heap
```

## Project Structure

```
myproject/
├── cmd/
│   └── myapp/
│       └── main.go           # Entry point
├── internal/
│   ├── config/               # Configuration
│   ├── handler/              # HTTP handlers
│   ├── service/              # Business logic
│   └── repository/           # Data access
├── pkg/                      # Public libraries
│   └── mylib/
├── api/                      # API definitions (proto, OpenAPI)
├── go.mod
├── go.sum
└── Makefile
```

## Common Gotchas

### Loop Variable Capture
```go
// Bug: all goroutines share same variable
for _, v := range items {
    go func() {
        process(v) // v is shared!
    }()
}

// Fix: capture in closure parameter
for _, v := range items {
    go func(v Item) {
        process(v)
    }(v)
}

// Fix (Go 1.22+): loop variables are per-iteration
for _, v := range items {
    go func() {
        process(v) // OK in Go 1.22+
    }()
}
```

### Nil Interface vs Nil Value
```go
type MyError struct{ msg string }
func (e *MyError) Error() string { return e.msg }

func returnsError() error {
    var err *MyError = nil
    return err // Returns non-nil interface containing nil pointer!
}

func main() {
    if err := returnsError(); err != nil {
        // This WILL execute because err interface is not nil
    }
}

// Fix: return nil directly
func returnsError() error {
    var err *MyError
    if err == nil {
        return nil // Return untyped nil
    }
    return err
}
```

### Defer in Loops
```go
// Bug: defers accumulate until function returns
for _, f := range files {
    file, _ := os.Open(f)
    defer file.Close() // Won't close until loop function returns
    process(file)
}

// Fix: use anonymous function
for _, f := range files {
    func() {
        file, _ := os.Open(f)
        defer file.Close()
        process(file)
    }()
}
```

## References

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [Go Proverbs](https://go-proverbs.github.io/)
- [Standard Go Project Layout](https://github.com/golang-standards/project-layout)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
