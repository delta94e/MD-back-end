# Part 6 — Concurrency

> Concurrency la diem manh lon nhat cua Go.
> `goroutine` nhe hon `thread` 1000x, `channel` giai quyet race condition bang thiet ke.
> **"Do not communicate by sharing memory; share memory by communicating."** — Go Proverb

---

## 6.1 — Goroutines

### Goroutine La Gi?

```
╔══════════════════════════════════════════════════════════════════════╗
║  THREAD vs GOROUTINE                                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  OS Thread:                                                          ║
║  - Stack co dinh: 1-8 MB                                            ║
║  - Tao/huy ton kem (syscall)                                         ║
║  - Context switch: ~1-10 microseconds                               ║
║  - He dieu hanh quan ly (kernel scheduling)                         ║
║                                                                      ║
║  Goroutine:                                                          ║
║  - Stack ban dau: 2-4 KB (tu mo rong)                               ║
║  - Tao rat nhanh (~1 microsecond)                                   ║
║  - Context switch: ~100 nanoseconds                                 ║
║  - Go runtime quan ly (user-space scheduling, M:N model)            ║
║                                                                      ║
║  → Co the chay hang trieu goroutines tren mot machine               ║
╚══════════════════════════════════════════════════════════════════════╝
```

```go
// Khoi chay goroutine: them tu khoa "go" truoc function call
// Ham duoc goi chay DONG THOI, khong block caller

func say(s string) {
    for i := 0; i < 3; i++ {
        time.Sleep(100 * time.Millisecond)
        fmt.Println(s)
    }
}

func main() {
    go say("world") // chay song song
    say("hello")    // chay trong main goroutine
}
// Output (thu tu co the thay doi):
// hello
// world
// hello
// world
// hello
// world
```

```go
// CHAY NHIEU GOROUTINES
func fetchURL(url string) {
    resp, err := http.Get(url)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp.Body.Close()
    fmt.Printf("%s → %d\n", url, resp.StatusCode)
}

urls := []string{
    "https://golang.org",
    "https://google.com",
    "https://github.com",
}

// Khong dung goroutine: tuan tu ~3 seconds (1 giay moi URL)
for _, url := range urls {
    fetchURL(url)
}

// Dung goroutine: song song ~1 second!
for _, url := range urls {
    go fetchURL(url) // moi URL chay dong thoi
}
// Van de: main() co the ket thuc truoc khi goroutines xong
// Giai phap: WaitGroup hoac Channel (xem duoi)
```

### GOROUTINE GOTCHA — Closure + Loop

```go
// BUG DIEN HINH: closure capture bien loop

// SAI — tat ca goroutines share cung bien i
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i) // i co the la 3, 3, 3 (gia tri cuoi)
    }()
}

// DUNG — truyen i nhu argument (copy)
for i := 0; i < 3; i++ {
    go func(n int) {
        fmt.Println(n) // n la ban sao cua i tai thoi diem goi
    }(i)
}

// DUNG (Go 1.22+) — loop variable moi moi iteration
for i := range 3 {
    go func() {
        fmt.Println(i) // i rieng cho moi iteration
    }()
}
```

---

## 6.2 — Channels

### Khoi Tao Va Su Dung Co Ban

```go
// Channel = "ong" de goroutines giao tiep an toan
// Syntax: chan T (hai chieu), <-chan T (chi doc), chan<- T (chi ghi)

// Tao channel: make(chan Type)
ch := make(chan int)     // unbuffered channel - dong bo

// Ghi vao channel (blocking neu khong co ai doc)
ch <- 42

// Doc tu channel (blocking cho den khi co data)
v := <-ch

// Doc va kiem tra closed
v, ok := <-ch  // ok = false neu channel da dong

// UNBUFFERED CHANNEL: sender va receiver phai "gap nhau"
// → Sync point: sender block cho den khi receiver san sang va nguoc lai

func sum(s []int, ch chan int) {
    total := 0
    for _, v := range s {
        total += v
    }
    ch <- total  // gui ket qua vao channel
}

func main() {
    s := []int{7, 2, 8, -9, 4, 0}

    ch := make(chan int)
    go sum(s[:len(s)/2], ch)  // tinh nua dau
    go sum(s[len(s)/2:], ch)  // tinh nua sau

    x, y := <-ch, <-ch  // doi ca 2 goroutines
    fmt.Println(x, y, x+y) // -5 17 12 (thu tu co the dao)
}
```

### Buffered Channels

```go
// Buffered channel: co hang doi
// Syntax: make(chan T, capacity)
// → Sender chi block khi buffer DAY
// → Receiver chi block khi buffer RONG

ch := make(chan int, 3) // buffer size = 3

// Gui 3 gia tri ma khong can goroutine doc (buffer chua day)
ch <- 1
ch <- 2
ch <- 3
// ch <- 4  // BLOCK! buffer day

// Doc ra
fmt.Println(<-ch) // 1
fmt.Println(<-ch) // 2
fmt.Println(<-ch) // 3

// Ung dung: worker pool
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {   // range tren channel - doc cho den khi dong
        time.Sleep(time.Second)  // simulate work
        results <- j * 2
    }
}

func main() {
    jobs    := make(chan int, 100)   // buffer lon tranh block
    results := make(chan int, 100)

    // Khoi tao 5 workers
    for w := 1; w <= 5; w++ {
        go worker(w, jobs, results)
    }

    // Gui 20 jobs
    for j := 1; j <= 20; j++ {
        jobs <- j
    }
    close(jobs) // bao workers khong con job nua

    // Thu thap ket qua
    for a := 1; a <= 20; a++ {
        fmt.Println(<-results)
    }
}
```

### Range and Close

```go
// Dong channel: bao rang khong con du lieu nua
// Chi SENDER moi nen dong channel (receiver khong dong)

func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n  // gui tung so vao channel
        }
        close(out)  // bao da xong
    }()
    return out  // tra ve channel chi-doc
}

// Range tren channel: tu dong stop khi channel dong
for n := range generator(2, 3, 5, 7, 11, 13) {
    fmt.Println(n)
}
// In: 2, 3, 5, 7, 11, 13

// Pipeline pattern — ket noi cac stages
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Ket hop:
nums := generator(2, 3, 4, 5)
squares := square(nums)
for n := range squares {
    fmt.Println(n) // 4, 9, 16, 25
}
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  CHANNEL DIRECTION — TYPE SAFETY                                     ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  chan int    → doc VÀ ghi (bidirectional)                           ║
║  <-chan int  → chi DOC (receive-only)  → "output" cua goroutine     ║
║  chan<- int  → chi GHI (send-only)     → "input" cua goroutine     ║
║                                                                      ║
║  Su dung trong function signature:                                  ║
║  func producer(out chan<- int)   → chi gui, khong doc               ║
║  func consumer(in <-chan int)    → chi nhan, khong gui              ║
║  func main() {                                                      ║
║      ch := make(chan int)  // bidirectional                         ║
║      go producer(ch)       // Go tu cast sang chan<- int            ║
║      consumer(ch)          // Go tu cast sang <-chan int            ║
║  }                                                                  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 6.3 — Select

```go
// Select = switch cho channels
// Block cho den khi MOT trong cac cases san sang

ch1 := make(chan string)
ch2 := make(chan string)

go func() { time.Sleep(1*time.Second); ch1 <- "one" }()
go func() { time.Sleep(2*time.Second); ch2 <- "two" }()

// Doi 2 messages, xu ly cai nao den truoc
for i := 0; i < 2; i++ {
    select {
    case msg1 := <-ch1:
        fmt.Println("Received from ch1:", msg1)
    case msg2 := <-ch2:
        fmt.Println("Received from ch2:", msg2)
    }
}
// Output:
// Received from ch1: one   (sau 1 giay)
// Received from ch2: two   (sau 2 giay)
```

### Default Selection — Non-Blocking

```go
// "default" chay NGAY khi khong co case nao san sang
// → Non-blocking channel operation

ch := make(chan int, 1)

// Non-blocking send:
select {
case ch <- 42:
    fmt.Println("sent 42")
default:
    fmt.Println("no one receiving, skip") // chay neu ch day
}

// Non-blocking receive:
select {
case v := <-ch:
    fmt.Println("got:", v)
default:
    fmt.Println("no value ready") // chay neu ch rong
}
```

### Select Voi Timeout Va Cancel

```go
// TIMEOUT — quan trong trong production!
func fetchWithTimeout(url string, timeout time.Duration) (string, error) {
    ch := make(chan string, 1)

    go func() {
        resp, err := http.Get(url)
        if err != nil {
            ch <- ""
            return
        }
        defer resp.Body.Close()
        body, _ := io.ReadAll(resp.Body)
        ch <- string(body)
    }()

    select {
    case result := <-ch:
        return result, nil
    case <-time.After(timeout):
        return "", fmt.Errorf("timeout after %v", timeout)
    }
}

// CANCELLATION — dung voi context (production pattern)
func fetchWithContext(ctx context.Context, url string) (string, error) {
    ch := make(chan string, 1)
    errCh := make(chan error, 1)

    go func() {
        req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
        resp, err := http.DefaultClient.Do(req)
        if err != nil {
            errCh <- err
            return
        }
        defer resp.Body.Close()
        body, _ := io.ReadAll(resp.Body)
        ch <- string(body)
    }()

    select {
    case result := <-ch:
        return result, nil
    case err := <-errCh:
        return "", err
    case <-ctx.Done():
        return "", ctx.Err() // context.DeadlineExceeded or context.Canceled
    }
}
```

---

## 6.4 — sync.Mutex

```go
// Mutex = MUTual EXclusion lock
// Dung khi nhieu goroutines chia se 1 bien (shared state)
// Thu hep "vung nguy hiem" (critical section)

// VAN DE: race condition
type UnsafeCounter struct {
    count int
}

func (c *UnsafeCounter) Inc() { c.count++ } // KHONG AN TOAN!
// count++ = doc + tinh + ghi = 3 buoc rieng
// Neu 2 goroutines cung luc: co the mat du lieu!

// GIAI PHAP: sync.Mutex
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()         // gianh lock (block neu goroutine khac dang giu)
    defer c.mu.Unlock() // tra lock khi ham ket thuc (luon dung defer!)
    c.count++           // chi mot goroutine vao duoc day tai 1 thoi diem
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

// Chay 1000 goroutines cung tang count
counter := &SafeCounter{}
var wg sync.WaitGroup

for i := 0; i < 1000; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        counter.Inc()
    }()
}

wg.Wait()
fmt.Println(counter.Value()) // 1000 — luon dung
```

### RWMutex — Toi Uu Hon Mutex Thuong

```go
// RWMutex (Read/Write Mutex):
// - Nhieu goroutines co the DOC cung luc (read lock)
// - Chi 1 goroutine co the GHI   (write lock, block moi read)
// → Hieu qua hon khi read nhieu, write it

type UserCache struct {
    mu    sync.RWMutex
    users map[string]*User
}

// Read: dung RLock (nhieu goroutines cung doc duoc)
func (c *UserCache) Get(id string) (*User, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    user, ok := c.users[id]
    return user, ok
}

// Write: dung Lock (chi 1 goroutine, block ca read)
func (c *UserCache) Set(id string, user *User) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.users[id] = user
}
```

### sync.WaitGroup — Doi Nhieu Goroutines

```go
// WaitGroup: doi mot nhom goroutines hoan thanh
// Add(n)  → them n vao counter
// Done()  → giam counter di 1 (goi khi goroutine xong)
// Wait()  → block cho den khi counter = 0

func processItems(items []string) []string {
    var (
        wg      sync.WaitGroup
        mu      sync.Mutex
        results []string
    )

    for _, item := range items {
        wg.Add(1)
        go func(s string) {
            defer wg.Done()

            // Xu ly song song (simulate work)
            processed := strings.ToUpper(s)
            time.Sleep(10 * time.Millisecond)

            // Ghi ket qua an toan
            mu.Lock()
            results = append(results, processed)
            mu.Unlock()
        }(item)
    }

    wg.Wait()  // doi tat ca goroutines xong
    return results
}

items := []string{"apple", "banana", "cherry", "date"}
results := processItems(items)
fmt.Println(results) // [APPLE BANANA CHERRY DATE] (thu tu ngau nhien)
```

### sync.Once — Khoi Tao 1 Lan

```go
// sync.Once: dam bao 1 func chi chay DUNG 1 LAN du goi tu nhieu goroutines
// Dung cho: singleton, lazy initialization

type Database struct {
    conn *sql.DB
}

var (
    db   *Database
    once sync.Once
)

func GetDB() *Database {
    once.Do(func() {
        // Chi chay 1 lan, du GetDB() goi tu 100 goroutines dong thoi
        conn, err := sql.Open("postgres", os.Getenv("DB_URL"))
        if err != nil {
            panic(err)
        }
        db = &Database{conn: conn}
    })
    return db
}
```

---

## 6.5 — Patterns Thuc Te

### Worker Pool Pattern

```go
// Gioi han so goroutines chay dong thoi (tranh over-provision)

type Job struct {
    ID  int
    URL string
}

type Result struct {
    Job    Job
    Status int
    Err    error
}

func workerPool(ctx context.Context, numWorkers int, jobs []Job) []Result {
    jobCh    := make(chan Job, len(jobs))
    resultCh := make(chan Result, len(jobs))

    // Khoi chay workers
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobCh {
                // Kiem tra context truoc khi xu ly
                select {
                case <-ctx.Done():
                    return
                default:
                }

                resp, err := http.Get(job.URL)
                if err != nil {
                    resultCh <- Result{Job: job, Err: err}
                    continue
                }
                resp.Body.Close()
                resultCh <- Result{Job: job, Status: resp.StatusCode}
            }
        }()
    }

    // Gui jobs
    for _, job := range jobs {
        jobCh <- job
    }
    close(jobCh)

    // Dong resultCh khi tat ca workers xong
    go func() {
        wg.Wait()
        close(resultCh)
    }()

    // Thu thap ket qua
    var results []Result
    for r := range resultCh {
        results = append(results, r)
    }
    return results
}

// Su dung:
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

jobs := []Job{{1, "https://a.com"}, {2, "https://b.com"}, {3, "https://c.com"}}
results := workerPool(ctx, 3, jobs)  // toi da 3 goroutines cung luc
```

### Fan-Out / Fan-In Pattern

```go
// Fan-Out: 1 input → nhieu workers xu ly song song
// Fan-In:  ket qua tu nhieu workers → 1 channel output

// Fan-out: phan phoi viec cho nhieu workers
func fanOut[T, R any](
    ctx context.Context,
    input <-chan T,
    numWorkers int,
    fn func(T) R,
) []<-chan R {
    outputs := make([]<-chan R, numWorkers)
    for i := 0; i < numWorkers; i++ {
        out := make(chan R)
        outputs[i] = out
        go func() {
            defer close(out)
            for v := range input {
                select {
                case <-ctx.Done():
                    return
                case out <- fn(v):
                }
            }
        }()
    }
    return outputs
}

// Fan-in: gop ket qua tu nhieu channels
func fanIn[T any](ctx context.Context, inputs ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup

    forward := func(ch <-chan T) {
        defer wg.Done()
        for v := range ch {
            select {
            case <-ctx.Done():
                return
            case out <- v:
            }
        }
    }

    wg.Add(len(inputs))
    for _, ch := range inputs {
        go forward(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

---

## 6.6 — Detecting Race Conditions

```go
// Go co race detector built-in!
// Chay: go run -race main.go
//       go test -race ./...

// Vi du race condition:
var count int

func increment() { count++ }  // KHONG AN TOAN

go increment()
go increment()
// go run -race → WARNING: DATA RACE

// Kiem tra race condition trong CI/CD:
// go test -race -count=5 ./...
// → Chay 5 lan de tang kha nang phat hien race
```

---

## 6.7 — So Sanh: Channel vs Mutex

```
╔══════════════════════════════════════════════════════════════════════╗
║  CHANNEL vs MUTEX — Khi Nao Dung Cai Nao?                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  CHANNEL — dung khi:                                                ║
║  → Truyen du lieu GIUA cac goroutines                               ║
║  → Bao hieu su kien (done, cancel, error)                           ║
║  → Pipeline: goroutine A san xuat, B tieu thu                       ║
║  → Ket qua tu nhieu goroutines song song                            ║
║                                                                      ║
║  MUTEX — dung khi:                                                  ║
║  → Bao ve SHARED STATE (map, slice, counter, cache)                 ║
║  → Don gian hon channel cho read/write guard                        ║
║  → In-memory cache, pool, singleton                                 ║
║                                                                      ║
║  CA HAI — dung khi:                                                 ║
║  → WorkerPool: channel phan phoi job, mutex bao ve result           ║
║                                                                      ║
║  NGUYEN TAC: "Share memory by communicating (channel)"              ║
║  → Goi y dung channel khi co the, mutex khi thuc su can             ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 6.8 — Ung Dung Trong Clean Architecture

```go
// Use case xu ly bat dong bo voi timeout va cancel

type CreateOrderUseCase struct {
    orderRepo   OrderRepository
    paymentSvc  PaymentService
    emailSvc    EmailService
    logger      *zap.Logger
}

func (uc *CreateOrderUseCase) Execute(
    ctx context.Context,
    req CreateOrderRequest,
) (*Order, error) {
    // 1. Tao order
    order, err := uc.orderRepo.Save(ctx, &Order{
        UserID: req.UserID,
        Items:  req.Items,
        Status: OrderStatusPending,
    })
    if err != nil {
        return nil, fmt.Errorf("CreateOrder: save: %w", err)
    }

    // 2. Xu ly payment va gui email SONG SONG
    // Dung goroutine + channel bao tap ket qua
    type result struct {
        name string
        err  error
    }
    results := make(chan result, 2)

    go func() {
        err := uc.paymentSvc.Charge(ctx, order.ID, req.PaymentInfo)
        results <- result{"payment", err}
    }()

    go func() {
        err := uc.emailSvc.SendConfirmation(ctx, order)
        results <- result{"email", err}
    }()

    // Doi ca 2, xu ly loi
    for i := 0; i < 2; i++ {
        r := <-results
        if r.err != nil {
            uc.logger.Error("CreateOrder: async step failed",
                zap.String("step", r.name),
                zap.Error(r.err))
            // Email fail khong quan trong, payment fail thi huy order
            if r.name == "payment" {
                _ = uc.orderRepo.Delete(ctx, order.ID)
                return nil, fmt.Errorf("CreateOrder: payment: %w", r.err)
            }
        }
    }

    return order, nil
}
```

---

## Cau Hoi Tu Kiem Tra

1. Goroutine khac Thread o diem nao? Tai sao Go co the chay hang trieu goroutines?
2. Unbuffered channel khac Buffered channel the nao? Khi nao dung cai nao?
3. `close(ch)` lam gi? Ai nen dong channel — sender hay receiver?
4. `select` voi `default` khac `select` khong co `default` the nao?
5. Race condition la gi? Lam the nao Go giup phat hien no?
6. `sync.Mutex` vs `sync.RWMutex` — khi nao dung cai nao?
7. `sync.WaitGroup` dung de lam gi? Tai sao phai `wg.Add(1)` truoc khi `go`?
8. `sync.Once` giai quyet van de gi? Dung trong truong hop nao?
9. Tai sao phai truyen `context.Context` vao goroutine?
10. Worker pool giai quyet van de gi khi co qua nhieu goroutines?

---

*Sau phan nay ban da nam du toan bo Go Fundamentals!*
*Tiep theo: [Lesson 01 — Project Setup & Clean Architecture](./lesson-01-project-setup-and-architecture.md)*
