# Concurrency Is Not Parallelism — Đồng Thời Không Phải Song Song

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài nói "Concurrency is not Parallelism" của Rob Pike (đồng tác giả Go)
> **Góc nhìn:** Phân biệt Concurrency vs Parallelism — nền tảng thiết kế Go
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                   | Mô tả                                 |
| --- | ---------------------------------------- | ------------------------------------- |
| §1  | Thế giới là Concurrent, không phải OOP   | Tại sao concurrency quan trọng        |
| §2  | Concurrency ≠ Parallelism                | Định nghĩa chính xác từng khái niệm   |
| §3  | Bài toán Gopher đốt sách                 | 6 thiết kế concurrent giải 1 bài toán |
| §4  | Goroutines, Channels, Select             | 3 công cụ concurrency trong Go        |
| §5  | Ví dụ thực tế: Load Balancer             | Từ đơn giản đến realistic             |
| §6  | Replicated Database Query                | Fan-out pattern trên 1 slide          |
| §7  | CSP — Communicating Sequential Processes | Nền tảng lý thuyết (Tony Hoare 1978)  |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior      | Ôn tập & thực hành                    |

---

## §1. Thế Giới Là Concurrent

### 1.1 Thế giới thực = đồng thời

```
╔═══════════════════════════════════════════════════════════════╗
║   THẾ GIỚI THỰC = CONCURRENT!                                ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Rob Pike:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "If you looked at programming languages of   │             ║
║  │  today you'd get the idea the world is       │             ║
║  │  OBJECT-ORIENTED. But it's NOT.              │             ║
║  │  It's actually PARALLEL."                    │             ║
║  │                                              │             ║
║  │ → Mọi thứ xảy ra ĐỒNG THỜI:               │             ║
║  │   • CPU multicore                            │             ║
║  │   • Network packets                          │             ║
║  │   • Users đang click                         │             ║
║  │   • Databases đang ghi                      │             ║
║  │   • Hành tinh đang quay!                    │             ║
║  │                                              │             ║
║  │ → Nhưng tools lập trình hiện tại KHÔNG      │             ║
║  │   express được worldview này!               │             ║
║  │ → Go được tạo ra để FIX điều này!          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  KHI GO RA MẮT (2009):                                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Programmers: "Concurrent tools! Tuyệt!       │             ║
║  │ Giờ tôi chạy SONG SONG trên nhiều CPU!"    │             ║
║  │                                              │             ║
║  │ → Họ chạy chương trình trên nhiều CPU hơn  │             ║
║  │ → Chương trình CHẬM HƠN!                   │             ║
║  │ → "Go broken! Không work!"                  │             ║
║  │                                              │             ║
║  │ Rob Pike: "What was broken was the           │             ║
║  │ WORLDVIEW" — Cái sai là TƯ DUY!            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Concurrency ≠ Parallelism

### 2.1 Định nghĩa chính xác

```
╔═══════════════════════════════════════════════════════════════╗
║   CONCURRENCY vs PARALLELISM — KHÁC NHAU!                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CONCURRENCY (Đồng thời):                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = COMPOSITION of independently executing     │             ║
║  │   things (typically functions)                │             ║
║  │                                              │             ║
║  │ = Cách PHÂN TÁCH bài toán thành các phần   │             ║
║  │   THỰC THI ĐỘC LẬP                         │             ║
║  │                                              │             ║
║  │ → Về STRUCTURE (cấu trúc)                   │             ║
║  │ → "DEALING with lots of things at once"     │             ║
║  │ → XỬ LÝ nhiều thứ cùng lúc                 │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  PARALLELISM (Song song):                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = SIMULTANEOUS execution of multiple things  │             ║
║  │                                              │             ║
║  │ = Nhiều thứ CHẠY CÙNG LÚC thật sự          │             ║
║  │   (trên nhiều CPU/core)                      │             ║
║  │                                              │             ║
║  │ → Về EXECUTION (thực thi)                    │             ║
║  │ → "DOING lots of things at once"            │             ║
║  │ → LÀM nhiều thứ cùng lúc                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ SO SÁNH:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ CONCURRENCY (1 CPU):                          │             ║
║  │ CPU: ─A──B──A──C──B──A──C──B──►             │             ║
║  │      Xen kẽ A, B, C trên 1 CPU             │             ║
║  │      → CẤU TRÚC concurrent                  │             ║
║  │      → KHÔNG chạy đồng thời thật!          │             ║
║  │                                              │             ║
║  │ PARALLELISM (nhiều CPU):                      │             ║
║  │ CPU1: ─A──A──A──A──A──►                     │             ║
║  │ CPU2: ─B──B──B──B──B──►                     │             ║
║  │ CPU3: ─C──C──C──C──C──►                     │             ║
║  │      → THỰC THI đồng thời thật!            │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  MỐI QUAN HỆ:                                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Concurrency is a way to STRUCTURE a thing   │             ║
║  │  so that you can MAYBE use parallelism to    │             ║
║  │  do a better job. But parallelism is NOT     │             ║
║  │  the GOAL of concurrency."                   │             ║
║  │                                              │             ║
║  │ → Concurrency = CẤU TRÚC!                   │             ║
║  │ → Parallelism = TÙY CHỌN (có thể có, có   │             ║
║  │   thể không)!                                │             ║
║  │ → Mục tiêu concurrency = GOOD STRUCTURE!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VÍ DỤ QUEN THUỘC — HỆ ĐIỀU HÀNH:                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Mouse driver ──┐                              │             ║
║  │ Keyboard driver─┤                             │             ║
║  │ Display driver ─┼→ OS quản lý CONCURRENT    │             ║
║  │ Network driver ─┤   trên 1 CPU!              │             ║
║  │ Disk driver ────┘                             │             ║
║  │                                              │             ║
║  │ → 5 drivers CONCURRENT                       │             ║
║  │ → Nhưng KHÔNG NHẤT THIẾT PARALLEL!          │             ║
║  │ → 1 CPU = chỉ 1 driver chạy tại 1 thời    │             ║
║  │   điểm, nhưng thiết kế VẪN ĐÚNG!           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Bài Toán Gopher Đốt Sách

### 3.1 Bài toán gốc

```
╔═══════════════════════════════════════════════════════════════╗
║   BÀI TOÁN GOPHER ĐỐT SÁCH — 6 THIẾT KẾ                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  BÀI TOÁN: Đốt đống sách C++ cũ!                            ║
║  → Có đống sách (pile)                                        ║
║  → Có lò đốt (incinerator)                                   ║
║  → Gopher dùng xe đẩy (cart) chở sách                       ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 3.2 Thiết kế 1: 1 Gopher

```
    THIẾT KẾ 1: 1 Gopher (Tuần tự)

    ┌──────┐     🐿️ →→→     ┌──────┐
    │ SÁCH │  ──────────────→ │ LÒ   │
    │ 📚📚 │   cart 🛒        │ 🔥🔥 │
    └──────┘  ←←←←←←←←←←←←  └──────┘

    → 1 Gopher, 1 cart
    → Chạy qua chạy lại
    → CHẬM! Nhưng đúng!
```

### 3.3 Thiết kế 2: 2 Gophers song song

```
    THIẾT KẾ 2: 2 Gophers (Parallel!)

    ┌──────┐  🐿️ →→→ 🛒     ┌──────┐
    │ SÁCH │  ──────────────→ │ LÒ   │
    │ 📚📚 │                  │ 🔥🔥 │
    └──────┘  ──────────────→ └──────┘
              🐿️ →→→ 🛒

    → 2 Gophers, mỗi con 1 cart
    → NHANH HƠN! Nhưng cần ĐỒNG BỘ!
    → Có thể kẹt ở pile hoặc ở lò!
    → Gophers gửi messages để phối hợp!
```

### 3.4 Thiết kế 3: Pipeline 3 Gophers

```
    THIẾT KẾ 3: Pipeline (3 Gophers, 3 vai trò!)

    ┌──────┐    🐿️₁     🐿️₂     🐿️₃    ┌──────┐
    │ SÁCH │ → [LOAD] → [CARRY] → [UNLOAD]→│ LÒ   │
    │ 📚📚 │                                │ 🔥🔥 │
    └──────┘                                └──────┘

    → Gopher 1: CHỈ xếp sách lên xe
    → Gopher 2: CHỈ chở xe đi
    → Gopher 3: CHỈ dỡ sách vào lò

    → MỖI gopher = 1 việc đơn giản!
    → NHƯNG: Gopher 2 phải chạy về lấy xe trống!
    → Thời gian chạy về = LÃNG PHÍ!
```

### 3.5 Thiết kế 4: Pipeline 4 Gophers (thêm 1!)

```
    THIẾT KẾ 4: Pipeline + Return (4 Gophers!)

    ┌──────┐  🐿️₁     🐿️₂     🐿️₃    ┌──────┐
    │ SÁCH │→[LOAD]→ [CARRY→]→[UNLOAD]→│ LÒ   │
    │ 📚📚 │                             │ 🔥🔥 │
    └──────┘ ←←←←← 🐿️₄ ←←←←←←←←←←←← └──────┘
                  [RETURN EMPTIES]

    → Gopher 4: CHỈ trả xe trống về!
    → THÊM 1 gopher nhưng NHANH HƠN!

    ╔══════════════════════════════════════════╗
    ║  INSIGHT QUAN TRỌNG:                      ║
    ║  Thêm concurrent procedure vào design    ║
    ║  → Tổng thể NHANH HƠN!                 ║
    ║  → Thêm NHIỀU việc hơn nhưng            ║
    ║    chạy NHANH hơn!                       ║
    ║                                          ║
    ║  "We improved performance by ADDING      ║
    ║   a concurrent procedure to an           ║
    ║   existing design."                      ║
    ║  → ADDING things made it MORE EFFICIENT! ║
    ╚══════════════════════════════════════════╝
```

### 3.6 Thiết kế 5: Staging dump

```
    THIẾT KẾ 5: Staging Dump (Trạm trung chuyển)

    ┌──────┐  🐿️₁    ┌──────┐  🐿️₂    ┌──────┐
    │ SÁCH │→[CARRY]→ │ DUMP │→[CARRY]→ │ LÒ   │
    │ 📚📚 │          │  📚  │          │ 🔥🔥 │
    └──────┘  ←←←←←  └──────┘  ←←←←←  └──────┘

    → Gopher 1: chở sách → dump → chạy về
    → Gopher 2: chở sách dump → lò → chạy về
    → 2 Gophers, KHÁC procedure!
    → Có thể NHANH GẤP ĐÔI design 1!
    → Hoàn toàn KHÁC thiết kế trước!
```

### 3.7 Scale & Compose

```
╔═══════════════════════════════════════════════════════════════╗
║   SCALE & COMPOSE — KẾT HỢP CÁC THIẾT KẾ!                  ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Có thể PARALLELIZE bất kỳ thiết kế nào:                    ║
║                                                               ║
║  Design 4 × 2 instances = 8 Gophers!                         ║
║  Design 5 × 2 instances = 4 Gophers!                         ║
║  Design 5 + Design 4 = 8 Gophers!                            ║
║  Kết hợp cả 2 × 2 = 16 Gophers!                             ║
║                                                               ║
║  BÀI HỌC THEN CHỐT:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Có NHIỀU cách concurrent cho 1 bài toán │             ║
║  │ 2. Không nhất thiết tương đương             │             ║
║  │ 3. Nhưng đều CÓ THỂ hoạt động đúng       │             ║
║  │ 4. Có thể SCALE trên nhiều chiều           │             ║
║  │                                              │             ║
║  │ 5. QUAN TRỌNG NHẤT:                          │             ║
║  │    ┌──────────────────────────────────┐      │             ║
║  │    │ Thiết kế vẫn ĐÚNG dù chỉ có   │      │             ║
║  │    │ 1 CPU! Chỉ 1 Gopher chạy tại  │      │             ║
║  │    │ 1 thời điểm → VẪN ĐÚNG!       │      │             ║
║  │    │                                  │      │             ║
║  │    │ "The design is still CORRECT    │      │             ║
║  │    │  and that's a pretty big deal   │      │             ║
║  │    │  because it means we don't have │      │             ║
║  │    │  to worry about parallelism     │      │             ║
║  │    │  when doing concurrency."       │      │             ║
║  │    │                                  │      │             ║
║  │    │ → Parallelism = FREE VARIABLE!  │      │             ║
║  │    └──────────────────────────────────┘      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ÁP DỤNG THỰC TẾ:                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Pile of books  = Web content / Data          │             ║
║  │ Gophers        = CPUs / Goroutines           │             ║
║  │ Carts          = Networking / Marshalling    │             ║
║  │ Incinerator    = Web browser / Consumer      │             ║
║  │                                              │             ║
║  │ → Bạn vừa xây WEB SERVER ARCHITECTURE!     │             ║
║  │ → Proxies, buffers, scaling = trên sơ đồ!  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Goroutines, Channels, Select

### 4.1 Ba công cụ concurrency trong Go

```
╔═══════════════════════════════════════════════════════════════╗
║   3 CÔNG CỤ CONCURRENCY TRONG GO                             ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  1. GOROUTINE — "Gopher"                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Giống threads nhưng RẺ HƠN RẤT NHIỀU    │             ║
║  │ → ~2KB stack (thread ~1MB!)                  │             ║
║  │ → Có thể tạo HÀNG TRIỆU goroutines!       │             ║
║  │   (Rob Pike debug production: 1.3 triệu!)   │             ║
║  │ → Multiplexed lên OS threads tự động       │             ║
║  │ → Khi 1 goroutine block → không ảnh hưởng │             ║
║  │   goroutines khác!                            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  2. CHANNEL — "Đường ống giao tiếp"                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Typed pipe giữa goroutines                │             ║
║  │ → Gửi và nhận values                        │             ║
║  │ → Block khi chưa có data (synchronization!) │             ║
║  │ → First-class value: truyền channel          │             ║
║  │   như tham số, trong struct, etc.            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  3. SELECT — "Multi-way concurrent switch"                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Lắng nghe NHIỀU channels cùng lúc        │             ║
║  │ → Channel nào sẵn sàng → thực thi case đó │             ║
║  │ → Cả 2 sẵn sàng → chọn RANDOM!            │             ║
║  │ → Có default clause cho non-blocking        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 4.2 Code examples

```go
// ═══ 3 CÔNG CỤ CỦA GO CONCURRENCY ═══

// 1. GOROUTINE — Keyword "go"
func main() {
    // Gọi bình thường → CHỜĐỢI f() xong
    f(a, b)

    // Thêm "go" → f() chạy NỀN, main tiếp tục!
    go f(a, b)
    // → Giống "f &" trong shell!
    // → main() KHÔNG chờ f() xong!
}

// 2. CHANNEL — Giao tiếp giữa goroutines
func main() {
    // Tạo channel kiểu time.Time
    timer := make(chan time.Time)

    // Goroutine: sleep rồi gửi thời gian
    go func() {
        time.Sleep(5 * time.Second)
        timer <- time.Now() // GỬI vào channel
    }()

    // Main: làm việc khác...
    doOtherWork()

    // Khi cần kết quả → NHẬN từ channel
    completedAt := <-timer // BLOCK cho đến khi có data!
    fmt.Println("Done at:", completedAt)
}

// 3. SELECT — Lắng nghe nhiều channels
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    // ... launch goroutines gửi vào ch1, ch2

    select {
    case msg := <-ch1:
        fmt.Println("From ch1:", msg)
    case msg := <-ch2:
        fmt.Println("From ch2:", msg)
    default:
        fmt.Println("No one ready!")
    }
    // → Ai sẵn sàng TRƯỚC → chạy case đó!
    // → Cả 2 sẵn sàng → RANDOM!
    // → Không ai sẵn sàng → default!
}

// CLOSURES — Tạo concurrent procedures!
func compose(f, g func(x float64) float64) func(x float64) float64 {
    return func(x float64) float64 {
        return f(g(x))
    }
}
// → Closures = essential cho concurrent Go!
// → Tạo functions as values, truyền qua channels!
```

---

## §5. Ví Dụ Thực Tế: Load Balancer

### 5.1 Simple Load Balancer

```go
// ═══ SIMPLE LOAD BALANCER ═══

// Bài toán: N workers xử lý M jobs

// 1. Định nghĩa Work
type Work struct {
    x, y, z int
}

// 2. Worker: đọc input → tính toán → ghi output
func worker(in <-chan Work, out chan<- int) {
    for w := range in {
        // range channel = đọc cho đến khi channel closed!
        result := w.x * w.y * w.z
        time.Sleep(time.Duration(w.z)) // Giả lập blocking
        out <- result
    }
}

// 3. TOÀN BỘ SOLUTION!
func main() {
    in := make(chan Work)
    out := make(chan int)

    // Khởi tạo N workers — TẤT CẢ đọc CÙNG 1 channel!
    for i := 0; i < numWorkers; i++ {
        go worker(in, out)
    }

    // Tạo work (chạy nền)
    go func() {
        for i := 0; i < numJobs; i++ {
            in <- Work{x: i, y: i + 1, z: i % 3}
        }
        close(in)
    }()

    // Nhận kết quả
    for i := 0; i < numJobs; i++ {
        result := <-out
        fmt.Println(result)
    }
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   PHÂN TÍCH SIMPLE LOAD BALANCER                              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  SƠ ĐỒ:                                                       ║
║                                                               ║
║              ┌──────────┐                                     ║
║         ┌───→│ Worker 1 │───┐                                 ║
║         │    └──────────┘   │                                 ║
║  ┌────┐ │    ┌──────────┐   │  ┌─────┐                       ║
║  │ in ├─┼───→│ Worker 2 │───┼─→│ out │                       ║
║  └────┘ │    └──────────┘   │  └─────┘                       ║
║         │    ┌──────────┐   │                                 ║
║         └───→│ Worker N │───┘                                 ║
║              └──────────┘                                     ║
║                                                               ║
║  ĐẶC ĐIỂM:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ✅ KHÔNG CÓ mutex/lock!                      │             ║
║  │ ✅ Đúng dù 1 CPU hay 1000 CPU!              │             ║
║  │ ✅ numWorkers có thể là BẤT KỲ số nào!     │             ║
║  │ ✅ Scale tự nhiên!                            │             ║
║  │ ✅ Implicitly parallel!                       │             ║
║  │                                              │             ║
║  │ Rob Pike: "No locking, no mutexes... and    │             ║
║  │ yet this is a CORRECT concurrent and         │             ║
║  │ parallelizable algorithm."                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 5.2 Realistic Load Balancer (Heap-based)

```go
// ═══ REALISTIC LOAD BALANCER ═══

// Request chứa function + channel trả kết quả!
type Request struct {
    fn   func() int     // Hàm cần tính
    resCh chan int       // Channel trả kết quả
    // → Channel LÀ first-class value!
    // → Requester và Worker nói chuyện TRỰC TIẾP!
    // → Balancer KHÔNG cần relay kết quả!
}

// Worker có channel riêng + đếm pending tasks
type Worker struct {
    requests chan Request // Channel nhận work
    pending  int         // Số tasks đang chờ
}

// Worker xử lý: nhận request → tính → trả kết quả
func (w *Worker) work(done chan *Worker) {
    for req := range w.requests {
        result := req.fn()     // Tính toán
        req.resCh <- result    // Trả TRỰC TIẾP cho requester!
        done <- w              // Báo balancer: tôi xong!
    }
}

// Balancer: chỉ SELECT giữa 2 events!
func (b *Balancer) balance(work chan Request) {
    for {
        select {
        case req := <-work:
            // Có request mới → gửi cho worker ÍT VIỆC NHẤT!
            b.dispatch(req)
        case w := <-b.done:
            // Worker xong → cập nhật load!
            b.completed(w)
        }
    }
}

// Dispatch: 4 dòng executable!
func (b *Balancer) dispatch(req Request) {
    w := heap.Pop(&b.pool).(*Worker) // Worker ít việc nhất
    w.requests <- req                 // Gửi work
    w.pending++                       // Tăng load
    heap.Push(&b.pool, w)             // Đặt lại heap
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   SƠ ĐỒ REALISTIC LOAD BALANCER                              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Requesters          Balancer          Workers                ║
║  ┌─────────┐                          ┌──────────┐           ║
║  │Requester│──work──→┌────────┐──req─→│ Worker 1 │──┐       ║
║  │    1    │         │        │       └──────────┘  │       ║
║  └─────────┘         │        │       ┌──────────┐  │       ║
║  ┌─────────┐         │BALANCE │──req─→│ Worker 2 │──┤       ║
║  │Requester│──work──→│  (heap)│       └──────────┘  │done   ║
║  │    2    │         │        │       ┌──────────┐  │       ║
║  └─────────┘         │        │──req─→│ Worker N │──┤       ║
║  ┌─────────┐         └────────┘←done──└──────────┘  │       ║
║  │Requester│──work──→    ▲                          │       ║
║  │    N    │             └──────────────────────────┘       ║
║  └─────────┘                                                 ║
║                                                               ║
║  ĐẶC BIỆT:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Kết quả trả TRỰC TIẾP Worker→Requester!  │             ║
║  │ → Balancer KHÔNG relay kết quả!              │             ║
║  │ → Channels are FIRST-CLASS values!           │             ║
║  │ → Channel truyền TRONG Request struct!       │             ║
║  │ → Không có locking!                           │             ║
║  │ → Scalable: N requesters, M workers!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Replicated Database Query

### 6.1 Fan-out pattern — Fit trên 1 slide!

```go
// ═══ REPLICATED DATABASE QUERY ═══
// Bài toán: Query N replicas, lấy kết quả NHANH NHẤT!

func Query(conns []Conn, query string) Result {
    // Channel buffered = số replicas
    ch := make(chan Result, len(conns))

    // Fan-out: gửi query tới TẤT CẢ replicas!
    for _, conn := range conns {
        go func(c Conn) {
            ch <- c.DoQuery(query) // Mỗi replica 1 goroutine
        }(conn)
    }

    // Lấy kết quả ĐẦU TIÊN!
    return <-ch
    // → Ai trả lời TRƯỚC = ta dùng!
    // → Replicas khác? Kệ, kết quả giống nhau!
    // → 1 replica chậm/chết? KHÔNG ẢNH HƯỞNG!
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   FAN-OUT PATTERN — SƠ ĐỒ                                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║               ┌────────────┐                                  ║
║          ┌───→│ Replica 1  │───┐                              ║
║          │    └────────────┘   │                              ║
║  ┌─────┐ │    ┌────────────┐   │  ┌────┐                     ║
║  │Query├─┼───→│ Replica 2  │───┼─→│ ch │→ return FIRST!     ║
║  └─────┘ │    └────────────┘   │  └────┘                     ║
║          │    ┌────────────┐   │                              ║
║          └───→│ Replica 3  │───┘                              ║
║               └────────────┘                                  ║
║                                                               ║
║  → Query song song TẤT CẢ replicas                           ║
║  → Lấy kết quả ĐẦU TIÊN                                    ║
║  → Latency = MIN(replica latencies)!                         ║
║  → Fault tolerant: 1 replica down = OK!                      ║
║                                                               ║
║  Rob Pike: "This looks like a toy but it's                   ║
║  actually a COMPLETE CORRECT implementation."                ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. CSP — Nền Tảng Lý Thuyết

### 7.1 Tony Hoare 1978

```
╔═══════════════════════════════════════════════════════════════╗
║   CSP — COMMUNICATING SEQUENTIAL PROCESSES                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Tony Hoare (1978):                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Communicating Sequential Processes"          │             ║
║  │                                              │             ║
║  │ Rob Pike: "TRULY one of the greatest papers  │             ║
║  │ in computer science. If anything from this   │             ║
║  │ talk sinks in, GO HOME AND READ THAT PAPER."│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Ý TƯỞNG CHÍNH:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Processes chạy ĐỘC LẬP                  │             ║
║  │ 2. GIAO TIẾP qua message passing            │             ║
║  │ 3. KHÔNG shared memory!                      │             ║
║  │ 4. Synchronize qua communication!            │             ║
║  │                                              │             ║
║  │ Go Proverb:                                   │             ║
║  │ "Don't communicate by sharing memory;        │             ║
║  │  share memory by communicating."              │             ║
║  │                                              │             ║
║  │  ❌ Shared memory → mutex → deadlocks       │             ║
║  │  ✅ Channels → message passing → SAFE!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CSP → GO:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ CSP Process     → Go goroutine              │             ║
║  │ CSP Channel     → Go channel                │             ║
║  │ CSP Guard/Alt   → Go select                 │             ║
║  │                                              │             ║
║  │ Ngôn ngữ khác dùng CSP: Erlang, Occam      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §8. Tổng Kết & Câu Hỏi Phỏng Vấn Senior Golang

### 8.1 Câu hỏi phỏng vấn

```
╔═══════════════════════════════════════════════════════════════╗
║   CÂU HỎI PHỎNG VẤN SENIOR GOLANG                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Q1: "Concurrency vs Parallelism — khác nhau?"              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Concurrency = COMPOSITION of independently   │             ║
║  │ executing things (STRUCTURE)                  │             ║
║  │ → "Dealing with lots of things at once"      │             ║
║  │                                              │             ║
║  │ Parallelism = SIMULTANEOUS execution         │             ║
║  │ (EXECUTION)                                   │             ║
║  │ → "Doing lots of things at once"             │             ║
║  │                                              │             ║
║  │ Concurrency ENABLES parallelism!              │             ║
║  │ Nhưng concurrency KHÔNG YÊU CẦU parallelism!│             ║
║  │ Mục tiêu concurrency = GOOD STRUCTURE!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Goroutine khác thread thế nào?"                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Goroutine:          Thread:                   │             ║
║  │ ~2KB stack           ~1MB stack               │             ║
║  │ Go runtime schedule  OS schedule             │             ║
║  │ Millions possible    Thousands max           │             ║
║  │ Channel communicate  Mutex/lock              │             ║
║  │ Multiplexed onto     1:1 with OS thread      │             ║
║  │ OS threads (M:N)                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Tại sao Go dùng channels thay vì mutex?"              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ CSP model (Tony Hoare 1978):                  │             ║
║  │ "Share memory by communicating,               │             ║
║  │  don't communicate by sharing memory."        │             ║
║  │                                              │             ║
║  │ Channels: explicit, composable, no deadlocks │             ║
║  │ Mutex: implicit, error-prone, deadlock risk  │             ║
║  │                                              │             ║
║  │ Load balancer trên: 0 mutex, 100% correct!  │             ║
║  │ (Mutex vẫn có trong Go nhưng cho low-level) │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Select statement dùng khi nào?"                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Khi cần lắng nghe NHIỀU channels cùng lúc:  │             ║
║  │ → Timeout + work: select giữa timer và data │             ║
║  │ → Fan-in: gom nhiều sources vào 1           │             ║
║  │ → Load balancer: work mới vs worker done    │             ║
║  │ → Graceful shutdown: done signal + requests  │             ║
║  │                                              │             ║
║  │ Có default → non-blocking                    │             ║
║  │ Không default → blocking cho đến khi 1 ch   │             ║
║  │ sẵn sàng                                    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: CONCURRENCY IS NOT PARALLELISM               │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Concurrency = STRUCTURE               │            │
    │  │    Parallelism = EXECUTION                │            │
    │  │    → KHÁC NHAU HOÀN TOÀN!               │            │
    │  │                                          │            │
    │  │ 2. Concurrency ENABLES parallelism        │            │
    │  │    → Cấu trúc đúng → scale tự nhiên    │            │
    │  │    → Parallelism = FREE VARIABLE!         │            │
    │  │                                          │            │
    │  │ 3. Thêm concurrency = có thể NHANH HƠN  │            │
    │  │    → 4 Gophers > 3 Gophers dù thêm     │            │
    │  │      công việc!                           │            │
    │  │                                          │            │
    │  │ 4. Go tools: goroutine + channel + select │            │
    │  │    → Không mutex, không lock              │            │
    │  │    → Correct dù 1 CPU hay 1000 CPU       │            │
    │  │                                          │            │
    │  │ 5. CSP (Hoare 1978) = nền tảng           │            │
    │  │    → "Share memory by communicating"       │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  Rob Pike:                                               │
    │  "Concurrency is POWERFUL. It's not parallelism.        │
    │   But it ENABLES parallelism.                           │
    │   And it makes parallelism EASY."                       │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
