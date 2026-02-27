# Scheduling In Go: Part II — Go Scheduler (Bộ Lập Lịch Go)

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** William Kennedy — "Scheduling In Go: Part II - Go Scheduler" (Ardan Labs)
> **Góc nhìn:** G-M-P Model, Cooperating Scheduler, Network Poller, Work Stealing
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                | Mô tả                                     |
| --- | ------------------------------------- | ----------------------------------------- |
| §1  | G-M-P Model                           | Goroutine, Machine (OS Thread), Processor |
| §2  | Cooperating Scheduler                 | User-space scheduling, safe points        |
| §3  | Goroutine States                      | Waiting, Runnable, Executing              |
| §4  | Context Switching Events              | go keyword, GC, syscalls, sync            |
| §5  | Async System Calls — Network Poller   | kqueue/epoll/iocp, non-blocking IO        |
| §6  | Sync System Calls — M Detach          | File IO, CGO, M1 detach + M2 replace      |
| §7  | Work Stealing                         | LRQ steal half, GRQ fallback              |
| §8  | Goroutine vs Thread — So Sánh Thực Tế | ~200ns vs ~1000ns, IO→CPU-bound trick     |
| §9  | Tổng kết & Câu hỏi phỏng vấn          | Ôn tập & thực hành                        |

---

## §1. G-M-P Model — Ba Thành Phần Cốt Lõi

```
╔═══════════════════════════════════════════════════════════════╗
║   G — M — P: TIM MẠCH CỦA GO SCHEDULER!                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  G = GOROUTINE:                                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → = "Application-level thread"                │             ║
║  │ → Giống OS Thread nhưng NHẸHƠN nhiều!       │             ║
║  │ → Stack khởi tạo chỉ ~2KB (Thread = ~1MB!) │             ║
║  │ → Quản lý bởi Go runtime, KHÔNG phải OS!   │             ║
║  │ → Tạo bằng: go myFunc()                     │             ║
║  │ → Goroutine = Coroutine nhưng đổi C→G!     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  M = MACHINE (OS Thread):                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → = OS Thread thật sự!                       │             ║
║  │ → Quản lý bởi HỆ ĐIỀU HÀNH!                │             ║
║  │ → OS đặt M lên Core để thực thi!           │             ║
║  │ → Goroutine chạy TRÊN M!                    │             ║
║  │ → M = "xe" chở G đi trên "đường" Core!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  P = PROCESSOR (Logical Processor):                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → = Bộ xử lý LOGIC (không phải CPU vật lý!)│             ║
║  │ → Mỗi virtual core = 1 P!                   │             ║
║  │ → runtime.NumCPU() = số P!                  │             ║
║  │ → Hyper-Threading: 4 cores × 2 = 8 P!       │             ║
║  │ → Mỗi P có 1 LRQ (Local Run Queue)!        │             ║
║  │ → P = "quản lý" phân công G cho M!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ G-M-P:                                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  GRQ (Global Run Queue)                       │             ║
║  │  ┌───┬───┐                                   │             ║
║  │  │ G │ G │  ← Chưa gán cho P nào!         │             ║
║  │  └───┴───┘                                   │             ║
║  │                                              │             ║
║  │  ┌─────────────────────────────────────┐     │             ║
║  │  │ P0                                   │     │             ║
║  │  │ ┌────┐                              │     │             ║
║  │  │ │ M0 │──→ Core 0                    │     │             ║
║  │  │ └────┘                              │     │             ║
║  │  │   ↑                                  │     │             ║
║  │  │  [G] ← đang executing trên M0      │     │             ║
║  │  │                                     │     │             ║
║  │  │ LRQ: [G][G][G] ← đợi thay phiên  │     │             ║
║  │  └─────────────────────────────────────┘     │             ║
║  │                                              │             ║
║  │  ┌─────────────────────────────────────┐     │             ║
║  │  │ P1                                   │     │             ║
║  │  │ ┌────┐                              │     │             ║
║  │  │ │ M1 │──→ Core 1                    │     │             ║
║  │  │ └────┘                              │     │             ║
║  │  │   ↑                                  │     │             ║
║  │  │  [G] ← đang executing trên M1      │     │             ║
║  │  │                                     │     │             ║
║  │  │ LRQ: [G][G] ← đợi thay phiên     │     │             ║
║  │  └─────────────────────────────────────┘     │             ║
║  │                                              │             ║
║  │  GIẢI THÍCH:                                  │             ║
║  │  → Mỗi P gắn 1 M (OS Thread)               │             ║
║  │  → M chạy trên 1 Core                       │             ║
║  │  → G context-switch trên M (KHÔNG phải Core!)│            ║
║  │  → LRQ: hàng đợi G riêng của mỗi P       │             ║
║  │  → GRQ: hàng đợi G chung, chưa gán P     │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  runtime.NumCPU() VÍ DỤ:                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ MacBook Pro i7: 4 cores × 2 HT = 8 P        │             ║
║  │ → 8 M (OS Threads) sẵn sàng!               │             ║
║  │ → 8 cores ảo chạy SONG SONG!               │             ║
║  │                                              │             ║
║  │ package main                                  │             ║
║  │ import ("fmt"; "runtime")                     │             ║
║  │ func main() {                                 │             ║
║  │     fmt.Println(runtime.NumCPU()) // 8        │             ║
║  │ }                                             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Cooperating Scheduler — Scheduler User-Space

```
╔═══════════════════════════════════════════════════════════════╗
║   GO SCHEDULER = COOPERATING, TRÔNG NHƯ PREEMPTIVE!          ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  OS Scheduler vs Go Scheduler:                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌──────────────┬──────────────────────┐     │             ║
║  │  │              │ OS Sched  │ Go Sched │     │             ║
║  │  ├──────────────┼──────────────────────┤     │             ║
║  │  │ Chạy ở      │ Kernel    │ User     │     │             ║
║  │  │              │ space     │ space    │     │             ║
║  │  ├──────────────┼──────────────────────┤     │             ║
║  │  │ Loại        │Preemptive │Cooperating│    │             ║
║  │  │              │(cưỡng chế)│(hợp tác) │    │             ║
║  │  ├──────────────┼──────────────────────┤     │             ║
║  │  │ Quản lý     │ OS Threads│ Goroutines│    │             ║
║  │  ├──────────────┼──────────────────────┤     │             ║
║  │  │ Nhúng vào   │ Kernel    │ App binary│    │             ║
║  │  │              │           │ (runtime) │    │             ║
║  │  ├──────────────┼──────────────────────┤     │             ║
║  │  │ Dự đoán    │ KHÔNG     │ KHÔNG     │    │             ║
║  │  │ được?       │           │           │    │             ║
║  │  └──────────────┴──────────────────────┘     │             ║
║  │                                              │             ║
║  │  KEY INSIGHT:                                 │             ║
║  │  → Go scheduler chạy TRONG app!              │             ║
║  │  → TRÊN kernel, DƯỚI code của bạn!          │             ║
║  │  → Cooperating = cần "safe points"!          │             ║
║  │  → NHƯNG trông + cảm giác = PREEMPTIVE!     │             ║
║  │  → Vì developer KHÔNG kiểm soát scheduing! │             ║
║  │  → Go runtime TỰ quyết định!               │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  KIẾN TRÚC TẦNG:                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌──────────────────────────────┐            │             ║
║  │  │ Your Go Code (Goroutines)    │ ← Bạn!   │             ║
║  │  ├──────────────────────────────┤            │             ║
║  │  │ Go Runtime (Go Scheduler)    │ ← G↔M!  │             ║
║  │  ├──────────────────────────────┤            │             ║
║  │  │ OS Kernel (OS Scheduler)     │ ← M↔Core!│             ║
║  │  ├──────────────────────────────┤            │             ║
║  │  │ Hardware (Cores)             │ ← Thực thi│             ║
║  │  └──────────────────────────────┘            │             ║
║  │                                              │             ║
║  │  → Go Scheduler: G context-switch trên M!   │             ║
║  │  → OS Scheduler: M context-switch trên Core!│             ║
║  │  → 2 TẦNG scheduling!                        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Goroutine States — Ba Trạng Thái

```
╔═══════════════════════════════════════════════════════════════╗
║   GOROUTINE CÓ 3 TRẠNG THÁI — GIỐNG OS THREAD!              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │        ┌────────────┐                        │             ║
║  │   ┌───→│ RUNNABLE   │←──────┐                │             ║
║  │   │    │ (muốn chạy)│       │                │             ║
║  │   │    └──────┬─────┘       │ hết lượt      │             ║
║  │   │           │ được chọn  │ hoặc event     │             ║
║  │   │           ↓             │                │             ║
║  │   │    ┌────────────┐       │                │             ║
║  │   │    │ EXECUTING  │───────┘                │             ║
║  │   │    │ (đang chạy) │                       │             ║
║  │   │    └──────┬─────┘                        │             ║
║  │   │           │ IO/sync/syscall              │             ║
║  │   │           ↓                               │             ║
║  │   │    ┌────────────┐                        │             ║
║  │   └────│  WAITING   │                        │             ║
║  │        │ (đang chờ) │                        │             ║
║  │        └────────────┘                        │             ║
║  │                                              │             ║
║  │  Waiting:   syscall, mutex, channel, atomic  │             ║
║  │  Runnable:  muốn M nhưng đang ĐỢI         │             ║
║  │  Executing: ĐANG chạy trên M → work done!  │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Context Switching Events — 4 Sự Kiện

```
╔═══════════════════════════════════════════════════════════════╗
║   4 SỰ KIỆN TẠO CƠ HỘI SCHEDULING!                          ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Cooperating scheduler = cần "safe points"!                    ║
║  Safe points = xảy ra trong FUNCTION CALLS!                   ║
║                                                               ║
║  ① go KEYWORD:                                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ go myFunc()                                   │             ║
║  │ → Tạo goroutine MỚI!                        │             ║
║  │ → Scheduler CÓ CƠ HỘI scheduling!           │             ║
║  │ → Có thể chuyển G hiện tại sang Runnable!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ② GARBAGE COLLECTION:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → GC chạy bằng CHÍNH CÁC goroutines!        │             ║
║  │ → GC goroutines cần M để chạy!              │             ║
║  │ → Tạo "scheduling chaos"!                    │             ║
║  │ → Smart: G touch heap ↔ G không touch heap │             ║
║  │   được swap thông minh!                      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ③ SYSTEM CALLS:                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → G gọi syscall → có thể BLOCK M!           │             ║
║  │ → Scheduler có thể context-switch G khỏi M!│             ║
║  │ → Hoặc tạo M MỚI! (§5 và §6!)             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ④ SYNCHRONIZATION & ORCHESTRATION:                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → atomic, mutex, channel operations!          │             ║
║  │ → G bị block → scheduler switch G khác!     │             ║
║  │ → G ready lại → quay vào queue!             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ⚠️ TIGHT LOOP PROBLEM:                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ for { /* compute, NO function calls */ }      │             ║
║  │ → KHÔNG CÓ safe point!                       │             ║
║  │ → Scheduler KHÔNG THỂ preempt!              │             ║
║  │ → Go 1.14+: non-cooperative preemption FIX!  │             ║
║  │ → Dùng signal-based preemption (async!)      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Async System Calls — Network Poller

### Network Poller là gì?

```
╔═══════════════════════════════════════════════════════════════════════╗
║   NETWORK POLLER = XỬ LÝ NETWORK IO MÀ KHÔNG BLOCK M!               ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  Khi G gọi network syscall (TCP connect, HTTP request, DNS...):      ║
║  → OS HIỆN ĐẠI hỗ trợ xử lý ASYNC!                                ║
║  → Go Runtime dùng NETWORK POLLER để tận dụng!                      ║
║                                                                       ║
║  ┌──────────────────────────────────────────────────────────┐        ║
║  │ Network Poller sử dụng gì bên dưới?                     │        ║
║  ├───────────────┬──────────────────────────────────────────┤        ║
║  │ MacOS         │ kqueue                                    │        ║
║  │ Linux         │ epoll                                     │        ║
║  │ Windows       │ iocp (I/O Completion Ports)               │        ║
║  ├───────────────┴──────────────────────────────────────────┤        ║
║  │ → Tất cả đều là EFFICIENT EVENT LOOP!                   │        ║
║  │ → Net Poller có OS Thread riêng!                        │        ║
║  │ → G chờ network = vào Net Poller, KHÔNG block M!       │        ║
║  └──────────────────────────────────────────────────────────┘        ║
║                                                                       ║
║  KẾT QUẢ:                                                            ║
║  → M vẫn available phục vụ G khác trong LRQ!                       ║
║  → KHÔNG CẦN tạo M mới!                                            ║
║  → GIẢM tải cho OS!                                                 ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Figure 3 — Trạng thái ban đầu: G1 đang executing, muốn gọi network

```
    G1 đang chạy trên M và muốn gọi network call.

    ┌═══════════╗
    ║           ║
    ║    Net    ║
    ║  Poller   ║                  /\
    ║           ║                 /  \
    ║  (idle)   ║                / M  \─────────────┐ Core │
    ║           ║               /______\            └──────┘
    ╚═══════════╝                  │
                                (G1) ← EXECUTING trên M
                                   │
                               ┌───┴───┐         LRQ
                               │       │    ┌─────┴─────────┐
                               │   P   │    │               │
                               │       │   (G2)  (G3)  (G4)
                               └───────┘    đang đợi trong LRQ

    GIẢI THÍCH:
    → G1 đang EXECUTING trên M → M đang chạy trên Core
    → Net Poller đang IDLE (chưa có ai)
    → G2, G3, G4 đang đợi trong LRQ (Runnable)
    → G1 chuẩn bị gọi net.Dial(), http.Get()...
```

### Figure 4 — G1 vào Net Poller, G2 lên M thay thế

```
    G1 được chuyển vào Net Poller.
    Network call xử lý ASYNC. M available cho G2!

    ┌═══════════╗
    ║           ║
    ║    Net    ║
    ║  Poller   ║                  /\
    ║           ║                 /  \
    ║   (G1) ←──║── G1 VÀO ĐÂY / M  \─────────────┐ Core │
    ║  waiting  ║               /______\            └──────┘
    ╚═══════════╝                  │
                                (G2) ← G2 LÊN THAY! context-switch!
                                   │
                               ┌───┴───┐         LRQ
                               │       │    ┌─────┴───────┐
                               │   P   │    │             │
                               │       │   (G3)      (G4)
                               └───────┘    G2 đã lên M rồi!

    ĐIỀU GÌ XẢY RA:
    ┌──────────────────────────────────────────────────────────────┐
    │ 1. G1 gọi network syscall (VD: net.Dial("tcp", addr))      │
    │ 2. Go runtime PHÁT HIỆN: đây là network call → ASYNC!      │
    │ 3. G1 KHÔNG block M! G1 chuyển vào NET POLLER!             │
    │ 4. Net Poller xử lý network call ASYNC (kqueue/epoll)       │
    │ 5. M bây giờ RẢNH → lấy G2 từ LRQ lên executing!          │
    │ 6. KHÔNG CẦN tạo M mới! ✅ M tái sử dụng ngay!           │
    └──────────────────────────────────────────────────────────────┘
```

### Figure 5 — Network call xong, G1 quay lại LRQ

```
    Net Poller hoàn thành network call.
    G1 quay lại LRQ, đợi được schedule lại.

    ┌═══════════╗
    ║           ║
    ║    Net    ║
    ║  Poller   ║                  /\
    ║           ║                 /  \
    ║  (idle) ──║── G1 ĐÃ RA!  / M  \─────────────┐ Core │
    ║           ║               /______\            └──────┘
    ╚═══════════╝                  │
                                (G2) ← vẫn EXECUTING
                                   │
                               ┌───┴───┐          LRQ
                               │       │    ┌──────┴──────────┐
                               │   P   │    │                 │
                               │       │   (G3)  (G4)  (G1) ★
                               └───────┘          G1 QUAY LẠI LRQ!

    ĐIỀU GÌ XẢY RA:
    ┌──────────────────────────────────────────────────────────────┐
    │ 1. Net Poller nhận được response từ network!                │
    │ 2. G1 chuyển từ Net Poller → LRQ của P!                    │
    │ 3. G1 trở thành Runnable, đợi lượt trên M!                │
    │ 4. G2 vẫn đang executing bình thường!                      │
    │ 5. G1 sẽ được schedule lại khi đến lượt!                  │
    │                                                              │
    │ ★ BIG WIN:                                                   │
    │ → KHÔNG M nào bị block! KHÔNG M nào bị lãng phí!          │
    │ → KHÔNG cần tạo M mới!                                     │
    │ → Net Poller chỉ dùng 1 OS Thread cho event loop!          │
    │ → = HIỆU QUẢ TỐI ĐA!                                     │
    └──────────────────────────────────────────────────────────────┘
```

### Tổng hợp Flow — Network Poller

```
    ┌─────────────────────────────────────────────────────────────────┐
    │  ASYNC SYSCALL FLOW (Network):                                  │
    │                                                                 │
    │  G1 executing ──→ G1 gọi net syscall                           │
    │       │                    │                                     │
    │       │                    ▼                                     │
    │       │            ┌──────────────┐                             │
    │       │            │  Net Poller  │  ← G1 vào đây, chờ ASYNC  │
    │       │            └──────┬───────┘                             │
    │       │                   │                                     │
    │       ▼                   │ (network response)                  │
    │  G2 lên M thay!          │                                     │
    │  (context-switch)        ▼                                     │
    │       │            G1 quay lại LRQ                              │
    │       │                   │                                     │
    │       ▼                   ▼                                     │
    │  M vẫn BUSY!       G1 đợi lượt schedule lại                  │
    │                                                                 │
    │  KEY: M KHÔNG BAO GIỜ bị block bởi network IO!                │
    └─────────────────────────────────────────────────────────────────┘
```

---

## §6. Sync System Calls — M Detach

### Khi nào Net Poller KHÔNG dùng được?

```
╔═══════════════════════════════════════════════════════════════════════╗
║   SYNC SYSCALL = M BỊ BLOCK → PHẢI DETACH M + THAY M MỚI!          ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  Net Poller CHỈ dùng cho NETWORK IO!                                  ║
║  Các trường hợp sau KHÔNG dùng được Net Poller:                      ║
║                                                                       ║
║  ┌──────────────────────────────────────────────────────────┐        ║
║  │ ❌ File I/O:   os.Open(), os.ReadFile(), os.WriteFile() │        ║
║  │ ❌ CGO calls:  gọi C functions qua cgo                  │        ║
║  │ ❌ Disk I/O:   database file operations                  │        ║
║  │ ❌ Syscalls:   mà OS không hỗ trợ async                │        ║
║  ├──────────────────────────────────────────────────────────┤        ║
║  │ ⚠️ Trường hợp đặc biệt:                                │        ║
║  │ → Windows CÓ THỂ file I/O async (dùng iocp)!           │        ║
║  │ → Linux/Mac: file I/O = LUÔN SYNC = BLOCK M!           │        ║
║  └──────────────────────────────────────────────────────────┘        ║
║                                                                       ║
║  HẬU QUẢ: M BỊ BLOCK! Không thể tránh!                             ║
║  GIẢI PHÁP: DETACH M cũ, ATTACH M mới!                              ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Figure 6 — G1 đang executing, chuẩn bị gọi blocking syscall

```
    G1 đang chạy trên M1 và muốn gọi blocking system call (file I/O).

                                  /\
                                 /  \
                                / M1 \─────────────┐ Core │
                               /______\            └──────┘
                                  │
                                (G1) ← EXECUTING, chuẩn bị gọi
                                  │      os.ReadFile("data.json")
                              ┌───┴───┐         LRQ
                              │       │    ┌─────┴─────────┐
                              │   P   │    │               │
                              │       │   (G2)  (G3)  (G4)
                              └───────┘    đang đợi Runnable

    → G1 sắp gọi file I/O syscall
    → File I/O = SYNC = SẼ BLOCK M1!
    → G2, G3, G4 trong LRQ sẽ bị ẢNH HƯỞNG nếu không xử lý!
```

### Figure 7 — M1 bị detach, M2 mới được đưa vào!

```
    M1 bị tách khỏi P (vẫn giữ G1 blocked).
    M2 mới được đưa vào. G2 context-switch lên M2!

         /\                                  /\
        /  \                                /  \
       / M1 \    ← BLOCKED!               / M2 \─────────────┐ Core │
      /______\     chờ file I/O           /______\            └──────┘
         │                                   │
       (G1)   ← vẫn gắn M1!            (G2) ← G2 LÊN M2 THAY!
                 đang blocked!               │
                                         ┌───┴───┐         LRQ
      ════════════════════               │       │    ┌─────┴───────┐
       TÁCH RA KHỎI P!                  │   P   │    │             │
      ════════════════════               │       │   (G3)      (G4)
                                         └───────┘

    ĐIỀU GÌ XẢY RA — TỪNG BƯỚC:
    ┌──────────────────────────────────────────────────────────────────┐
    │ 1. G1 gọi os.ReadFile() → syscall BLOCK M1!                    │
    │ 2. Scheduler PHÁT HIỆN: M1 bị blocked!                         │
    │ 3. Scheduler TÁCH M1 ra khỏi P! (M1 + G1 = cặp blocked)      │
    │ 4. Scheduler ĐƯA M2 VÀO thay thế! (M mới HOẶC tái dùng!)    │
    │ 5. P bây giờ GẮN VỚI M2!                                      │
    │ 6. G2 được chọn từ LRQ → context-switch lên M2!               │
    │ 7. M2 chạy trên Core → G2 bắt đầu executing!                 │
    │                                                                  │
    │ ★ KEY: Nếu M2 đã tồn tại từ lần swap TRƯỚC                   │
    │   → transition NHANH hơn tạo M mới!                            │
    │ ★ Vì vậy Go runtime GIỮ LẠI M cũ để tái dùng!               │
    └──────────────────────────────────────────────────────────────────┘
```

### Figure 8 — Blocking syscall xong, G1 quay lại LRQ

```
    File I/O hoàn thành. G1 quay lại LRQ.
    M1 được giữ lại để dùng cho lần sau.

         /\                                  /\
        /  \                                /  \
       / M1 \    ← ĐỂ DÀNH!              / M2 \─────────────┐ Core │
      /______\     cho lần sau            /______\            └──────┘
                   (idle)                    │
                                          (G2) ← vẫn EXECUTING
                                             │
                                         ┌───┴───┐          LRQ
                                         │       │    ┌──────┴──────────┐
                                         │   P   │    │                 │
                                         │       │   (G3)  (G4)  (G1) ★
                                         └───────┘    G1 QUAY LẠI LRQ!

    ĐIỀU GÌ XẢY RA:
    ┌──────────────────────────────────────────────────────────────────┐
    │ 1. os.ReadFile() HOÀN THÀNH! File I/O xong!                    │
    │ 2. G1 rời M1 → chuyển lại vào LRQ của P!                      │
    │ 3. G1 trở thành Runnable, đợi lượt trên M2!                   │
    │ 4. M1 KHÔNG bị hủy! M1 giữ lại cho lần blocking tiếp theo!   │
    │ 5. Nếu G nào khác gọi sync syscall → M1 sẵn sàng tái dùng!  │
    │                                                                  │
    │ ★ THÔNG MINH: Go runtime duy trì "POOL" các M!                 │
    │   → Tạo M mới = TỐN KÉM (OS syscall)!                        │
    │   → Tái dùng M cũ = NHANH + TIẾT KIỆM!                       │
    └──────────────────────────────────────────────────────────────────┘
```

### Tổng hợp Flow — Sync Syscall

```
    ┌─────────────────────────────────────────────────────────────────┐
    │  SYNC SYSCALL FLOW (File I/O / CGO):                            │
    │                                                                 │
    │  G1 executing ──→ G1 gọi sync syscall (file I/O)              │
    │       │                    │                                     │
    │       │                    ▼                                     │
    │       │            M1 BỊ BLOCKED!                               │
    │       │            (G1 gắn M1)                                  │
    │       │                    │                                     │
    │       ▼                    ▼                                     │
    │  P TÁCH khỏi M1      M1 chờ syscall xong                      │
    │  P GẮN VÀO M2             │                                     │
    │       │                    │ (syscall complete)                  │
    │       ▼                    ▼                                     │
    │  G2 lên M2 thay!    G1 quay lại LRQ                            │
    │  (context-switch)    M1 giữ lại (idle pool)                    │
    │       │                                                         │
    │       ▼                                                         │
    │  M2 phục vụ LRQ bình thường!                                   │
    │                                                                 │
    │  KEY: P là thứ KHÔNG THAY ĐỔI!                                │
    │       P chỉ đổi M khi M bị block!                              │
    └─────────────────────────────────────────────────────────────────┘
```

---

## §7. Work Stealing — Ăn Cắp Công Việc

### Tại sao cần Work Stealing?

```
╔═══════════════════════════════════════════════════════════════════════╗
║   WORK STEALING = P RẢNH → LẤY VIỆC CỦA P KHÁC!                    ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  VẤN ĐỀ:                                                             ║
║  → Nếu P rảnh → M của P rảnh → OS thấy M rảnh                     ║
║  → OS context-switch M KHỎI Core!                                    ║
║  → P không thể schedule G nào cả!                                    ║
║  → = LÃNG PHÍ CPU! Dù có G ở P khác đang đợi!                     ║
║                                                                       ║
║  GIẢI PHÁP: WORK STEALING!                                            ║
║  → P rảnh TỰ ĐI LẤY G từ P khác!                                  ║
║  → CÂN BẰNG tải giữa các P!                                        ║
║  → M giữ "spinning" state = KHÔNG bị OS switch off Core!           ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Figure 9 — Ban đầu: 2 P phục vụ 4 G mỗi P, GRQ có G9

```
    Chúng ta có 2 P, mỗi P phục vụ 4 G.
    GRQ có 1 G chưa gán cho P nào.

         GRQ
         ─┬─
         (G9) ← chưa gán P!

         /\                              /\
        /  \                            /  \
       / M  \                          / M  \
      /______\                        /______\
         │                               │
       (G2) ← executing              (G1) ← executing
         │                               │
     ┌───┴───┐        LRQ          ┌───┴───┐        LRQ
     │       │   ┌────┴────────┐   │       │   ┌────┴────────┐
     │  P1   │   │             │   │  P2   │   │             │
     │       │  (G4) (G6) (G8) │   │       │  (G3) (G5) (G7)
     └───────┘   └─────────────┘   └───────┘   └─────────────┘

    → P1: G2 executing, G4/G6/G8 trong LRQ (3 G đợi)
    → P2: G1 executing, G3/G5/G7 trong LRQ (3 G đợi)
    → GRQ: G9 chưa gán cho P nào
    → Tổng: 9 goroutines, phân bố đều!
```

### Figure 10 — P1 hết việc! LRQ trống, cần steal!

```
    P1 đã xử lý XONG tất cả G trong LRQ. TRỐNG!
    P1 cần tìm việc. P2 vẫn còn G3/G5/G7 trong LRQ.

         GRQ
         ─┬─
         (G9) ← vẫn ở GRQ

         /\                              /\
        /  \                            /  \
       / M  \    ← M RẢNH!            / M  \
      /______\     không có G!        /______\
                                         │
                                       (G1) ← vẫn executing
                                         │
     ┌───────┐        LRQ          ┌───┴───┐        LRQ
     │       │   ┌────────────┐    │       │   ┌────┴────────┐
     │  P1   │   │   TRỐNG!   │    │  P2   │   │             │
     │       │   │            │    │       │  (G3) (G5) (G7)
     └───────┘   └────────────┘    └───────┘   └─────────────┘

    → P1 services xong all G's! LRQ = TRỐNG!
    → P1 sẽ cố steal work từ P2!
```

### Figure 11 — P1 STEAL nửa từ P2! G3 executing ngay!

```
    P1 steals NỬA G's từ P2 (3 G ÷ 2 = lấy 2: G3/G5).
    G3 bắt đầu executing NGAY LẬP TỨC trên M của P1!

         GRQ
         ─┬─
         (G9) ← vẫn ở GRQ

         /\                              /\
        /  \                            /  \
       / M  \                          / M  \
      /______\                        /______\
         │                               │
       (G3) ← G3 executing NGAY!    (G1) ← vẫn executing
         │                               │
     ┌───┴───┐        LRQ          ┌───┴───┐        LRQ
     │       │   ┌────┴──────┐     │       │   ┌────┴──────┐
     │  P1   │   │           │     │  P2   │   │           │
     │       │  (G5) ★STOLEN!│     │       │  (G7)         │
     └───────┘   └───────────┘     └───────┘   └───────────┘

    ĐIỀU GÌ XẢY RA:
    ┌──────────────────────────────────────────────────────────────┐
    │ 1. P1 chạy runtime.schedule()                               │
    │ 2. Check LRQ mình → TRỐNG!                                 │
    │ 3. Steal từ P2 → LRQ P2 có 3 G (G3/G5/G7)               │
    │ 4. Lấy NỬA = 2 G: G3 + G5!                               │
    │ 5. G3 → executing NGAY trên M của P1!                      │
    │ 6. G5 → vào LRQ của P1 (đợi G3 xong)!                    │
    │ 7. P2 còn G7 trong LRQ!                                    │
    │ 8. CÂN BẰNG! P1: 2G, P2: 2G (G1 exec + G7 LRQ)         │
    └──────────────────────────────────────────────────────────────┘
```

### Figure 12 — P2 cũng hết việc! P1 LRQ cũng trống!

```
    P2 services xong all G's. P1 cũng chỉ còn G5 đang executing.
    P2 cần tìm việc: steal P1? → P1 LRQ trống! → Check GRQ!

         GRQ
         ─┬─
         (G9) ← P2 sẽ lấy!

         /\                              /\
        /  \                            /  \
       / M  \                          / M  \    ← RẢNH!
      /______\                        /______\
         │
       (G5) ← executing
         │
     ┌───┴───┐        LRQ          ┌───────┐        LRQ
     │       │   ┌────────────┐    │       │   ┌────────────┐
     │  P1   │   │   TRỐNG!   │    │  P2   │   │   TRỐNG!   │
     │       │   │            │    │       │   │            │
     └───────┘   └────────────┘    └───────┘   └────────────┘

    → P2 chạy runtime.schedule():
    → Check LRQ mình     → TRỐNG!
    → Steal từ P1        → P1 LRQ CŨNG TRỐNG!
    → Check GRQ          → CÓ G9!
```

### Figure 13 — P2 lấy G9 từ GRQ! G9 executing!

```
    G9 được P2 lấy từ GRQ. G9 context-switch vào M.
    Work is getting done!

         GRQ
         ────── (TRỐNG!)

         /\                              /\
        /  \                            /  \
       / M  \                          / M  \
      /______\                        /______\
         │                               │
       (G5) ← executing              (G9) ← executing!
         │                               │       lấy từ GRQ!
     ┌───┴───┐                     ┌───┴───┐
     │  P1   │                     │  P2   │
     └───────┘                     └───────┘

    → P2 lấy G9 từ GRQ! G9 context-switch ngay!
    → P1 vẫn executing G5!
    → Cả 2 P đều có việc! Không G nào bị "bỏ quên"!
    → GRQ giờ TRỐNG → tất cả G đã được phân phối!
```

### Luật Work Stealing (runtime.schedule)

```
    ┌──────────────────────────────────────────────────────────────┐
    │  runtime.schedule() — THỨ TỰ ƯU TIÊN:                      │
    │                                                              │
    │  ┌───┐                                                       │
    │  │ 1 │ 1/61 lần: Check GRQ (Global Run Queue)              │
    │  └─┬─┘   → Tại sao 1/61? Tránh LOCK CONTENTION!           │
    │    │      → GRQ dùng mutex → check quá nhiều = CHẬM!      │
    │    ▼                                                         │
    │  ┌───┐                                                       │
    │  │ 2 │ Check LRQ CỦA MÌNH (Local Run Queue)                │
    │  └─┬─┘   → Ưu tiên! Data locality tốt!                    │
    │    │      → KHÔNG cần lock (chỉ P mình truy cập)!          │
    │    ▼                                                         │
    │  ┌───┐                                                       │
    │  │ 3 │ Steal từ LRQ CỦA P KHÁC                             │
    │  └─┬─┘   → Lấy NỬA số G trong LRQ P khác!                │
    │    │      → Tại sao nửa? CÂN BẰNG chứ không lấy sạch!    │
    │    ▼                                                         │
    │  ┌───┐                                                       │
    │  │ 4 │ Check GRQ lần nữa                                   │
    │  └─┬─┘   → Phòng trường hợp GRQ vừa có G mới!           │
    │    │                                                         │
    │    ▼                                                         │
    │  ┌───┐                                                       │
    │  │ 5 │ Poll network poller                                   │
    │  └───┘   → Có G nào xong network call chưa?               │
    │                                                              │
    │  ★ M "SPINNING":                                             │
    │  → M không tìm được G = "spinning"                          │
    │  → Spinning = vẫn chiếm Core, NHƯNG không idle!            │
    │  → Tốt hơn idle vì không cần OS re-schedule M lên Core!   │
    └──────────────────────────────────────────────────────────────┘
```

---

## §8. Goroutine vs OS Thread — So Sánh Thực Tế

### Bài toán: 2 "threads" gửi message qua lại

```
╔═══════════════════════════════════════════════════════════════════════╗
║   CÙNG 1 BÀI TOÁN — 2 CÁCH GIẢI: OS Thread vs Goroutine             ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  Bài toán: T1 gửi message cho T2, T2 xử lý xong gửi lại T1...      ║
║  Lặp lại liên tục (ping-pong pattern).                                ║
║                                                                       ║
║  ┌─────────────┬────────────────────────────────────────────┐        ║
║  │ OS Threads  │ MỖI lần gửi = OS CONTEXT SWITCH!           │        ║
║  │ (C/C++)     │ T1 pause → T2 resume, nhảy giữa cores     │        ║
║  ├─────────────┼────────────────────────────────────────────┤        ║
║  │ Goroutines  │ MỖI lần gửi = GO RUNTIME SWITCH!           │        ║
║  │ (Go)        │ G1 pause → G2 resume, CÙNG core + M!      │        ║
║  └─────────────┴────────────────────────────────────────────┘        ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Figure 14 — OS Threads: T1 gửi message cho T2 (1 chiều)

```
    T1 đang executing trên C1. T2 đang waiting.
    T1 gửi message → T1 chuyển waiting, T2 nhận → executing!

         /\                                        /\
        /  \                                      /  \
       / T1 \                                    / T2 \
      /______\                                  /______\
         │                                         │
     ┌───┴───┐                                 ┌───┴───┐
     │  ctx  │                                 │Waiting│
     │switch │                                 │       │
     └───┬───┘                                 └───┬───┘
         │                                         │
     ┌───┴─────┐                                   │
  ⬠ │Executing│──── Message ──────────────────→   │
 C1  └───┬─────┘                                   │
         │                                     ┌───┴─────┐
     ┌───┴───┐                                 │Runnable │
     │  ctx  │                                 └───┬─────┘
     │switch │                                 ┌───┴───┐
     └───┬───┘                                 │  ctx  │
         │                                     │switch │
     ┌───┴─────┐                               └───┬───┘
     │Waiting  │                                   │
     └───┬─────┘                               ┌───┴─────┐  ⬠
         │                                     │Executing│  C2
         │                                     └───┬─────┘
         ▼                                         ▼

    CHÚ Ý:
    → T1 chạy trên C1 (Core 1)
    → T2 nhận message → Runnable → OS ctx switch → Executing trên C2!
    → T1 và T2 trên KHÁC CORE! (C1 ≠ C2)
    → ~1,000ns cho MỖI lần OS context switch!
```

### Figure 15 — OS Threads: Ping-Pong đầy đủ (T1→T2→T1)

```
    Full round trip: T1 gửi → T2 nhận + gửi lại → T1 nhận.
    CHÚ Ý: T1 nhảy từ C1 → C3! (KHÁC CORE mỗi lần!)

         /\                                        /\
        /  \                                      /  \
       / T1 \                                    / T2 \
      /______\                                  /______\
         │                                         │
     ┌───┴───┐                                 ┌───┴─────┐
     │  ctx  │                                 │Waiting  │
     │switch │                                 │         │
     └───┬───┘                                 └───┬─────┘
         │                                         │
     ┌───┴─────┐                                   │
     │Executing│──── Message ──────────────────→   │
     └───┬─────┘                                   │
         │                                     ┌───┴─────┐
     ┌───┴───┐                                 │Runnable │
     │  ctx  │                                 └───┬─────┘
     │switch │                                 ┌───┴───┐
     └───┬───┘                                 │  ctx  │
         │                                     │switch │
     ┌───┴─────┐                               └───┬───┘
     │Waiting  │                                   │
     └───┬─────┘                               ┌───┴─────┐  ⬠
         │                                     │Executing│  C2
         │     ←──── Message ──────────────────┤         │
         │                                     └───┬─────┘
     ┌───┴─────┐                               ┌───┴───┐
     │Runnable │                               │  ctx  │
     └───┬─────┘                               │switch │
     ┌───┴───┐                                 └───┬───┘
     │  ctx  │                                     │
     │switch │                                 ┌───┴─────┐
     └───┬───┘                                 │Waiting  │
         │                                     └───┬─────┘
     ┌───┴─────┐  ⬠                               │
     │Executing│  C3  ← KHÁC CORE!               │
     └───┬─────┘       (C1→C3!)                   │
         │                                         │
         ▼                                         ▼

    ⚠️ VẤN ĐỀ NGHIÊM TRỌNG:
    ┌──────────────────────────────────────────────────────────────┐
    │ 1 round trip = 4 lần OS context switch!                     │
    │                                                              │
    │ → T1 bắt đầu trên C1, kết thúc trên C3! (KHÁC CORE!)     │
    │ → Mỗi ctx switch ~1,000ns = ~12,000 instructions mất!     │
    │ → Cache C1 ≠ Cache C3 = phải load lại = CACHE MISS!       │
    │ → OS thấy T1 Waiting → switch T1 off Core!                 │
    │ → OS thấy T2 Waiting → switch T2 off Core!                 │
    │ → DOUBLE PENALTY: switch cost + cache miss cost!            │
    │ → Core liên tục bị đổi = data locality = RẤT TỆ!          │
    └──────────────────────────────────────────────────────────────┘
```

### Figure 16 — Goroutines: G1 gửi message cho G2 (1 chiều)

```
    G1 đang executing trên M1/C1. G2 đang waiting.
    G1 gửi message → Go runtime switch → G2 executing!
    KEY: G1 và G2 chia sẻ CÙNG M1 và CÙNG C1!

         /\                                        /\
        /  \                                      /  \
       / G1 \                                    / G2 \
      /______\                                  /______\
         │                                         │
     ┌───┴───┐                                 ┌───┴─────┐
     │  ctx  │                                 │Waiting  │
     │switch │ ← Go runtime!                  │         │
     └───┬───┘   KHÔNG phải OS!               └───┬─────┘
         │                                         │
  /\  ┌──┴──────┐                                  │
 / M1\ │Executing│──── Message ────────────────→   │
/____/ └──┬──────┘                                  │
  │       │                                         │
  ⬠      ▼                                         ▼
  C1

    → G1 executing trên M1/C1, gửi message cho G2
    → G2 nhận message → Runnable → Go ctx switch
    → Go runtime switch G1 off, G2 on = CÙNG M1, CÙNG C1!
    → ~200ns! (5x nhanh hơn OS switch!)
```

### Figure 17 — Goroutines: Ping-Pong đầy đủ (G1→G2→G1)

```
    Full round trip: G1 gửi → G2 nhận + gửi lại → G1 nhận.
    TẤT CẢ xảy ra trên CÙNG M1, CÙNG C1! KHÔNG ĐỔI CORE!

         /\                                        /\
        /  \                                      /  \
       / G1 \                                    / G2 \
      /______\                                  /______\
         │                                         │
     ┌───┴───┐                                 ┌───┴─────┐
     │  ctx  │                                 │Waiting  │
     │switch │ ← Go runtime!                  │         │
     └───┬───┘   ~200ns!                      └───┬─────┘
         │                                         │
     ┌───┴─────┐                                   │
     │Executing│──── Message ──────────────────→   │
     └───┬─────┘                                   │
         │                                     ┌───┴─────┐
     ┌───┴───┐                                 │Runnable │
     │  ctx  │ ← Go runtime!                  └───┬─────┘
     │switch │   ~200ns!                       ┌───┴───┐
     └───┬───┘                                 │  ctx  │ ← Go runtime!
         │                                     │switch │   ~200ns!
     ┌───┴─────┐                               └───┬───┘
     │Waiting  │                                   │
     └───┬─────┘                               ┌───┴─────┐  /\
         │                                     │Executing│ / M1\
         │     ←──── Message ──────────────────┤         │/____/
         │                                     └───┬─────┘  │
     ┌───┴─────┐                               ┌───┴───┐    ⬠
     │Runnable │                               │  ctx  │    C1
     └───┬─────┘                               │switch │
     ┌───┴───┐                                 └───┬───┘ ← VẪN C1!
     │  ctx  │ ← Go runtime!                      │
     │switch │   ~200ns!                       ┌───┴─────┐
     └───┬───┘                                 │Waiting  │
         │                                     └───┬─────┘
     ┌───┴─────┐  /\                               │
     │Executing│ / M1\  ← VẪN M1, VẪN C1!        │
     └───┬─────┘/____/    KHÔNG ĐỔI CORE!         │
         │       │                                  │
         │       ⬠                                  │
         ▼       C1 ← VẪN C1!                      ▼

    ✅ SỰ KHÁC BIỆT CỐT LÕI:
    ┌──────────────────────────────────────────────────────────────┐
    │ 1 round trip = 4 lần Go context switch!                     │
    │                                                              │
    │ → G1 BẮT ĐẦU trên M1/C1                                   │
    │ → G2 chạy trên M1/C1 (CÙNG!)                              │
    │ → G1 KẾT THÚC trên M1/C1 (VẪN CÙNG!)                     │
    │ → Mỗi Go switch ~200ns = ~2,400 instructions mất!         │
    │ → CÙNG core = Cache HIT! Data KHÔNG cần reload!            │
    │ → OS KHÔNG THẤY gì! M1 luôn BUSY trên C1!                 │
    │ → OS KHÔNG switch M1 off C1!                                │
    │ → ZERO OS overhead! ZERO cache miss penalty!                │
    └──────────────────────────────────────────────────────────────┘
```

### So sánh song song: OS Thread vs Goroutine

```
    ┌──────────────────────────────────────────────────────────────┐
    │     OS THREAD               vs        GOROUTINE              │
    │                                                              │
    │  T1        T2                G1        G2                    │
    │  │         │                 │         │                     │
    │  ├ctx sw   ├Waiting          ├ctx sw   ├Waiting              │
    │  ├Exec─────→Message          ├Exec─────→Message              │
    │  ├ctx sw   ├Runnable         ├ctx sw   ├Runnable             │
    │  ├Waiting  ├ctx sw           ├Waiting  ├ctx sw               │
    │  │     ←msg┤Exec             │     ←msg┤Exec                 │
    │  ├Runnable ├ctx sw           ├Runnable ├ctx sw               │
    │  ├ctx sw   ├Waiting          ├ctx sw   ├Waiting              │
    │  ├Exec     │                 ├Exec     │                     │
    │  │         │                 │         │                     │
    │  C1→C3     C2                M1/C1     M1/C1  ← CÙNG!      │
    │  ~~~~~~     ~~~              ~~~~~     ~~~~~                  │
    │  KHÁC      KHÁC              CÙNG     CÙNG                  │
    │  CORE!     CORE!             CORE!    CORE!                  │
    │                                                              │
    │  4x OS ctx switch            4x Go ctx switch                │
    │  4x ~1,000ns                 4x ~200ns                       │
    │  = 4,000ns                   = 800ns                         │
    │  + cache miss mỗi lần      + cache HIT mỗi lần            │
    │  = CHẬM!                     = 5x NHANH HƠN!               │
    └──────────────────────────────────────────────────────────────┘
```

### Bảng so sánh chi tiết

```
    ┌──────────────────────┬─────────────────┬─────────────────┐
    │ Tiêu chí             │ OS Thread       │ Goroutine       │
    ├──────────────────────┼─────────────────┼─────────────────┤
    │ Context switch       │ ~1,000ns        │ ~200ns          │
    │ Instructions lost    │ ~12,000         │ ~2,400          │
    │ Stack size           │ ~1MB (fixed)    │ ~2KB (growable) │
    │ Creation cost        │ ~1μs            │ ~0.3μs          │
    │ Cache behavior       │ MISS (đổi core)│ HIT (cùng core)✅│
    │ OS sees switching?   │ YES → overhead! │ NO → invisible! │
    │ M state from OS view │ WAITING ❌      │ ALWAYS BUSY ✅  │
    │ Scheduling level     │ Kernel space    │ User space      │
    │ Core bouncing        │ C1→C2→C3 ❌     │ C1→C1→C1 ✅     │
    │ Concurrent 10K       │ 10GB RAM!       │ 20MB RAM! ✅    │
    │ Tốc độ tương đối   │ 1x              │ ~5x ✅          │
    └──────────────────────┴─────────────────┴─────────────────┘
```

### KEY INSIGHT — "IO → CPU-bound at OS level"

```
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │     ★★★ WILLIAM KENNEDY — CÂU QUAN TRỌNG NHẤT ★★★         │
    │                                                              │
    │  "Go has turned IO/Blocking work into                       │
    │   CPU-BOUND work at the OS level."                          │
    │                                                              │
    │  ┌────────────────────────────────────────────────────────┐  │
    │  │ Từ góc nhìn Go code:                                  │  │
    │  │ → Goroutine WAITING (IO, channel, mutex...)           │  │
    │  │ → = IO-BOUND work                                     │  │
    │  │                                                        │  │
    │  │ Từ góc nhìn OS:                                       │  │
    │  │ → M LUÔN EXECUTING! Không bao giờ idle!              │  │
    │  │ → = CPU-BOUND work! (OS nghĩ M luôn bận computation)│  │
    │  │ → OS KHÔNG CẦN context-switch M off Core!           │  │
    │  │                                                        │  │
    │  │ KẾT QUẢ:                                              │  │
    │  │ → HIỆU QUẢ TỐI ĐA:                                 │  │
    │  │   OS giữ M trên Core (no switch penalty!)             │  │
    │  │   Go runtime switch G trên M (rẻ, ~200ns, cache hit!)│  │
    │  │ → = MAGIC CỦA GO SCHEDULER!                          │  │
    │  └────────────────────────────────────────────────────────┘  │
    │                                                              │
    │  VÌ VẬY William Kennedy nói:                                │
    │  "You can reasonably expect to get all your work done       │
    │   with just ONE OS Thread per virtual core."                │
    │                                                              │
    │  → Go KHÔNG CẦN nhiều OS Threads!                          │
    │  → 1 M per P = ĐỦ cho hầu hết workloads!                 │
    │  → Extra M chỉ tạo khi sync syscall block!               │
    │  → "LESS IS MORE" = triết lý Go!                          │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §9. Tổng Kết & Câu Hỏi Phỏng Vấn Senior Golang

```
╔═══════════════════════════════════════════════════════════════╗
║   CÂU HỎI PHỎNG VẤN SENIOR GOLANG                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Q1: "G-M-P model là gì?"                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ G = Goroutine (application-level thread, 2KB) │             ║
║  │ M = Machine = OS Thread (managed by OS)       │             ║
║  │ P = Logical Processor (= virtual core)        │             ║
║  │ → P có LRQ chứa G, gắn 1 M!               │             ║
║  │ → G context-switch trên M!                   │             ║
║  │ → M context-switch trên Core (by OS)!        │             ║
║  │ → GRQ = G chưa gán P!                       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Async vs Sync syscall xử lý thế nào?"                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ASYNC (network): G → Net Poller, M available!│             ║
║  │ → Dùng kqueue/epoll/iocp!                    │             ║
║  │ → KHÔNG cần M mới! M vẫn phục vụ LRQ!     │             ║
║  │                                              │             ║
║  │ SYNC (file I/O): G block M → P detach M1,    │             ║
║  │ attach M2! G1 vẫn gắn M1 blocked!           │             ║
║  │ → Xong: G1 → LRQ, M1 để dành!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Work stealing hoạt động thế nào?"                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ P rảnh → check LRQ mình → steal NỬA G     │             ║
║  │ từ P khác → check GRQ → poll network!       │             ║
║  │ → GRQ check 1/61 lần (tránh lock contention)│             ║
║  │ → M "spinning" = KHÔNG idle = efficient!     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Tại sao Go context switch nhanh hơn OS?"              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Go: ~200ns vs OS: ~1,000ns (5x nhanh hơn!)│             ║
║  │ → Go switch trong USER SPACE (không kernel!) │             ║
║  │ → CÙNG M, CÙNG core = cache HIT!            │             ║
║  │ → OS KHÔNG THẤY switching = M luôn busy!    │             ║
║  │ → "Go turns IO into CPU-bound at OS level!"  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q5: "Tại sao không cần nhiều M hơn virtual cores?"        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Go scheduler tận dụng TỐI ĐA mỗi M!     │             ║
║  │ → G blocking IO → Net Poller (async) hoặc   │             ║
║  │   M detach (sync) → M mới serve LRQ!        │             ║
║  │ → Work stealing = không P nào rảnh!         │             ║
║  │ → 1 M per virtual core = ĐỦ cho hầu hết   │             ║
║  │   workloads (CPU + IO bound)!                 │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: GO SCHEDULER                                 │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  1. G-M-P Model:                                         │
    │     G=Goroutine, M=OS Thread, P=Logical Processor        │
    │     GRQ (global) + LRQ (per-P) run queues                │
    │                                                          │
    │  2. Cooperating Scheduler:                               │
    │     User-space, safe points = function calls             │
    │     Looks preemptive, Go runtime decides!                │
    │                                                          │
    │  3. 4 scheduling events:                                 │
    │     go keyword, GC, syscalls, sync/orchestration         │
    │                                                          │
    │  4. Async syscalls → Net Poller:                        │
    │     M NOT blocked! kqueue/epoll/iocp!                    │
    │                                                          │
    │  5. Sync syscalls → M detach + M2 replace:              │
    │     P detaches from blocked M1, attaches M2!             │
    │                                                          │
    │  6. Work Stealing:                                       │
    │     P steals HALF from other P's LRQ!                    │
    │     GRQ checked 1/61 times!                              │
    │                                                          │
    │  7. Go turns IO → CPU-bound at OS level!                │
    │     ~200ns switch vs OS ~1,000ns = 5x faster!            │
    │     Same core = cache hits + NUMA friendly!              │
    │                                                          │
    │  William Kennedy:                                        │
    │  "You can reasonably expect to get all your work done    │
    │   with just one OS Thread per virtual core."             │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
