# Scheduling In Go: Part I — OS Scheduler (Bộ Lập Lịch Hệ Điều Hành)

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** William Kennedy — "Scheduling In Go: Part I - OS Scheduler" (Ardan Labs)
> **Góc nhìn:** Thread, Process, Context Switch, Cache Line, False Sharing, Mechanical Sympathy
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                       | Mô tả                                         |
| --- | ---------------------------- | --------------------------------------------- |
| §1  | Thread & Process             | Chương trình = chuỗi instructions tuần tự     |
| §2  | Program Counter (PC)         | Con trỏ instruction tiếp theo                 |
| §3  | Thread States                | Waiting, Runnable, Executing                  |
| §4  | CPU-Bound vs IO-Bound        | 2 loại công việc khác nhau hoàn toàn          |
| §5  | Context Switching            | Chi phí hoán đổi Thread ~12K-18K instructions |
| §6  | Less Is More                 | Ít Thread = hiệu quả hơn                      |
| §7  | Cache Lines & False Sharing  | Cache line 64 bytes, cache-coherency problem  |
| §8  | Tổng kết & Câu hỏi phỏng vấn | Ôn tập & thực hành                            |

---

## §1. Thread & Process — Đơn Vị Thực Thi

```
╔═══════════════════════════════════════════════════════════════╗
║   THREAD = "MỘT ĐƯỜNG ĐI CỦA THỰC THI"                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CHƯƠNG TRÌNH LÀ GÌ?                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  = Một chuỗi MACHINE INSTRUCTIONS!           │             ║
║  │  = Phải được thực thi TUẦN TỰ, từng lệnh!  │             ║
║  │  = Hệ điều hành dùng THREAD để thực thi!   │             ║
║  │                                              │             ║
║  │  Thread = "a path of execution"               │             ║
║  │         = "đường đi thực thi"               │             ║
║  │                                              │             ║
║  │  → Thread CHỊU TRÁCH NHIỆM thực thi        │             ║
║  │    instructions TUẦN TỰ cho đến hết!       │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  PROCESS vs THREAD:                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Chạy chương trình → OS tạo PROCESS         │             ║
║  │  Process → được gán 1 THREAD ban đầu       │             ║
║  │  Thread đó → có thể tạo THÊM Threads      │             ║
║  │                                              │             ║
║  │  ┌─────────────────────────────────┐         │             ║
║  │  │ PROCESS (chương trình đang chạy)│         │             ║
║  │  │                                 │         │             ║
║  │  │  ┌─────────┐  ┌─────────┐     │         ║
║  │  │  │ Thread 1 │  │ Thread 2 │     │         ║
║  │  │  │ (main)   │  │ (worker) │     │         ║
║  │  │  └─────────┘  └─────────┘     │         ║
║  │  │       ↓              ↓         │         ║
║  │  │  instruction 1  instruction A  │         ║
║  │  │  instruction 2  instruction B  │         ║
║  │  │  instruction 3  instruction C  │         ║
║  │  │  ...             ...           │         ║
║  │  └─────────────────────────────────┘         │             ║
║  │                                              │             ║
║  │  QUAN TRỌNG:                                  │             ║
║  │  → Scheduling = ở mức THREAD!                │             ║
║  │  → KHÔNG phải ở mức Process!                 │             ║
║  │  → Các threads chạy ĐỘC LẬP nhau!         │             ║
║  │  → Mỗi thread có STATE riêng!               │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CONCURRENT vs PARALLEL:                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  CONCURRENT (đồng thời — 1 core):           │             ║
║  │  Core 0: [T1][T2][T1][T2][T1][T2]            │             ║
║  │  → Thay phiên nhau! Tạo ẢO GIÁC chạy      │             ║
║  │    cùng lúc!                                  │             ║
║  │                                              │             ║
║  │  PARALLEL (song song — nhiều cores):         │             ║
║  │  Core 0: [T1][T1][T1][T1][T1][T1]            │             ║
║  │  Core 1: [T2][T2][T2][T2][T2][T2]            │             ║
║  │  → THỰC SỰ chạy cùng lúc!                  │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TRÁCH NHIỆM CỦA OS SCHEDULER:                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Core KHÔNG được rảnh nếu có thread       │             ║
║  │    cần chạy!                                  │             ║
║  │ 2. Tạo ẢO GIÁC mọi thread chạy đồng thời! │             ║
║  │ 3. Thread ưu tiên CAO chạy trước!           │             ║
║  │ 4. Thread ưu tiên THẤP KHÔNG bị bỏ đói!   │             ║
║  │ 5. Giảm scheduling latency xuống mức TỐI   │             ║
║  │    THIỂU!                                     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Program Counter (PC) — Con Trỏ Lệnh

```
╔═══════════════════════════════════════════════════════════════╗
║   PROGRAM COUNTER = "BẠN ĐANG ĐỌC ĐẾN TRANG NÀO?"         ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  PC (Program Counter) = IP (Instruction Pointer):             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  = Thanh ghi giữ địa chỉ LỆNH TIẾP THEO! │             ║
║  │  = Như bookmark: "đọc đến trang này rồi!"  │             ║
║  │  = QUAN TRỌNG: trỏ đến lệnh TIẾP THEO,    │             ║
║  │    KHÔNG PHẢI lệnh đang chạy!               │             ║
║  │                                              │             ║
║  │  ┌──────────────────────────────┐            │             ║
║  │  │ Instructions    │ PC         │            │             ║
║  │  ├──────────────────┼───────────┤            │             ║
║  │  │ MOVQ GS:0x30, CX│           │            │             ║
║  │  │ CMPQ 0x10, SP   │           │            │             ║
║  │  │ SUBQ $0x18, SP  │ ← đang   │            │             ║
║  │  │ MOVQ BP, 0x10   │ ← PC ★   │            │             ║
║  │  │ LEAQ 0x10, BP   │           │            │             ║
║  │  │ ...              │           │            │             ║
║  │  └──────────────────┴───────────┘            │             ║
║  │                                              │             ║
║  │  → SUBQ đang chạy, PC trỏ đến MOVQ       │             ║
║  │    (lệnh TIẾP THEO!)                         │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VÍ DỤ STACK TRACE TRONG GO:                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  goroutine 1 [running]:                       │             ║
║  │  main.example(...)                            │             ║
║  │    example1.go:13 +0x39  ← PC offset!       │             ║
║  │  main.main()                                  │             ║
║  │    example1.go:8 +0x72   ← PC offset!       │             ║
║  │                                              │             ║
║  │  +0x39 = 57 bytes từ đầu function!          │             ║
║  │  = Lệnh TIẾP THEO (nếu không panic)!        │             ║
║  │  = Lệnh TRƯỚC +0x39 = lệnh đang chạy!     │             ║
║  │                                              │             ║
║  │  → Đọc stack trace: +0xNN cho biết          │             ║
║  │    CHÍNH XÁC vị trí trong machine code!     │             ║
║  │  → go tool objdump để xem instructions!     │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Thread States — Ba Trạng Thái

```
╔═══════════════════════════════════════════════════════════════╗
║   THREAD CÓ 3 TRẠNG THÁI!                                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │         ┌───────────┐                        │             ║
║  │    ┌───→│ RUNNABLE  │←──────┐                │             ║
║  │    │    │ (sẵn sàng)│       │                │             ║
║  │    │    └─────┬─────┘       │                │             ║
║  │    │          │ được chọn  │ hết time slice │             ║
║  │    │          ↓             │                │             ║
║  │    │    ┌───────────┐       │                │             ║
║  │    │    │ EXECUTING │───────┘                │             ║
║  │    │    │ (đang chạy)│                       │             ║
║  │    │    └─────┬─────┘                        │             ║
║  │    │          │ IO/sync                      │             ║
║  │    │          ↓                               │             ║
║  │    │    ┌───────────┐                        │             ║
║  │    └────│ WAITING   │                        │             ║
║  │         │ (đang chờ)│                        │             ║
║  │         └───────────┘                        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CHI TIẾT TỪNG TRẠNG THÁI:                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ① WAITING (Đang chờ):                       │             ║
║  │  → Thread DỪNG, CHỜ gì đó để tiếp tục!    │             ║
║  │  → Chờ hardware: disk, network               │             ║
║  │  → Chờ OS: system calls                      │             ║
║  │  → Chờ sync: mutex, atomic                   │             ║
║  │  → ⚠️ = NGUYÊN NHÂN GỐC của bad perf!      │             ║
║  │                                              │             ║
║  │  ② RUNNABLE (Sẵn sàng):                     │             ║
║  │  → Thread MUỐN chạy nhưng CHƯA có core!    │             ║
║  │  → Đang ĐỢI trong hàng đợi!               │             ║
║  │  → Nhiều threads runnable = ĐỢI LÂU HƠN!  │             ║
║  │  → Mỗi thread được ÍT thời gian hơn!       │             ║
║  │  → ⚠️ = CÓ THỂ gây bad perf!               │             ║
║  │                                              │             ║
║  │  ③ EXECUTING (Đang chạy):                    │             ║
║  │  → Thread ĐANG chạy trên 1 core!             │             ║
║  │  → Thực thi machine instructions!            │             ║
║  │  → CÔNG VIỆC đang được hoàn thành!          │             ║
║  │  → ✅ = AI CŨNG MUỐN ở trạng thái này!    │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. CPU-Bound vs IO-Bound

```
╔═══════════════════════════════════════════════════════════════╗
║   HAI LOẠI CÔNG VIỆC KHÁC NHAU HOÀN TOÀN!                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CPU-BOUND:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  = Công việc KHÔNG BAO GIỜ khiến thread     │             ║
║  │    vào trạng thái Waiting!                    │             ║
║  │  = Tính toán LIÊN TỤC!                       │             ║
║  │                                              │             ║
║  │  Ví dụ:                                       │             ║
║  │  → Tính số Pi đến chữ số thứ N             │             ║
║  │  → Mã hóa video                              │             ║
║  │  → Machine learning training                  │             ║
║  │  → Nén dữ liệu                              │             ║
║  │                                              │             ║
║  │  Timeline:                                    │             ║
║  │  [CALC][CALC][CALC][CALC][CALC][CALC]         │             ║
║  │  → KHÔNG CÓ gaps! CPU luôn bận!            │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  IO-BOUND:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  = Công việc khiến thread VÀO Waiting!       │             ║
║  │  = Chờ tài nguyên BÊN NGOÀI!               │             ║
║  │                                              │             ║
║  │  Ví dụ:                                       │             ║
║  │  → Truy cập database                         │             ║
║  │  → HTTP request đến API                     │             ║
║  │  → Đọc/ghi file                             │             ║
║  │  → Mutex lock/unlock                          │             ║
║  │                                              │             ║
║  │  Timeline:                                    │             ║
║  │  [CALC][WAIT.....][CALC][WAIT...][CALC]       │             ║
║  │  → CÓ gaps! CPU rảnh trong lúc chờ!        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TẠI SAO QUAN TRỌNG?                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌──────────────┬────────────┬────────────┐  │             ║
║  │  │              │ CPU-Bound  │ IO-Bound   │  │             ║
║  │  ├──────────────┼────────────┼────────────┤  │             ║
║  │  │ Context      │ ❌ TỆ!     │ ✅ TỐT!   │  │             ║
║  │  │ switch       │ LÃNG PHÍ! │ TẬN DỤNG! │  │             ║
║  │  ├──────────────┼────────────┼────────────┤  │             ║
║  │  │ Optimal      │ Threads =  │ Threads >  │  │             ║
║  │  │ threads      │ Cores      │ Cores      │  │             ║
║  │  ├──────────────┼────────────┼────────────┤  │             ║
║  │  │ Ví dụ Go     │ Heavy      │ Web server │  │             ║
║  │  │              │ compute    │ API calls  │  │             ║
║  │  └──────────────┴────────────┴────────────┘  │             ║
║  │                                              │             ║
║  │  CPU-Bound + context switch = NIGHTMARE!     │             ║
║  │  → Thread đang CALC → BỊ DỪNG → LÃNG PHÍ! │             ║
║  │                                              │             ║
║  │  IO-Bound + context switch = ADVANTAGE!      │             ║
║  │  → Thread đang WAIT → KHÁC chạy thay!      │             ║
║  │  → Core luôn BẬN = TỐT!                    │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Context Switching — Chi Phí Hoán Đổi Thread

```
╔═══════════════════════════════════════════════════════════════╗
║   CONTEXT SWITCH = HOÁN ĐỔI THREAD TRÊN CORE!                ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  QUÁ TRÌNH:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  TRƯỚC context switch:                        │             ║
║  │  Core 0: [Thread A đang EXECUTING]           │             ║
║  │  Run Queue: [Thread B] [Thread C] ...         │             ║
║  │                                              │             ║
║  │  Scheduler QUYẾT ĐỊNH switch!                │             ║
║  │                                              │             ║
║  │  1. LƯU state Thread A (registers, PC, stack)│             ║
║  │  2. KÉO Thread A ra khỏi core              │             ║
║  │  3. ĐẶT Thread A vào Runnable hoặc Waiting │             ║
║  │  4. LẤY Thread B từ Run Queue               │             ║
║  │  5. LOAD state Thread B lên core             │             ║
║  │  6. Thread B bắt đầu EXECUTING!             │             ║
║  │                                              │             ║
║  │  SAU context switch:                          │             ║
║  │  Core 0: [Thread B đang EXECUTING]           │             ║
║  │  Run Queue: [Thread A] [Thread C] ...         │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CHI PHÍ:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Context switch = ~1,000 - 1,500 ns!         │             ║
║  │                                              │             ║
║  │  CPU thực thi ~12 instructions/ns/core!       │             ║
║  │                                              │             ║
║  │  → 1 context switch = MẤT:                   │             ║
║  │    ~12,000 — ~18,000 instructions!            │             ║
║  │                                              │             ║
║  │  ┌────────────────────────────────────┐      │             ║
║  │  │ 1 context switch:                  │      │             ║
║  │  │ ████████████████████ ~12K-18K      │      │             ║
║  │  │ instructions MẤT TRẮNG!           │      │             ║
║  │  │                                    │      │             ║
║  │  │ → Chương trình KHÔNG làm gì      │      │             ║
║  │  │   trong lúc switch!                │      │             ║
║  │  │ → Chỉ save/load thread state!     │      │             ║
║  │  └────────────────────────────────────┘      │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  PREEMPTIVE SCHEDULER:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Linux, Mac, Windows = PREEMPTIVE!             │             ║
║  │                                              │             ║
║  │ → Scheduler KHÔNG DỰ ĐOÁN ĐƯỢC!           │             ║
║  │ → Không biết chọn thread nào, lúc nào!      │             ║
║  │ → KHÔNG BAO GIỜ dựa vào "tôi thấy nó      │             ║
║  │   chạy kiểu này 1000 lần, chắc luôn vậy!" │             ║
║  │ → PHẢI dùng synchronization nếu cần        │             ║
║  │   determinism!                                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Less Is More — Ít Thread = Hiệu Quả Hơn

```
╔═══════════════════════════════════════════════════════════════╗
║   "LESS IS MORE" — QUY TẮC VÀNG!                             ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  BÀI TOÁN — SCHEDULER PERIOD:                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Scheduler period = 1000ms (1 giây)          │             ║
║  │  Chia đều cho mỗi thread:                   │             ║
║  │                                              │             ║
║  │  ┌──────────┬────────────┬──────────────┐   │             ║
║  │  │ Threads  │ Time/thread│ Context switch│   │             ║
║  │  ├──────────┼────────────┼──────────────┤   │             ║
║  │  │ 10       │ 100ms      │ 10 × ~1.5μs ✅│  │             ║
║  │  │ 100      │ 10ms       │ 100 × ~1.5μs ⚠│  │             ║
║  │  │ 1,000    │ 1ms        │ ❌ QUÁ NGẮN!  │  │             ║
║  │  │ 10,000   │ 0.1ms      │ ❌❌ BẤT KHẢ! │  │             ║
║  │  └──────────┴────────────┴──────────────┘   │             ║
║  │                                              │             ║
║  │  1,000 threads × 1ms time slice:             │             ║
║  │  → Context switch ~1.5μs = 0.15% thời gian│             ║
║  │  → OK... nhưng ĐÃ ở biên!                  │             ║
║  │                                              │             ║
║  │  Nếu min time slice = 10ms:                   │             ║
║  │  → 1,000 threads × 10ms = 10,000ms = 10s!   │             ║
║  │  → 10,000 threads × 10ms = 100,000ms = 100s!│             ║
║  │  → Mất 100 GIÂY để mỗi thread chạy 1 lần!│             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  QUY TẮC:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  "LESS IS MORE"                               │             ║
║  │                                              │             ║
║  │  ÍT threads Runnable:                         │             ║
║  │  → ÍT scheduling overhead!                   │             ║
║  │  → NHIỀU thời gian mỗi thread!              │             ║
║  │  → NHIỀU công việc hoàn thành!               │             ║
║  │                                              │             ║
║  │  NHIỀU threads Runnable:                      │             ║
║  │  → NHIỀU scheduling overhead!                 │             ║
║  │  → ÍT thời gian mỗi thread!                │             ║
║  │  → ÍT công việc hoàn thành!                 │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  THREAD POOL — CÂN BẰNG:                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  William Kennedy (kinh nghiệm NT/C++):       │             ║
║  │  → Web service + Database                     │             ║
║  │  → Magic number = 3 threads / core!           │             ║
║  │                                              │             ║
║  │  ┌──────────┬─────────────────────────┐      │             ║
║  │  │ Threads  │ Kết quả                │      │             ║
║  │  │ per core │                         │      │             ║
║  │  ├──────────┼─────────────────────────┤      │             ║
║  │  │ 2        │ Chậm — có idle time!   │      │             ║
║  │  │ 3 ★      │ TỐT NHẤT!             │      │             ║
║  │  │ 4        │ Chậm — context switch! │      │             ║
║  │  └──────────┴─────────────────────────┘      │             ║
║  │                                              │             ║
║  │  → Go KHÔNG cần Thread pool!                 │             ║
║  │  → Go scheduler TỰ ĐỘNG quản lý!           │             ║
║  │  → Đó là lý do Go tuyệt vời! (Part II!)   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Cache Lines & False Sharing

```
╔═══════════════════════════════════════════════════════════════╗
║   CACHE LINE = 64 BYTES — ĐƠN VỊ TRAO ĐỔI DỮ LIỆU!         ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CPU CACHE HIERARCHY:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌────────┐  ┌────────┐                      │             ║
║  │  │ Core 0 │  │ Core 1 │                      │             ║
║  │  │┌──────┐│  │┌──────┐│                      │             ║
║  │  ││L1 I  ││  ││L1 I  ││  ~1-3ns             │             ║
║  │  ││L1 D  ││  ││L1 D  ││                      │             ║
║  │  │├──────┤│  │├──────┤│                      │             ║
║  │  ││L2    ││  ││L2    ││  ~3-10ns             │             ║
║  │  │└──────┘│  │└──────┘│                      │             ║
║  │  └────┬───┘  └────┬───┘                      │             ║
║  │       └─────┬─────┘                           │             ║
║  │       ┌─────┴─────┐                           │             ║
║  │       │  L3 Cache  │  ~10-40ns                │             ║
║  │       └─────┬─────┘                           │             ║
║  │       ┌─────┴─────┐                           │             ║
║  │       │Main Memory │  ~100-300ns ← CHẬM!    │             ║
║  │       └───────────┘                           │             ║
║  │                                              │             ║
║  │  → Data trao đổi bằng CACHE LINE = 64 bytes│             ║
║  │  → Mỗi core có BẢN COPY riêng!             │             ║
║  │  → Hardware dùng VALUE SEMANTICS!            │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  FALSE SHARING = ARC THẢM HỌA HIỆU SUẤT!                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Tình huống:                                  │             ║
║  │  → Thread A (Core 0) đọc biến X             │             ║
║  │  → Thread B (Core 1) đọc biến Y             │             ║
║  │  → X và Y NẰM CÙNG cache line!              │             ║
║  │  → Chúng ĐỘC LẬP nhau!                     │             ║
║  │                                              │             ║
║  │  Cache line 64 bytes:                         │             ║
║  │  ┌────────────────────────────────┐           │             ║
║  │  │ ... | X | Y | ...              │           │             ║
║  │  └────────────────────────────────┘           │             ║
║  │                                              │             ║
║  │  Core 0 cache: [...|X|Y|..]  (copy)          │             ║
║  │  Core 1 cache: [...|X|Y|..]  (copy)          │             ║
║  │                                              │             ║
║  │  Thread A SỬA X trên Core 0:                │             ║
║  │  → Core 0's cache line: UPDATED!             │             ║
║  │  → Core 1's cache line: DIRTY! ❌            │             ║
║  │  → DÙ Y KHÔNG THAY ĐỔI!                    │             ║
║  │                                              │             ║
║  │  Thread B đọc Y trên Core 1:                │             ║
║  │  → Cache line DIRTY → phải lấy LẠI từ     │             ║
║  │    Main Memory! ~100-300ns!                    │             ║
║  │  → DÙ Y KHÔNG HỀ THAY ĐỔI!               │             ║
║  │                                              │             ║
║  │  = FALSE SHARING!                              │             ║
║  │  = "Chia sẻ GIẢ" — không share data         │             ║
║  │    nhưng SHARE CACHE LINE!                    │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  32-CORE = THẢM HỌA:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → 32 threads song song                        │             ║
║  │ → CÙNG cache line                              │             ║
║  │ → 1 thread sửa → 31 copies DIRTY!           │             ║
║  │ → 31 threads phải fetch RAM!                 │             ║
║  │ → = "THRASHING through memory!"               │             ║
║  │ → Performance HORRIBLE!                        │             ║
║  │ → "Most likely you'll have NO understanding  │             ║
║  │    WHY!" — William Kennedy                    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GIẢI PHÁP TRONG GO:                                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  // ❌ False sharing!                         │             ║
║  │  type Counters struct {                       │             ║
║  │      A int64  // Core 0 sửa                 │             ║
║  │      B int64  // Core 1 sửa ← CÙNG line!  │             ║
║  │  }                                            │             ║
║  │                                              │             ║
║  │  // ✅ Padding tránh false sharing!          │             ║
║  │  type Counters struct {                       │             ║
║  │      A int64                                  │             ║
║  │      _ [56]byte  // padding → KHÁC line!    │             ║
║  │      B int64                                  │             ║
║  │  }                                            │             ║
║  │  // A và B nằm trên CACHE LINE KHÁC NHAU!  │             ║
║  │  // Sửa A → B KHÔNG bị dirty!              │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §8. Tổng Kết & Câu Hỏi Phỏng Vấn Senior Golang

```
╔═══════════════════════════════════════════════════════════════╗
║   CÂU HỎI PHỎNG VẤN SENIOR GOLANG                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Q1: "Thread có mấy trạng thái?"                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 3 trạng thái: Waiting, Runnable, Executing!  │             ║
║  │ → Waiting: dừng, chờ IO/sync = bad perf!    │             ║
║  │ → Runnable: muốn chạy, đợi core!           │             ║
║  │ → Executing: đang chạy trên core!           │             ║
║  │ → Mục tiêu: maximize Executing time!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "CPU-Bound vs IO-Bound khác gì?"                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ CPU-Bound: tính toán liên tục, KHÔNG wait!   │             ║
║  │ → Context switch = LÃNG PHÍ (đang calc!)!   │             ║
║  │ → Optimal: threads = cores!                   │             ║
║  │                                              │             ║
║  │ IO-Bound: chờ IO, thread vào Waiting!        │             ║
║  │ → Context switch = TỐT (tận dụng core!)!    │             ║
║  │ → Optimal: threads > cores (3:1 rule!)!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Context switch tốn bao nhiêu?"                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → ~1,000 - 1,500 nanoseconds!                │             ║
║  │ → CPU ~12 instructions/ns/core!               │             ║
║  │ → 1 switch = MẤT ~12K-18K instructions!      │             ║
║  │ → Preemptive scheduler = KHÔNG dự đoán!     │             ║
║  │ → PHẢI dùng sync nếu cần determinism!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "False sharing là gì?"                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → 2 threads truy cập data ĐỘC LẬP!         │             ║
║  │ → NHƯNG data nằm CÙNG cache line (64 bytes!)│             ║
║  │ → 1 sửa → INVALIDATE copy của thread kia!  │             ║
║  │ → Thread kia phải fetch RAM (~100-300ns!)!    │             ║
║  │ → Fix: padding struct fields để KHÁC line!  │             ║
║  │ → Go: _ [56]byte giữa 2 fields hot!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q5: "Less Is More nghĩa là gì?"                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → ÍT threads Runnable = TỐT hơn!            │             ║
║  │ → Ít scheduling overhead!                     │             ║
║  │ → Mỗi thread được NHIỀU thời gian hơn!     │             ║
║  │ → Thread pool: tìm BALANCE threads vs cores! │             ║
║  │ → Go scheduler TỰ ĐỘNG làm điều này!      │             ║
║  │   (Part II — Go Scheduler!)                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: OS SCHEDULER                                 │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  1. Thread = "path of execution"                         │
    │     → Thực thi instructions TUẦN TỰ!                   │
    │     → Scheduling ở mức THREAD, không Process!           │
    │                                                          │
    │  2. Program Counter (PC):                                │
    │     → Trỏ đến lệnh TIẾP THEO!                         │
    │     → Stack trace +0xNN = PC offset!                     │
    │                                                          │
    │  3. Thread States: Waiting → Runnable → Executing       │
    │     → Waiting = root cause bad perf!                     │
    │                                                          │
    │  4. CPU-Bound vs IO-Bound:                               │
    │     → CPU: context switch = LÃNG PHÍ!                   │
    │     → IO: context switch = TẬN DỤNG!                    │
    │                                                          │
    │  5. Context Switch = ~12K-18K instructions MẤT!         │
    │     → Preemptive = KHÔNG dự đoán được!                │
    │                                                          │
    │  6. Less Is More:                                        │
    │     → Ít threads = hiệu quả hơn!                      │
    │     → Thread pool: 3 threads/core (IO-Bound)!            │
    │                                                          │
    │  7. Cache Lines & False Sharing:                         │
    │     → Cache line = 64 bytes!                              │
    │     → False sharing: cùng line, khác data → DISASTER!  │
    │     → Fix: padding struct fields!                        │
    │                                                          │
    │  William Kennedy:                                        │
    │  "Don't allow a core to go idle                          │
    │   if there is work to be done."                          │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
