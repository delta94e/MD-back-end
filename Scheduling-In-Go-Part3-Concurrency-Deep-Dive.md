# Scheduling In Go: Part III — Concurrency Deep Dive

> **Nguồn gốc**: William Kennedy — "Scheduling In Go: Part III - Concurrency"
> **Mục đích**: Tài liệu ôn tập phỏng vấn Senior Golang — hoàn toàn bằng Tiếng Việt
> **Đối tượng**: Người mới bắt đầu, chưa biết gì về backend
> **Series**: Part I (OS Scheduler) → Part II (Go Scheduler) → **Part III (Concurrency)** ← BẠN ĐANG ĐÂY

---

## Mục Lục

1. [§1. Giới Thiệu — Concurrency Là Gì?](#1-giới-thiệu--concurrency-là-gì)
2. [§2. Concurrency vs Parallelism — Khác Nhau Hoàn Toàn!](#2-concurrency-vs-parallelism--khác-nhau-hoàn-toàn)
3. [§3. Phân Loại Workload — CPU-Bound vs IO-Bound](#3-phân-loại-workload--cpu-bound-vs-io-bound)
4. [§4. CPU-Bound: Bài Toán Cộng Số](#4-cpu-bound-bài-toán-cộng-số)
5. [§5. CPU-Bound: Bubble Sort — Khi Concurrency KHÔNG Giúp Ích](#5-cpu-bound-bubble-sort--khi-concurrency-không-giúp-ích)
6. [§6. IO-Bound: Đọc File & Tìm Kiếm](#6-io-bound-đọc-file--tìm-kiếm)
7. [§7. Tổng Kết & Câu Hỏi Phỏng Vấn](#7-tổng-kết--câu-hỏi-phỏng-vấn)

---

## §1. Giới Thiệu — Concurrency Là Gì?

### Tư duy của William Kennedy

```
╔═══════════════════════════════════════════════════════════════════════╗
║                                                                       ║
║  "When I'm solving a problem, especially if it's a new problem,      ║
║   I don't initially think about whether concurrency is a good fit    ║
║   or not. I look for a SEQUENTIAL solution first and make sure       ║
║   that it's working."                                                 ║
║                                                                       ║
║                                          — William Kennedy            ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

**Quy trình tư duy đúng:**

```
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  BƯỚC 1: Viết giải pháp TUẦN TỰ (sequential) trước!           │
    │     ↓                                                           │
    │  BƯỚC 2: Đảm bảo nó HOẠT ĐỘNG ĐÚNG!                          │
    │     ↓                                                           │
    │  BƯỚC 3: Review code (readability + technical review)           │
    │     ↓                                                           │
    │  BƯỚC 4: HỎI: "Concurrency có hợp lý và thực tế không?"      │
    │     ↓                                                           │
    │  ┌────────────────────┬────────────────────┐                   │
    │  │ CÓ! Rõ ràng       │ KHÔNG rõ ràng      │                   │
    │  │ → Thêm concurrency │ → GIỮ sequential!  │                   │
    │  └────────────────────┴────────────────────┘                   │
    │                                                                 │
    │  ★ RULE: Concurrency phải thêm ĐỦ performance gain           │
    │    để BÙ LẠI complexity cost!                                  │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### Định nghĩa Concurrency

```
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  CONCURRENCY = "OUT OF ORDER" EXECUTION                      │
    │                                                              │
    │  → Lấy 1 tập hợp instructions VỐN CHẠY TUẦN TỰ            │
    │  → Tìm cách chạy KHÔNG THEO THỨ TỰ                        │
    │  → VẪN RA CÙNG KẾT QUẢ!                                   │
    │                                                              │
    │  VÍ DỤ ĐƠNN GIẢN:                                            │
    │                                                              │
    │  Sequential:  Task A → Task B → Task C → KẾT QUẢ          │
    │                                                              │
    │  Concurrent:  Task A ─┐                                      │
    │               Task B ─┼─→ KẾT QUẢ (CÙNG KẾT QUẢ!)       │
    │               Task C ─┘                                      │
    │                                                              │
    │  ĐIỀU KIỆN:                                                   │
    │  1. Out of order execution phải KHẢ THI                     │
    │  2. Phải thêm VALUE (performance gain > complexity cost)     │
    │  3. Kết quả phải GIỐNG NHƯ sequential!                     │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §2. Concurrency vs Parallelism — Khác Nhau Hoàn Toàn!

### Figure 1 — Concurrency vs Parallelism

```
    ┌──────────────────────────────────────────────────────────────────────┐
    │                                                                      │
    │     P1 (Processor 1)                    P2 (Processor 2)            │
    │                                                                      │
    │         /\         ┌──────┐                /\         ┌──────┐      │
    │        /  \  ──── │ Core │               /  \  ──── │ Core │      │
    │       / M  \       └──────┘              / M  \       └──────┘      │
    │      /______\                           /______\                     │
    │         │                                  │                         │
    │       (G2) ← executing                  (G1) ← executing           │
    │         │                                  │                         │
    │     ┌───┴───┐        LRQ            ┌───┴───┐        LRQ           │
    │     │       │   ┌────┴──────┐       │       │   ┌────┴──────┐      │
    │     │  P1   │   │           │       │  P2   │   │           │      │
    │     │       │  (G3) (G5)   │       │       │  (G4) (G6)   │      │
    │     └───────┘   └───────────┘       └───────┘   └───────────┘      │
    │                                                                      │
    │  ◄─────── PARALLELISM ──────►                                       │
    │  G2 và G1 executing ĐỒNG THỜI                                      │
    │  trên 2 Core KHÁC NHAU!                                            │
    │                                                                      │
    │  ◄── CONCURRENCY ──►   ◄── CONCURRENCY ──►                        │
    │  G2/G3/G5 chia sẻ P1    G1/G4/G6 chia sẻ P2                      │
    │  luân phiên trên 1 M     luân phiên trên 1 M                      │
    │                                                                      │
    └──────────────────────────────────────────────────────────────────────┘
```

### Giải thích chi tiết

```
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  CONCURRENCY ≠ PARALLELISM!                                  │
    │                                                              │
    │  ┌───────────────────┬───────────────────────────────────┐   │
    │  │ CONCURRENCY       │ PARALLELISM                       │   │
    │  ├───────────────────┼───────────────────────────────────┤   │
    │  │ "Out of order"    │ "At the same time"                │   │
    │  │ execution         │ execution                         │   │
    │  ├───────────────────┼───────────────────────────────────┤   │
    │  │ Nhiều G chia sẻ  │ Nhiều G chạy ĐỒNG THỜI          │   │
    │  │ 1 M/Core          │ trên nhiều M/Core                │   │
    │  ├───────────────────┼───────────────────────────────────┤   │
    │  │ Chỉ cần 1 Core   │ Cần ≥ 2 Cores                   │   │
    │  ├───────────────────┼───────────────────────────────────┤   │
    │  │ G luân phiên      │ G thực sự chạy cùng lúc         │   │
    │  ├───────────────────┼───────────────────────────────────┤   │
    │  │ Quản lý THỜI GIAN │ Nhân đôi NĂNG LỰC              │   │
    │  └───────────────────┴───────────────────────────────────┘   │
    │                                                              │
    │  ★ CẨN THẬN:                                                │
    │  → Concurrency KHÔNG CÓ parallelism có thể CHẬM HƠN!     │
    │  → Concurrency CÓ parallelism KHÔNG PHẢI LÚC NÀO          │
    │    cũng nhanh hơn bạn nghĩ!                                │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §3. Phân Loại Workload — CPU-Bound vs IO-Bound

### Hai loại workload cốt lõi

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    2 LOẠI WORKLOAD CỐT LÕI                           ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────┐     ║
║  │ 1. CPU-BOUND (Bị giới hạn bởi CPU)                         │     ║
║  │                                                              │     ║
║  │ → Goroutine KHÔNG BAO GIỜ tự nhiên vào trạng thái Waiting │     ║
║  │ → Liên tục tính toán, tính toán, tính toán...              │     ║
║  │ → VÍ DỤ: Tính Pi đến chữ số thứ N                        │     ║
║  │          Cộng số, sắp xếp, mã hóa, hash...               │     ║
║  │                                                              │     ║
║  │  Goroutine:  ████████████████████████ (luôn EXECUTING!)    │     ║
║  │              KHÔNG CÓ khoảng trống!                        │     ║
║  │                                                              │     ║
║  └─────────────────────────────────────────────────────────────┘     ║
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────┐     ║
║  │ 2. IO-BOUND (Bị giới hạn bởi IO)                           │     ║
║  │                                                              │     ║
║  │ → Goroutine TỰ NHIÊN vào trạng thái Waiting               │     ║
║  │ → Phải đợi resource bên ngoài (network, disk, mutex...)   │     ║
║  │ → VÍ DỤ: Đọc file, gọi API, query database               │     ║
║  │          Mutex lock, atomic operations...                   │     ║
║  │                                                              │     ║
║  │  Goroutine:  ████░░░░████░░░░████░░░░ (có WAITING!)       │     ║
║  │              exec wait exec wait exec wait                 │     ║
║  │                                                              │     ║
║  └─────────────────────────────────────────────────────────────┘     ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### CPU-Bound: Khi nào cần Parallelism?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  CPU-BOUND + Concurrency KHÔNG CÓ Parallelism = CHẬM HƠN! │
    │                                                              │
    │  TẠI SAO?                                                    │
    │                                                              │
    │  1 Core, nhiều Goroutines CPU-bound:                        │
    │                                                              │
    │  G1: ████──switch──████──switch──████                       │
    │  G2:      ████──switch──████──switch──████                  │
    │                                                              │
    │  → G1, G2 KHÔNG BAO GIỜ waiting tự nhiên!                 │
    │  → Mỗi lần context switch = "STOP THE WORLD"!              │
    │  → KHÔNG có work nào được làm trong lúc switch!            │
    │  → Switch cost = LATENCY THUẦN = LÃNG PHÍ!                 │
    │                                                              │
    │  KẾT LUẬN:                                                   │
    │  ★ CPU-Bound CẦN PARALLELISM để có lợi từ concurrency!   │
    │  ★ 1 Goroutine / 1 OS Thread / 1 Core = TỐI ƯU!          │
    │  ★ Nhiều G hơn số Core = CHẬM HƠN!                        │
    └──────────────────────────────────────────────────────────────┘
```

### IO-Bound: Concurrency KHÔNG cần Parallelism!

```
    ┌──────────────────────────────────────────────────────────────┐
    │  IO-BOUND + Concurrency KHÔNG CẦN Parallelism = ĐÃ NHANH! │
    │                                                              │
    │  TẠI SAO?                                                    │
    │                                                              │
    │  1 Core, nhiều Goroutines IO-bound:                         │
    │                                                              │
    │  G1: ████░░░░░░░░████░░░░░░░░████                          │
    │  G2:     ████░░░░░░░░████░░░░░░░░                           │
    │  G3:         ████░░░░░░░░████                                │
    │                                                              │
    │  → G1 waiting (IO) → G2 nhảy vào dùng Core!               │
    │  → G2 waiting (IO) → G3 nhảy vào dùng Core!               │
    │  → Context switch KHÔNG phải "Stop The World"!              │
    │  → Vì G đã DỪNG TỰ NHIÊN (waiting for IO)!               │
    │  → Core KHÔNG BAO GIỜ IDLE = HIỆU QUẢ TỐI ĐA!          │
    │                                                              │
    │  KẾT LUẬN:                                                   │
    │  ★ IO-Bound KHÔNG CẦN parallelism!                        │
    │  ★ 1 Core + nhiều G = ĐÃ RẤT HIỆU QUẢ!                 │
    │  ★ Thêm Core = KHÔNG cải thiện đáng kể!                  │
    └──────────────────────────────────────────────────────────────┘
```

### Bảng tóm tắt quyết định

```
    ┌──────────────────┬───────────────────┬───────────────────┐
    │                  │ CPU-BOUND         │ IO-BOUND          │
    ├──────────────────┼───────────────────┼───────────────────┤
    │ Cần Parallelism? │ CÓ! ✅ Bắt buộc  │ KHÔNG cần ❌     │
    ├──────────────────┼───────────────────┼───────────────────┤
    │ Concurrency      │ Chỉ có lợi KHI   │ Luôn có lợi!     │
    │ có lợi khi nào? │ có parallelism!   │ Kể cả 1 Core!   │
    ├──────────────────┼───────────────────┼───────────────────┤
    │ Số G tối ưu     │ = Số Cores!       │ > Số Cores OK!   │
    │                  │ Nhiều hơn = chậm! │ Nhiều hơn = tốt! │
    ├──────────────────┼───────────────────┼───────────────────┤
    │ Context switch   │ = "Stop World"    │ = Tận dụng idle  │
    │ impact           │ 100% overhead!    │ FREE switching!  │
    ├──────────────────┼───────────────────┼───────────────────┤
    │ Ví dụ           │ Cộng số, sort     │ File I/O, HTTP   │
    │                  │ Hash, encrypt     │ DB query, mutex  │
    └──────────────────┴───────────────────┴───────────────────┘
```

---

## §4. CPU-Bound: Bài Toán Cộng Số

### Bài toán: Cộng 10 triệu số

```
    ┌──────────────────────────────────────────────────────────────┐
    │  BÀI TOÁN: Cho 1 slice 10,000,000 số nguyên.               │
    │  YÊU CẦU: Tính tổng tất cả các số.                        │
    │                                                              │
    │  HỎI: Có thể dùng concurrency không?                       │
    │  TRẢ LỜI: CÓ! Vì có thể chia list thành nhiều phần nhỏ, │
    │  tính tổng từng phần, rồi cộng các tổng con lại!          │
    │                                                              │
    │  HỎI: Đây là workload gì?                                  │
    │  TRẢ LỜI: CPU-BOUND! Vì chỉ tính toán thuần túy,         │
    │  không có IO, không có waiting!                              │
    └──────────────────────────────────────────────────────────────┘
```

### Phiên bản Sequential — Hàm `add`

```go
func add(numbers []int) int {
    var v int
    for _, n := range numbers {
        v += n
    }
    return v
}
```

```
    Luồng thực thi Sequential:

    numbers: [1, 3, 5, 2, 8, 4, 7, 6, ...]  (10 triệu số)
             ──────────────────────────→
             v += 1 → v += 3 → v += 5 → ...

    → 1 Goroutine, duyệt TOÀN BỘ list từ đầu đến cuối.
    → Đơn giản! 5 dòng code!
    → Chỉ cần 1 Core!
```

### Phiên bản Concurrent — Hàm `addConcurrent`

```go
func addConcurrent(goroutines int, numbers []int) int {
    var v int64
    totalNumbers := len(numbers)
    lastGoroutine := goroutines - 1
    stride := totalNumbers / goroutines

    var wg sync.WaitGroup
    wg.Add(goroutines)

    for g := 0; g < goroutines; g++ {
        go func(g int) {
            start := g * stride
            end := start + stride
            if g == lastGoroutine {
                end = totalNumbers
            }

            var lv int
            for _, n := range numbers[start:end] {
                lv += n
            }

            atomic.AddInt64(&v, int64(lv))
            wg.Done()
        }(g)
    }

    wg.Wait()
    return int(v)
}
```

### Sơ đồ hoạt động: 8 Goroutines chia nhau 10 triệu số

```
    10,000,000 số chia cho 8 Goroutines:

    numbers: [═══════════════════════════════════════════════════]
              ↓         ↓         ↓         ↓         ↓
             ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐ ...
             │ G0  │  │ G1  │  │ G2  │  │ G3  │  │ G4  │
             │1.25M│  │1.25M│  │1.25M│  │1.25M│  │1.25M│
             │ số  │  │ số  │  │ số  │  │ số  │  │ số  │
             └──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘
                │        │        │        │        │
             sum=A    sum=B    sum=C    sum=D    sum=E  ...
                │        │        │        │        │
                └────────┴────┬───┴────────┴────────┘
                              │
                    atomic.AddInt64(&v, ...)
                              │
                         TỔNG CUỐI

    QUAN TRỌNG:
    → stride = 10,000,000 / 8 = 1,250,000 số mỗi G
    → G cuối cùng (lastGoroutine) lấy phần DƯ thêm
    → Mỗi G tính tổng phần CỦA MÌNH (lv = biến local!)
    → Cuối cùng: atomic.AddInt64 cộng an toàn vào v
    → wg.Wait() đợi TẤT CẢ G xong!
```

### Benchmark: CPU-Bound — 1 Core vs 8 Cores

```
╔═══════════════════════════════════════════════════════════════════════╗
║  BENCHMARK CPU-BOUND: 10 Triệu Số, 8 Goroutines                     ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ═══ THỬ 1: Concurrency KHÔNG CÓ Parallelism (1 Core) ═══         ║
║                                                                       ║
║  $ GOGC=off go test -cpu 1 -bench .                                  ║
║                                                                       ║
║  BenchmarkSequential     5,720,764 ns/op  ← ~10% NHANH HƠN! ✅    ║
║  BenchmarkConcurrent     6,387,344 ns/op  ← CHẬM HƠN! ❌          ║
║                                                                       ║
║  ┌──────────────────────────────────────────────────────────┐        ║
║  │ TẠI SAO Sequential NHANH HƠN?                           │        ║
║  │                                                          │        ║
║  │ 1 Core chia cho 8 Goroutines CPU-bound:                 │        ║
║  │                                                          │        ║
║  │ G0: ███──sw──███──sw──███                               │        ║
║  │ G1:    ███──sw──███──sw──███                            │        ║
║  │ G2:       ███──sw──███──sw──███                         │        ║
║  │ ...                                                      │        ║
║  │                                                          │        ║
║  │ → "sw" = context switch = LÃNG PHÍ THỜI GIAN!         │        ║
║  │ → 8 G trên 1 Core = 7 lần switch overhead không cần!  │        ║
║  │ → Sequential: 0 switch = NHANH hơn 10-13%!             │        ║
║  └──────────────────────────────────────────────────────────┘        ║
║                                                                       ║
║  ═══ THỬ 2: Concurrency CÓ Parallelism (8 Cores) ═══              ║
║                                                                       ║
║  $ GOGC=off go test -cpu 8 -bench .                                  ║
║                                                                       ║
║  BenchmarkSequential-8   5,910,799 ns/op                             ║
║  BenchmarkConcurrent-8   3,362,643 ns/op  ← ~43% NHANH HƠN! ✅    ║
║                                                                       ║
║  ┌──────────────────────────────────────────────────────────┐        ║
║  │ TẠI SAO Concurrent NHANH HƠN?                           │        ║
║  │                                                          │        ║
║  │ 8 Cores, mỗi Core 1 Goroutine:                         │        ║
║  │                                                          │        ║
║  │ Core 1: G0 ████████████████                             │        ║
║  │ Core 2: G1 ████████████████                             │        ║
║  │ Core 3: G2 ████████████████  ← THỰC SỰ SONG SONG!    │        ║
║  │ Core 4: G3 ████████████████                             │        ║
║  │ ...                                                      │        ║
║  │                                                          │        ║
║  │ → 0 context switch! Mỗi G có Core riêng!              │        ║
║  │ → 8 G song song = ~43% nhanh hơn! ✅                  │        ║
║  │ → Tại sao không 8x? Overhead: atomic, WaitGroup, GC   │        ║
║  └──────────────────────────────────────────────────────────┘        ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

## §5. CPU-Bound: Bubble Sort — Khi Concurrency KHÔNG Giúp Ích

### Bubble Sort — Thuật toán sắp xếp

```go
func bubbleSort(numbers []int) {
    n := len(numbers)
    for i := 0; i < n; i++ {
        if !sweep(numbers, i) {
            return
        }
    }
}

func sweep(numbers []int, currentPass int) bool {
    var idx int
    idxNext := idx + 1
    n := len(numbers)
    var swap bool

    for idxNext < (n - currentPass) {
        a := numbers[idx]
        b := numbers[idxNext]
        if a > b {
            numbers[idx] = b
            numbers[idxNext] = a
            swap = true
        }
        idx++
        idxNext = idx + 1
    }
    return swap
}
```

### Tại sao Bubble Sort KHÔNG phù hợp với Concurrency?

```
    ┌──────────────────────────────────────────────────────────────┐
    │ HỎI: bubbleSort có phù hợp "out of order" execution?       │
    │ TRẢ LỜI: KHÔNG! ❌                                         │
    │                                                              │
    │ VẤN ĐỀ: Sau khi chia list → sort từng phần →              │
    │ KHÔNG CÓ CÁCH HIỆU QUẢ để ghép lại!                      │
    │                                                              │
    │ VÍ DỤ: 36 số chia 3 phần:                                  │
    │                                                              │
    │ TRƯỚC:                                                       │
    │ Phần 1: [25 51 15 57 87 10 10 85 90 32 98 53]              │
    │ Phần 2: [91 82 84 97 67 37 71 94 26  2 81 79]              │
    │ Phần 3: [66 70 93 86 19 81 52 75 85 10 87 49]              │
    │                                                              │
    │ SAU KHI SORT TỪNG PHẦN (concurrent):                        │
    │ Phần 1: [10 10 15 25 32 51 53 57 85 87 90 98]  ← OK       │
    │ Phần 2: [ 2 26 37 67 71 79 81 82 84 91 94 97]  ← OK       │
    │ Phần 3: [10 19 49 52 66 70 75 81 85 86 87 93]  ← OK       │
    │                                                              │
    │ NHƯNG GHÉP LẠI:                                              │
    │ [10 10 15 25 32 51 53 57 85 87 90 98 | 2 26 37 67...]      │
    │ ← CHƯA SORT TOÀN BỘ!!! ❌                                  │
    │ → Phải SORT LẠI TOÀN BỘ = LÃNG PHÍ!                      │
    │ → bubbleSort(numbers) lần nữa = HỦY HẾT concurrent gain! │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ★ BÀI HỌC QUAN TRỌNG:                                     │
    │                                                              │
    │  KHÔNG PHẢI TẤT CẢ CPU-bound workloads                     │
    │  đều phù hợp với concurrency!                               │
    │                                                              │
    │  Concurrency KHÔNG phù hợp khi:                             │
    │  1. Chi phí CHIA NHỎ work quá cao                          │
    │  2. Chi phí GỘP KẾT QUẢ quá cao (★ Bubble Sort!)         │
    │  3. Kết quả concurrent = phải sort lại = vô nghĩa!        │
    │                                                              │
    │  SO SÁNH:                                                    │
    │  ┌───────────────┬──────────┬────────────────────────┐      │
    │  │ Thuật toán    │ OK?      │ Lý do                  │      │
    │  ├───────────────┼──────────┼────────────────────────┤      │
    │  │ add (cộng số)│ CÓ ✅    │ Chia list → cộng      │      │
    │  │              │          │ → atomic.Add = RẺ!     │      │
    │  ├───────────────┼──────────┼────────────────────────┤      │
    │  │ bubbleSort   │ KHÔNG ❌ │ Chia list → sort       │      │
    │  │              │          │ → ghép lại = ĐẮT!     │      │
    │  └───────────────┴──────────┴────────────────────────┘      │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §6. IO-Bound: Đọc File & Tìm Kiếm

### Bài toán: Tìm topic trong 1000 file

```
    ┌──────────────────────────────────────────────────────────────┐
    │  BÀI TOÁN: Cho 1000 files XML.                              │
    │  YÊU CẦU: Đọc từng file, tìm xem có chứa "topic" không?  │
    │  TRẢ VỀ: Số lần topic xuất hiện.                           │
    │                                                              │
    │  HỎI: Đây là workload gì?                                  │
    │  TRẢ LỜI: IO-BOUND! Vì mỗi lần đọc file = disk I/O      │
    │  = Goroutine phải WAITING cho disk trả data!               │
    │                                                              │
    │  HỎI: Cần parallelism không?                               │
    │  TRẢ LỜI: KHÔNG! IO-Bound chỉ cần concurrency!           │
    └──────────────────────────────────────────────────────────────┘
```

### Phiên bản Sequential — Hàm `find`

```go
func find(topic string, docs []string) int {
    var found int
    for _, doc := range docs {
        items, err := read(doc)
        if err != nil {
            continue
        }
        for _, item := range items {
            if strings.Contains(item.Description, topic) {
                found++
            }
        }
    }
    return found
}

func read(doc string) ([]item, error) {
    time.Sleep(time.Millisecond) // Giả lập disk I/O ~1ms
    var d document
    if err := xml.Unmarshal([]byte(file), &d); err != nil {
        return nil, err
    }
    return d.Channel.Items, nil
}
```

```
    Luồng thực thi Sequential:

    doc1 → read(1ms) → search → doc2 → read(1ms) → search → ...
           ░░░░░░░░    ████         ░░░░░░░░    ████

    → 1000 docs × 1ms mỗi doc = ~1000ms = ~1 giây!
    → Goroutine WAITING 1ms mỗi lần read!
    → Core IDLE trong lúc waiting = LÃNG PHÍ!
```

### Phiên bản Concurrent — Hàm `findConcurrent`

```go
func findConcurrent(goroutines int, topic string, docs []string) int {
    var found int64

    ch := make(chan string, len(docs))
    for _, doc := range docs {
        ch <- doc
    }
    close(ch)

    var wg sync.WaitGroup
    wg.Add(goroutines)

    for g := 0; g < goroutines; g++ {
        go func() {
            var lFound int64
            for doc := range ch {
                items, err := read(doc)
                if err != nil {
                    continue
                }
                for _, item := range items {
                    if strings.Contains(item.Description, topic) {
                        lFound++
                    }
                }
            }
            atomic.AddInt64(&found, lFound)
            wg.Done()
        }()
    }

    wg.Wait()
    return int(found)
}
```

### Sơ đồ: Worker Pool Pattern

```
    BUFFERED CHANNEL = "Hàng đợi công việc"

    docs: [doc1, doc2, doc3, ..., doc1000]
              │
              ▼
    ┌─────────────────────────────────────┐
    │  ch := make(chan string, len(docs)) │
    │  [doc1|doc2|doc3|...|doc1000]       │ ← BUFFERED!
    │  close(ch) ← đóng sau khi đầy!    │
    └─────────────┬───────────────────────┘
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
    ┌──────┐ ┌──────┐ ┌──────┐    ...  ┌──────┐
    │  G0  │ │  G1  │ │  G2  │         │  G7  │
    │      │ │      │ │      │         │      │
    │ for  │ │ for  │ │ for  │         │ for  │
    │ doc  │ │ doc  │ │ doc  │         │ doc  │
    │ :=   │ │ :=   │ │ :=   │         │ :=   │
    │range │ │range │ │range │         │range │
    │ ch   │ │ ch   │ │ ch   │         │ ch   │
    └──┬───┘ └──┬───┘ └──┬───┘         └──┬───┘
       │        │        │                │
       │   lFound    lFound          lFound
       │        │        │                │
       └────────┴────┬───┴────────────────┘
                     │
           atomic.AddInt64(&found, ...)
                     │
                TỔNG CUỐI

    → 8 Worker Goroutines cùng lấy docs từ channel!
    → G nào xong doc hiện tại → lấy doc tiếp theo!
    → Channel tự động phân phối = LOAD BALANCING!
    → Khi channel hết docs → for range tự kết thúc!
```

### Benchmark: IO-Bound — 1 Core vs 8 Cores

```
╔═══════════════════════════════════════════════════════════════════════╗
║  BENCHMARK IO-BOUND: 1000 Docs, 8 Goroutines                         ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ═══ THỬ 1: Concurrency KHÔNG CÓ Parallelism (1 Core) ═══         ║
║                                                                       ║
║  $ GOGC=off go test -cpu 1 -bench .                                  ║
║                                                                       ║
║  BenchmarkSequential     1,483,458,120 ns/op                         ║
║  BenchmarkConcurrent       188,941,855 ns/op  ← ~87% NHANH HƠN! ✅ ║
║                                                                       ║
║  ┌──────────────────────────────────────────────────────────┐        ║
║  │ TẠI SAO Concurrent NHANH HƠN DÙ CHỈ 1 CORE?           │        ║
║  │                                                          │        ║
║  │ Sequential (1 G, 1 Core):                               │        ║
║  │ G0: ██░░░░██░░░░██░░░░██░░░░██░░░░                     │        ║
║  │     exec wait exec wait exec wait  ← Core IDLE khi wait│        ║
║  │                                                          │        ║
║  │ Concurrent (8 G, 1 Core):                               │        ║
║  │ G0: ██░░░░░░░░░░░░░░██                                 │        ║
║  │ G1:   ██░░░░░░░░░░░░░░██                               │        ║
║  │ G2:     ██░░░░░░░░░░░░░░██                             │        ║
║  │ G3:       ██░░░░░░░░░░░░░░██                           │        ║
║  │ ...                                                      │        ║
║  │ → Khi G0 waiting → G1 nhảy vào Core!                  │        ║
║  │ → Khi G1 waiting → G2 nhảy vào Core!                  │        ║
║  │ → Core KHÔNG BAO GIỜ IDLE! = 87% nhanh hơn!          │        ║
║  └──────────────────────────────────────────────────────────┘        ║
║                                                                       ║
║  ═══ THỬ 2: Concurrency CÓ Parallelism (8 Cores) ═══              ║
║                                                                       ║
║  $ GOGC=off go test -cpu 8 -bench .                                  ║
║                                                                       ║
║  BenchmarkSequential-8   1,490,947,198 ns/op                         ║
║  BenchmarkConcurrent-8     187,382,200 ns/op  ← ~88% NHANH HƠN     ║
║                                                                       ║
║  ┌──────────────────────────────────────────────────────────┐        ║
║  │ CHÚ Ý: 88% vs 87% = GẦN NHƯ KHÔNG KHÁC BIỆT!         │        ║
║  │                                                          │        ║
║  │ → Thêm 7 Core nữa = KHÔNG cải thiện đáng kể!         │        ║
║  │ → Vì bottleneck là IO (disk read 1ms), KHÔNG phải CPU! │        ║
║  │ → Thêm Core = thêm throughput CPU,                     │        ║
║  │   NHƯNG work đang bị block bởi IO!                    │        ║
║  │ → = PARALLELISM KHÔNG GIÚP ÍCH CHO IO-BOUND!         │        ║
║  └──────────────────────────────────────────────────────────┘        ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

## §7. Tổng Kết & Câu Hỏi Phỏng Vấn

### Sơ đồ quyết định: Khi nào dùng Concurrency?

```
    ┌──────────────────────────────────────────────────────────────┐
    │                 CÂY QUYẾT ĐỊNH CONCURRENCY                  │
    │                                                              │
    │  Bắt đầu: Bạn có 1 bài toán cần tối ưu tốc độ           │
    │     │                                                        │
    │     ▼                                                        │
    │  ┌─────────────────────────────────────┐                    │
    │  │ "Out of order" execution có         │                    │
    │  │ KHẢ THI không?                       │                    │
    │  └──────┬──────────────┬───────────────┘                    │
    │     CÓ  │              │  KHÔNG                              │
    │         ▼              ▼                                     │
    │  ┌──────────┐   ┌──────────────────┐                        │
    │  │ Tiếp!    │   │ GIỮ SEQUENTIAL! │                        │
    │  └────┬─────┘   │ (Bubble Sort)   │                        │
    │       │         └──────────────────┘                        │
    │       ▼                                                      │
    │  ┌─────────────────────────────────────┐                    │
    │  │ Workload loại gì?                    │                    │
    │  └──────┬──────────────┬───────────────┘                    │
    │  CPU-   │              │  IO-Bound                           │
    │  Bound  │              │                                     │
    │         ▼              ▼                                     │
    │  ┌──────────────┐ ┌──────────────────────┐                  │
    │  │ CÓ nhiều     │ │ Dùng concurrency!    │                  │
    │  │ Core không?   │ │ KHÔNG cần thêm Core!│                  │
    │  └───┬──────┬───┘ │ 1 Core đã đủ!       │                  │
    │  CÓ  │  KHÔNG│    │ ~87-88% faster! ✅   │                  │
    │      ▼      ▼     └──────────────────────┘                  │
    │  ┌──────┐ ┌────────────────┐                                │
    │  │ Dùng │ │GIỮ SEQUENTIAL!│                                │
    │  │conc +│ │Context switch  │                                │
    │  │para! │ │= overhead      │                                │
    │  │~43%  │ │= 10-13% chậm! │                                │
    │  │fast! │ └────────────────┘                                │
    │  └──────┘                                                    │
    └──────────────────────────────────────────────────────────────┘
```

### Bảng tổng kết 3 ví dụ

```
    ┌─────────────┬────────────┬─────────────┬─────────────────────┐
    │ Ví dụ      │ Workload   │ 1 Core      │ 8 Cores             │
    ├─────────────┼────────────┼─────────────┼─────────────────────┤
    │ add         │ CPU-Bound  │ Seq +10%    │ Conc +43% ✅        │
    │ (cộng số)  │            │ (conc chậm!)│ (parallelism WIN!)  │
    ├─────────────┼────────────┼─────────────┼─────────────────────┤
    │ bubbleSort  │ CPU-Bound  │ ❌ Vô dụng │ ❌ Vô dụng         │
    │ (sắp xếp) │            │ (phải sort  │ (phải sort          │
    │             │            │  lại 100%)  │  lại 100%)          │
    ├─────────────┼────────────┼─────────────┼─────────────────────┤
    │ find        │ IO-Bound   │ Conc +87% ✅│ Conc +88%           │
    │ (đọc file) │            │ (1 Core đủ!)│ (thêm Core ≈ 0!)  │
    └─────────────┴────────────┴─────────────┴─────────────────────┘

    KẾT LUẬN CỐT LÕI:
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  1. CPU-Bound: CẦN PARALLELISM → cần nhiều Core!          │
    │     → 1 G per Core = tối ưu!                               │
    │     → Nhiều G hơn số Core = CHẬM hơn!                     │
    │                                                              │
    │  2. IO-Bound: KHÔNG CẦN PARALLELISM → 1 Core đã đủ!     │
    │     → Nhiều G chia sẻ 1 Core = TỐT!                      │
    │     → Thêm Core = KHÔNG cải thiện đáng kể!               │
    │                                                              │
    │  3. KHÔNG PHẢI LÚC NÀO concurrency cũng tốt!             │
    │     → Bubble Sort = concurrency VÔ DỤNG!                   │
    │     → Chi phí chia + ghép > lợi ích → GIỮ sequential!    │
    │                                                              │
    │  4. LUÔN viết sequential trước, sau đó MỚI hỏi:          │
    │     "Concurrency có thêm đủ VALUE không?"                  │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Câu hỏi phỏng vấn Senior Golang

```
╔═══════════════════════════════════════════════════════════════════════╗
║                   CÂU HỎI PHỎNG VẤN SENIOR GOLANG                    ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  Q1: Concurrency và Parallelism khác nhau thế nào?                   ║
║  A1: Concurrency = "out of order" execution (1 Core đủ).            ║
║      Parallelism = "at the same time" (cần ≥2 Cores).              ║
║      Concurrency là về CẤU TRÚC, Parallelism là về THỰC THI.       ║
║                                                                       ║
║  Q2: Khi nào concurrency làm chương trình CHẬM hơn?                 ║
║  A2: CPU-Bound workload + chỉ 1 Core! Vì context switch           ║
║      = overhead thuần (Stop The World), không có waiting             ║
║      tự nhiên để tận dụng. Kết quả chậm 10-13%.                    ║
║                                                                       ║
║  Q3: Tại sao IO-Bound không cần parallelism?                        ║
║  A3: Vì Goroutines TỰ NHIÊN waiting (disk, network).               ║
║      Khi G1 waiting → G2 dùng Core. Context switch = FREE          ║
║      vì G đã dừng rồi! 1 Core + 8 G = 87% faster.                 ║
║      Thêm 7 Core nữa = chỉ thêm 1% = không đáng!                 ║
║                                                                       ║
║  Q4: Làm sao xác định workload là CPU hay IO bound?                 ║
║  A4: Hỏi: "Goroutine có TỰ NHIÊN vào Waiting state không?"        ║
║      → CÓ (IO, network, mutex) = IO-Bound                          ║
║      → KHÔNG (tính toán liên tục) = CPU-Bound                      ║
║                                                                       ║
║  Q5: Bao nhiêu Goroutines là tối ưu?                                ║
║  A5: CPU-Bound: = runtime.NumCPU() (1 G per Core)                   ║
║      IO-Bound: > runtime.NumCPU() OK! Nhiều hơn = tốt hơn         ║
║      (đến 1 ngưỡng nào đó, phụ thuộc vào latency IO).             ║
║                                                                       ║
║  Q6: Worker Pool pattern dùng khi nào?                               ║
║  A6: Khi cần control số lượng Goroutines.                           ║
║      Dùng buffered channel + N workers.                              ║
║      Channel tự phân phối work = load balancing tự nhiên.           ║
║                                                                       ║
║  Q7: Tại sao Bubble Sort không phù hợp concurrency?                 ║
║  A7: Vì chi phí GỘP KẾT QUẢ quá cao! Sort từng phần xong        ║
║      vẫn phải sort lại toàn bộ = HỦY toàn bộ lợi ích            ║
║      concurrent. Concurrency chỉ hợp khi chia + ghép RẺ.         ║
║                                                                       ║
║  Q8: "Go turns IO-Bound into CPU-Bound" nghĩa là gì?               ║
║  A8: Từ góc OS: M (OS Thread) luôn BUSY vì Go runtime             ║
║      liên tục switch G trên cùng M. OS không thấy waiting.         ║
║      → OS giữ M trên Core (no OS ctx switch).                      ║
║      → Maximize throughput trên mỗi Core!                          ║
║                                                                       ║
║  Q9: sync.WaitGroup và atomic.AddInt64 dùng thế nào?                ║
║  A9: WaitGroup: đợi tất cả G xong (wg.Add → wg.Done → wg.Wait)  ║
║      atomic: cộng an toàn từ nhiều G vào 1 biến shared            ║
║      (tránh race condition, KHÔNG CẦN mutex!)                      ║
║                                                                       ║
║  Q10: Khi nào KHÔNG nên dùng concurrency?                           ║
║  A10: 1. Khi sequential đã đủ nhanh                                 ║
║       2. Khi complexity cost > performance gain                      ║
║       3. Khi chia/ghép kết quả quá đắt (Bubble Sort)              ║
║       4. Khi out-of-order execution KHÔNG khả thi                   ║
║       5. Khi chỉ có 1 Core + CPU-Bound workload                    ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

> **Series hoàn chỉnh:**
>
> - Part I: [OS Scheduler](Scheduling-In-Go-Part1-OS-Scheduler-Deep-Dive.md) — Cơ chế OS Thread, Context Switch, Preemptive Scheduling
> - Part II: [Go Scheduler](Scheduling-In-Go-Part2-Go-Scheduler-Deep-Dive.md) — G-M-P Model, Work Stealing, Network Poller
> - **Part III: Concurrency** ← BẠN VỪA ĐỌC XONG
