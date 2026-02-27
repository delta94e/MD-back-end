# Language Mechanics On Memory Profiling — Cơ Chế Bộ Nhớ & Profiling

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Language Mechanics On Memory Profiling" của William Kennedy (2017)
> **Góc nhìn:** Escape Analysis, Heap Allocations, pprof — hiểu bộ nhớ Go ở mức sâu nhất
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                | Mô tả                                   |
| --- | ------------------------------------- | --------------------------------------- |
| §1  | Stack vs Heap — Ôn lại nền tảng       | Value ở đâu và tại sao                  |
| §2  | Escape Analysis — Compiler quyết định | Go compiler quyết định Stack hay Heap   |
| §3  | Bài toán thực tế: algOne              | Tìm "elvis" → thay "Elvis" trong stream |
| §4  | Benchmarking — Đo hiệu năng           | go test -bench -benchmem                |
| §5  | Profiling với pprof                   | Tìm chính xác dòng nào allocate         |
| §6  | Interface gây Escape                  | Chi phí ẩn của interface                |
| §7  | Stack Frame & Compile-time size       | Tại sao make([]byte, size) escape       |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior   | Ôn tập & thực hành                      |

---

## §1. Stack vs Heap — Ôn Lại Nền Tảng

### 1.1 Hai vùng nhớ chính

```
╔═══════════════════════════════════════════════════════════════╗
║   STACK vs HEAP — HAI VÙNG NHỚ CHÍNH                         ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  STACK:                                                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ┌──────────────────────┐  ← Stack Frame #3  │             ║
║  │ │ local vars of func C │     (func C)        │             ║
║  │ ├──────────────────────┤  ← Stack Frame #2  │             ║
║  │ │ local vars of func B │     (func B)        │             ║
║  │ ├──────────────────────┤  ← Stack Frame #1  │             ║
║  │ │ local vars of func A │     (func A)        │             ║
║  │ └──────────────────────┘                     │             ║
║  │                                              │             ║
║  │ ✅ CỰC NHANH (CPU cache friendly!)          │             ║
║  │ ✅ Tự dọn dẹp khi function return!          │             ║
║  │ ✅ KHÔNG cần Garbage Collector!              │             ║
║  │ ❌ Kích thước CỐ ĐỊNH tại compile time!    │             ║
║  │ ❌ Chỉ sống trong scope của function!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  HEAP:                                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ┌────┐ ┌────────┐ ┌──────┐ ┌────┐          │             ║
║  │ │ v1 │ │  v2    │ │  v3  │ │ v4 │          │             ║
║  │ └────┘ └────────┘ └──────┘ └────┘          │             ║
║  │ (rải rác, không theo thứ tự)                │             ║
║  │                                              │             ║
║  │ ✅ Chia sẻ giữa goroutines/functions!       │             ║
║  │ ✅ Kích thước LINH HOẠT!                    │             ║
║  │ ❌ CHẬM hơn Stack!                          │             ║
║  │ ❌ CẦN Garbage Collector để dọn!            │             ║
║  │ ❌ GC pressure = latency!                    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  MỤC TIÊU:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → GIỮ values trên Stack NHIỀU NHẤT có thể! │             ║
║  │ → GIẢM allocations trên Heap!                │             ║
║  │ → ÍT Heap = ÍT GC pressure = NHANH HƠN!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Escape Analysis — Compiler Quyết Định

### 2.1 Escape Analysis là gì?

```
╔═══════════════════════════════════════════════════════════════╗
║   ESCAPE ANALYSIS — COMPILER QUYẾT ĐỊNH                      ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ESCAPE ANALYSIS = quá trình COMPILER phân tích              ║
║  để quyết định value nên ở STACK hay HEAP!                   ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Compiler hỏi: "Value này có thể AN TOÀN    │             ║
║  │ sống trên Stack không?"                      │             ║
║  │                                              │             ║
║  │     CÓ → Đặt trên STACK ✅                  │             ║
║  │  KHÔNG → ESCAPE lên HEAP ❌                  │             ║
║  │                                              │             ║
║  │ KHI NÀO VALUE ESCAPE LÊN HEAP?              │             ║
║  │                                              │             ║
║  │ 1. Value được SHARE lên caller               │             ║
║  │    func create() *User {                      │             ║
║  │        u := User{Name: "A"}                   │             ║
║  │        return &u  // ← ESCAPE! Share lên!   │             ║
║  │    }                                          │             ║
║  │                                              │             ║
║  │ 2. Value gán vào INTERFACE                   │             ║
║  │    var r io.Reader = &buf // ← ESCAPE!       │             ║
║  │    // Compiler không biết concrete type lúc  │             ║
║  │    // compile → không biết size → HEAP!      │             ║
║  │                                              │             ║
║  │ 3. Size KHÔNG BIẾT tại compile time          │             ║
║  │    buf := make([]byte, n) // ← ESCAPE!       │             ║
║  │    // n không biết lúc compile!               │             ║
║  │                                              │             ║
║  │ 4. Value quá LỚN cho Stack                   │             ║
║  │    data := make([]byte, 10*1024*1024) // 10MB │             ║
║  │                                              │             ║
║  │ 5. Closure capture variable                   │             ║
║  │    go func() { fmt.Println(x) } // x ESCAPE! │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  XEM COMPILER REPORT:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ $ go build -gcflags "-m -m"                   │             ║
║  │                                              │             ║
║  │ → Cho biết CHÍNH XÁC giá trị nào escape!   │             ║
║  │ → Cho biết TẠI SAO escape!                  │             ║
║  │ → -m = report, -m -m = chi tiết hơn!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Bài Toán Thực Tế: algOne

### 3.1 Tìm "elvis" → thay "Elvis"

```
╔═══════════════════════════════════════════════════════════════╗
║   BÀI TOÁN: TÌM "elvis" → THAY "Elvis"                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  INPUT:                                                        ║
║  "abcelvisaElvisabcelviseelvisaelvisaab..."                   ║
║                                                               ║
║  OUTPUT:                                                       ║
║  "abcElvisaElvisabcElviseElvisaElvisaab..."                   ║
║                                                               ║
║  → Tìm "elvis" (viết thường)                                ║
║  → Thay bằng "Elvis" (viết hoa)                              ║
║  → Xử lý qua STREAM (byte-by-byte)!                         ║
╚═══════════════════════════════════════════════════════════════╝
```

### 3.2 Code algOne (phiên bản gốc)

```go
// ═══ algOne — PHIÊN BẢN GỐC ═══

func algOne(data []byte, find []byte, repl []byte, output *bytes.Buffer) {

    // Tạo bytes.Buffer từ data → cung cấp stream
    input := bytes.NewBuffer(data)  // ← ALLOCATION #1!

    // Kích thước pattern cần tìm
    size := len(find)  // = 5 (cho "elvis")

    // Tạo buffer để đọc stream
    buf := make([]byte, size)  // ← ALLOCATION #2!
    end := size - 1            // = 4

    // Đọc 4 bytes đầu tiên
    if n, err := io.ReadFull(input, buf[:end]); err != nil {
        output.Write(buf[:n])
        return
    }

    for {
        // Đọc thêm 1 byte → buf đầy 5 bytes
        if _, err := io.ReadFull(input, buf[end:]); err != nil {
            output.Write(buf[:end])
            return
        }

        // So sánh buf với "elvis"
        if bytes.Compare(buf, find) == 0 {
            output.Write(repl)  // Ghi "Elvis"!

            // Đọc 4 bytes mới
            if n, err := io.ReadFull(input, buf[:end]); err != nil {
                output.Write(buf[:n])
                return
            }
            continue
        }

        // Không match → ghi byte đầu tiên
        output.WriteByte(buf[0])

        // Dịch buffer sang trái 1 vị trí
        copy(buf, buf[1:])
    }
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   SLIDING WINDOW — CÁCH algOne HOẠT ĐỘNG                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Input: "abcelvisxyz..."                                      ║
║  Find:  "elvis" (5 bytes)                                     ║
║                                                               ║
║  Ban đầu: đọc 4 bytes                                        ║
║  buf: [a][b][c][e][_]                                         ║
║                    ↑ chưa đọc                                ║
║                                                               ║
║  Vòng lặp: đọc thêm 1 byte                                  ║
║  buf: [a][b][c][e][l]  → "abcel" ≠ "elvis" → ghi 'a'       ║
║                                                               ║
║  Dịch trái + đọc thêm 1:                                    ║
║  buf: [b][c][e][l][v]  → "bcelv" ≠ "elvis" → ghi 'b'       ║
║                                                               ║
║  Dịch trái + đọc thêm 1:                                    ║
║  buf: [c][e][l][v][i]  → "celvi" ≠ "elvis" → ghi 'c'       ║
║                                                               ║
║  Dịch trái + đọc thêm 1:                                    ║
║  buf: [e][l][v][i][s]  → "elvis" == "elvis" → GHI "Elvis"!  ║
║                                                               ║
║  → Sliding window 5 bytes trượt qua stream!                 ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Benchmarking — Đo Hiệu Năng

### 4.1 Viết benchmark

```go
// ═══ BENCHMARK FUNCTION ═══

func BenchmarkAlgorithmOne(b *testing.B) {
    var output bytes.Buffer
    in := assembleInputStream()
    find := []byte("elvis")
    repl := []byte("Elvis")

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        output.Reset()
        algOne(in, find, repl, &output)
    }
}
```

### 4.2 Chạy benchmark

```
╔═══════════════════════════════════════════════════════════════╗
║   CHẠY BENCHMARK — go test -bench                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  LỆNH:                                                        ║
║  $ go test -run none -bench AlgorithmOne                     ║
║           -benchtime 3s -benchmem                             ║
║                                                               ║
║  KẾT QUẢ:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ BenchmarkAlgorithmOne-8                       │             ║
║  │    2000000       2522 ns/op                   │             ║
║  │    117 B/op      2 allocs/op                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GIẢI NGHĨA:                                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ -8          = 8 CPU cores                     │             ║
║  │ 2000000     = chạy 2 triệu lần             │             ║
║  │ 2522 ns/op  = 2522 nanoseconds/operation     │             ║
║  │ 117 B/op    = 117 bytes allocated/operation   │             ║
║  │ 2 allocs/op = 2 heap allocations/operation   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  FLAG GIẢI THÍCH:                                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ -run none     = không chạy tests, chỉ bench │             ║
║  │ -bench X      = chạy benchmark match regex X │             ║
║  │ -benchtime 3s = chạy tối thiểu 3 giây      │             ║
║  │ -benchmem     = hiển thị memory allocations  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CÂU HỎI: 2 allocations ở đâu? 117 bytes là gì?            ║
║  → Cần PROFILING để tìm ra!                                  ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Profiling Với pprof

### 5.1 Tạo và phân tích profile data

```
╔═══════════════════════════════════════════════════════════════╗
║   PROFILING VỚI pprof — TÌM CHÍNH XÁC DÒNG NÀO ALLOCATE   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  BƯỚC 1: Tạo memory profile                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ $ go test -run none -bench AlgorithmOne      │             ║
║  │          -benchtime 3s -benchmem             │             ║
║  │          -memprofile mem.out                  │             ║
║  │                       ↑ FILE PROFILE MỚI!    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  → Tạo 2 file mới:                                           ║
║    mem.out       = profile data                               ║
║    memcpu.test   = test binary (chứa symbols)                ║
║                                                               ║
║  BƯỚC 2: Phân tích với pprof                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ $ go tool pprof -alloc_space                  │             ║
║  │                  memcpu.test mem.out          │             ║
║  │                                              │             ║
║  │ -alloc_space = xem MỌI allocation            │             ║
║  │ (không chỉ những cái đang dùng!)            │             ║
║  │                                              │             ║
║  │ Mặc định: -inuse_space (chỉ đang dùng)     │             ║
║  │ → Dùng -alloc_space để tìm "low hanging     │             ║
║  │   fruit"!                                    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BƯỚC 3: list command — xem từng dòng!                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ (pprof) list algOne                           │             ║
║  │ Total: 335.03MB                               │             ║
║  │                                              │             ║
║  │  335.03MB  335.03MB  (flat, cum) 100%        │             ║
║  │       .         .    80: func algOne(...)     │             ║
║  │       .         .    82: // Use bytes.Buffer  │             ║
║  │ 318.53MB  318.53MB   83: input := bytes.      │             ║
║  │                          NewBuffer(data)      │             ║
║  │                          ↑ ALLOCATION #1!     │             ║
║  │                          318.53 MB!           │             ║
║  │       .         .    86: size := len(find)    │             ║
║  │  16.50MB   16.50MB   89: buf := make([]byte,  │             ║
║  │                          size)                │             ║
║  │                          ↑ ALLOCATION #2!     │             ║
║  │                          16.50 MB!            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  FLAT vs CUM (cột 1 vs cột 2):                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ FLAT (cột 1) = allocation TRỰC TIẾP trong   │             ║
║  │ function đó!                                  │             ║
║  │ → "Function NÀY đã allocate!"              │             ║
║  │                                              │             ║
║  │ CUM (cột 2) = tổng allocation bao gồm cả   │             ║
║  │ calls bên trong!                              │             ║
║  │ → "Function này + tất cả calls bên trong"  │             ║
║  │                                              │             ║
║  │ Benchmark function: flat=0, cum=335MB        │             ║
║  │ → Benchmark KHÔNG allocate trực tiếp!       │             ║
║  │ → Tất cả từ algOne bên trong vòng lặp!    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Interface Gây Escape

### 6.1 Chi phí ẩn của interface

```
╔═══════════════════════════════════════════════════════════════╗
║   INTERFACE GÂY ESCAPE — CHI PHÍ ẨN!                        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  TẠI SAO bytes.Buffer ESCAPE?                                 ║
║                                                               ║
║  BƯỚC 1: Compiler INLINE bytes.NewBuffer:                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Code viết:                                    │             ║
║  │   input := bytes.NewBuffer(data)              │             ║
║  │                                              │             ║
║  │ Compiler chuyển thành (inlining):            │             ║
║  │   input := &bytes.Buffer{buf: data}          │             ║
║  │                                              │             ║
║  │ → bytes.NewBuffer KHÔNG ĐƯỢC GỌI!           │             ║
║  │ → Code bên trong được COPY vào algOne!      │             ║
║  │ → Buffer được tạo TRỰC TIẾP trong algOne!  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BƯỚC 2: Tại sao vẫn ESCAPE?                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Compiler report (-gcflags "-m -m"):           │             ║
║  │                                              │             ║
║  │ &bytes.Buffer literal escapes to heap        │             ║
║  │   from input (interface-converted) at :93    │             ║
║  │   from input (passed to call) at :93          │             ║
║  │                                              │             ║
║  │ → Dòng 93: io.ReadFull(input, buf[:end])     │             ║
║  │ → input gán vào INTERFACE!                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BƯỚC 3: io.ReadFull nhận INTERFACE!                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ type Reader interface {                       │             ║
║  │     Read(p []byte) (n int, err error)         │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ func ReadFull(r Reader, buf []byte) (int,error)│            ║
║  │                ↑ INTERFACE!                  │             ║
║  │                                              │             ║
║  │ io.ReadFull(input, buf[:end])                 │             ║
║  │             ↑ input (*bytes.Buffer)          │             ║
║  │             → gán vào Reader interface       │             ║
║  │             → ESCAPE LÊN HEAP!               │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ ESCAPE QUA INTERFACE:                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ algOne() stack frame:                         │             ║
║  │ ┌──────────────┐                             │             ║
║  │ │ input (ptr) ─┼───→ ???                     │             ║
║  │ └──────────────┘                             │             ║
║  │                                              │             ║
║  │ io.ReadFull() cần io.Reader interface:        │             ║
║  │ ┌──────────────┐                             │             ║
║  │ │ Reader iface │                             │             ║
║  │ │ ┌──────────┐ │                             │             ║
║  │ │ │ type ptr │─┼───→ *bytes.Buffer type info │             ║
║  │ │ │ data ptr │─┼───→ bytes.Buffer VALUE     │             ║
║  │ │ └──────────┘ │       (phải ở HEAP!)       │             ║
║  │ └──────────────┘                             │             ║
║  │                                              │             ║
║  │ → Interface chứa pointer tới value          │             ║
║  │ → Compiler không biết value sẽ đi đâu      │             ║
║  │ → AN TOÀN = đặt trên HEAP!                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  FIX: GỌI method TRỰC TIẾP, không qua interface!            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ❌ TRƯỚC (qua interface → escape!):          │             ║
║  │   io.ReadFull(input, buf[:end])               │             ║
║  │   → input gán vào io.Reader → HEAP!         │             ║
║  │                                              │             ║
║  │ ✅ SAU (gọi trực tiếp → no escape!):        │             ║
║  │   input.Read(buf[:end])                       │             ║
║  │   → Gọi Read() trực tiếp trên *bytes.Buffer│             ║
║  │   → KHÔNG cần interface!                     │             ║
║  │   → input ở lại STACK!                       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 6.2 Khi nào NÊN dùng interface?

```
╔═══════════════════════════════════════════════════════════════╗
║   INTERFACE: KHI NÀO NÊN vs KHÔNG NÊN                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Kennedy: "There is a COST to using an interface:            ║
║  ALLOCATION and INDIRECTION."                                ║
║                                                               ║
║  ✅ NÊN dùng interface khi:                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. User cần cung cấp implementation          │             ║
║  │    → API nhận io.Reader từ caller            │             ║
║  │                                              │             ║
║  │ 2. API có NHIỀU implementations bên trong    │             ║
║  │    → Database driver: MySQL, Postgres, etc.  │             ║
║  │                                              │             ║
║  │ 3. Phần code CÓ THỂ THAY ĐỔI cần decouple │             ║
║  │    → Strategy pattern, dependency injection  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ❌ KHÔNG NÊN dùng interface khi:                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Dùng "cho có" — không có lý do cụ thể   │             ║
║  │                                              │             ║
║  │ 2. Để "generalize" algorithm                 │             ║
║  │    → Trong algOne, io.ReadFull không cần!    │             ║
║  │    → input.Read() trực tiếp = đủ!           │             ║
║  │                                              │             ║
║  │ 3. Khi user có thể TỰ declare interface     │             ║
║  │    → Go implicit interfaces!                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Stack Frame & Compile-Time Size

### 7.1 Tại sao make([]byte, size) escape?

```
╔═══════════════════════════════════════════════════════════════╗
║   STACK FRAME & COMPILE-TIME SIZE                             ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  SAU KHI FIX interface, còn 1 allocation:                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ $ go test -bench AlgorithmOne -benchmem       │             ║
║  │ 2000000    1814 ns/op    5 B/op   1 allocs/op │             ║
║  │                                              │             ║
║  │ 1 allocation còn lại = buf!                   │             ║
║  │ Line 89: buf := make([]byte, size)            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  COMPILER REPORT:                                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ $ go build -gcflags "-m -m"                   │             ║
║  │                                              │             ║
║  │ ./stream.go:89: make([]byte, size)            │             ║
║  │   escapes to heap                             │             ║
║  │   from make([]byte, size)                     │             ║
║  │   (TOO LARGE FOR STACK)                       │             ║
║  │        ↑ MISLEADING!                         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  THỰC TẾ:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Too large for stack" = MISLEADING!           │             ║
║  │                                              │             ║
║  │ Vấn đề KHÔNG PhẢI là "quá lớn"             │             ║
║  │ Vấn đề LÀ: Compiler KHÔNG BIẾT size         │             ║
║  │ lúc COMPILE TIME!                             │             ║
║  │                                              │             ║
║  │ size = len(find) → runtime value!            │             ║
║  │ → Compiler không biết size = bao nhiêu!      │             ║
║  │ → Không thể tính stack frame size!           │             ║
║  │ → PHẢI đặt trên HEAP!                       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  QUY TẮC:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Values CHỈ ở Stack khi compiler BIẾT SIZE   │             ║
║  │ tại COMPILE TIME!                             │             ║
║  │                                              │             ║
║  │ → Stack frame size = tính lúc compile!        │             ║
║  │ → Không biết size → không tính được frame  │             ║
║  │ → PHẢI đặt trên HEAP!                       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CHỨNG MINH: Hardcode size = 5                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ buf := make([]byte, 5)  // ← SIZE CỐ ĐỊNH! │             ║
║  │                                              │             ║
║  │ Kết quả benchmark:                            │             ║
║  │ 3000000    1720 ns/op    0 B/op   0 allocs!  │             ║
║  │                                              │             ║
║  │ → 0 allocations! HOÀN TOÀN trên Stack!       │             ║
║  │ → Nhưng không thể hardcode trong thực tế!   │             ║
║  │ → Phải chấp nhận 1 allocation này!           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 7.2 Kết quả tối ưu qua 3 bước

```
╔═══════════════════════════════════════════════════════════════╗
║   KẾT QUẢ TỐI ƯU QUA 3 BƯỚC                                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  BƯỚC 1: Ban đầu (chưa tối ưu)                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 2000000    2522 ns/op   117 B/op  2 allocs   │             ║
║  │                                              │             ║
║  │ Allocation #1: bytes.Buffer (112 B)          │             ║
║  │ Allocation #2: buf slice backing array (5 B) │             ║
║  └──────────────────────────────────────────────┘             ║
║            │                                                   ║
║            ▼  Đổi io.ReadFull → input.Read()                 ║
║                                                               ║
║  BƯỚC 2: Bỏ interface allocation                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 2000000    1814 ns/op     5 B/op  1 allocs   │             ║
║  │                                              │             ║
║  │ ↑ ~29% NHANH HƠN!                           │             ║
║  │ Allocation #1: BIẾN MẤT! (-112 B!)          │             ║
║  │ Allocation #2: buf slice vẫn còn (5 B)      │             ║
║  └──────────────────────────────────────────────┘             ║
║            │                                                   ║
║            ▼  Hardcode make([]byte, 5)                        ║
║                                                               ║
║  BƯỚC 3: Bỏ tất cả allocations                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 3000000    1720 ns/op     0 B/op  0 allocs   │             ║
║  │                                              │             ║
║  │ ↑ ~33% NHANH HƠN so với bước 1!             │             ║
║  │ ZERO allocations! Mọi thứ trên Stack!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 2522 ns → 1814 ns → 1720 ns                 │             ║
║  │ 117 B   →    5 B  →    0 B                  │             ║
║  │ 2 alloc →  1 alloc→  0 alloc                │             ║
║  │                                              │             ║
║  │ → Giảm allocations = TĂNG ĐÁNG KỂ hiệu năng│             ║
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
║  Q1: "Escape Analysis là gì? Tại sao quan trọng?"           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = Quá trình compiler phân tích value nên     │             ║
║  │ ở Stack hay Heap!                             │             ║
║  │                                              │             ║
║  │ Quan trọng vì:                                │             ║
║  │ → Stack = NHANH, tự dọn, không GC           │             ║
║  │ → Heap = CHẬM, cần GC, có latency          │             ║
║  │ → Hiểu escape analysis = viết code hiệu năng│             ║
║  │                                              │             ║
║  │ Xem report: go build -gcflags "-m -m"        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Kể các trường hợp value escape lên Heap?"             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Return pointer lên caller (share up)      │             ║
║  │ 2. Gán value vào INTERFACE → escape!         │             ║
║  │ 3. make() với SIZE không biết compile time   │             ║
║  │ 4. Closure capture biến ngoài               │             ║
║  │ 5. Value quá lớn cho stack                   │             ║
║  │ 6. Gửi pointer vào channel                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Interface có chi phí gì?"                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. ALLOCATION: value gán vào interface      │             ║
║  │    có thể escape lên Heap!                    │             ║
║  │ 2. INDIRECTION: gọi method qua interface    │             ║
║  │    = indirect call (không inline được!)       │             ║
║  │                                              │             ║
║  │ → Không rõ interface giúp code TỐT HƠN?     │             ║
║  │ → ĐỪNG dùng!                                 │             ║
║  │ → Gọi method trực tiếp trên concrete type!  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Làm sao tối ưu memory allocations?"                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. go test -bench -benchmem → đo!           │             ║
║  │ 2. -memprofile → tạo profile data            │             ║
║  │ 3. go tool pprof -alloc_space → tìm dòng!  │             ║
║  │ 4. go build -gcflags "-m -m" → tìm LÝ DO!  │             ║
║  │ 5. Refactor:                                  │             ║
║  │    → Bỏ interface không cần thiết           │             ║
║  │    → Dùng compile-time known sizes           │             ║
║  │    → Tránh share up (return pointer)          │             ║
║  │                                              │             ║
║  │ Kennedy: "NEVER write code with performance  │             ║
║  │ as your FIRST priority. Optimize for         │             ║
║  │ CORRECTNESS first. Then profile."            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: MEMORY PROFILING                             │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Escape Analysis = compiler quyết định │            │
    │  │    Stack hay Heap!                         │            │
    │  │                                          │            │
    │  │ 2. Interface = có CHI PHÍ!                │            │
    │  │    → Allocation + Indirection             │            │
    │  │    → Chỉ dùng khi CÓ LÝ DO cụ thể!     │            │
    │  │                                          │            │
    │  │ 3. Compile-time unknown size = HEAP!      │            │
    │  │    → make([]byte, n) khi n là runtime    │            │
    │  │    → make([]byte, 5) = Stack!             │            │
    │  │                                          │            │
    │  │ 4. Quy trình tối ưu:                      │            │
    │  │    Benchmark → Profile → Compiler Report │            │
    │  │    → Refactor → Benchmark lại!           │            │
    │  │                                          │            │
    │  │ 5. CORRECTNESS trước, PERFORMANCE sau!    │            │
    │  │    "Never guess about performance"         │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "You are not going to write ZERO-ALLOCATION            │
    │   software but you want to MINIMIZE allocations          │
    │   when possible."                                        │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
