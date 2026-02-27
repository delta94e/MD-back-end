# Backend Roadmap Intensive — Từ Zero Đến Senior

## Lộ trình học Backend có hệ thống cho người chưa biết gì

> **Dành cho:** Người mới bắt đầu, chưa biết gì về backend
> **Mục tiêu:** Đạt trình độ Senior Backend Engineer tại Big Tech (Google, Meta, Uber, Stripe...)
> **Phương pháp:** Học theo sequence — từ high-level xuống low-level
> **Ngôn ngữ chính:** Golang (áp dụng được cho mọi ngôn ngữ)

---

## Tổng Quan Roadmap

```
╔═══════════════════════════════════════════════════════════════════════════╗
║                  BACKEND ROADMAP — 12 PHASES (BIG TECH LEVEL)            ║
╠═══════════════════════════════════════════════════════════════════════════╣
║                                                                           ║
║  ── FOUNDATION (Nền tảng) ──────────────────────────────────────────     ║
║  Phase 0  ──► Tư Duy & Nền Tảng CS           (Gốc rễ của mọi thứ)      ║
║  Phase 1  ──► Ngôn Ngữ Lập Trình             (Công cụ chính)            ║
║  Phase 2  ──► Networking & Protocols          (Cách máy nói chuyện)      ║
║  Phase 3  ──► Database & Storage              (Nơi chứa dữ liệu)        ║
║                                                                           ║
║  ── INTERMEDIATE (Trung cấp) ───────────────────────────────────────     ║
║  Phase 4  ──► API Design & Web Server         (Cửa ngõ của backend)      ║
║  Phase 5  ──► Software Design Patterns        (Thiết kế code đúng)       ║
║  Phase 6  ──► Advanced Testing & Quality      (Chất lượng phần mềm)     ║
║                                                                           ║
║  ── ADVANCED (Nâng cao) ────────────────────────────────────────────     ║
║  Phase 7  ──► System Design & Architecture    (Thiết kế hệ thống)        ║
║  Phase 8  ──► Distributed Systems Deep Dive   (Hệ thống phân tán)       ║
║  Phase 9  ──► Go Runtime & Performance Eng.   (Hiểu sâu bên trong)      ║
║                                                                           ║
║  ── BIG TECH READY ─────────────────────────────────────────────────     ║
║  Phase 10 ──► Production & Operations         (Vận hành ở quy mô lớn)   ║
║  Phase 11 ──► Big Tech Interview & Mindset    (Phỏng vấn & Tư duy)      ║
║                                                                           ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

### Tại sao thứ tự này quan trọng?

```
    ⚠️ SAI LẦM PHỔ BIẾN:

    ❌ Nhảy thẳng vào framework (Express, Gin, Spring)
       → Không hiểu tại sao code chạy

    ❌ Copy-paste từ tutorial
       → Không debug được khi lỗi

    ❌ Học database trước khi hiểu networking
       → Không hiểu tại sao query chậm

    ✅ ĐÚNG: Học từ nền tảng → lên cao
       → Mỗi tầng kiến thức XÂY TRÊN tầng trước
       → Khi gặp bug, bạn biết "đào" ở đâu
```

---

## Phase 0: Tư Duy & Nền Tảng Computer Science

### Tại sao Phase này QUAN TRỌNG NHẤT?

```
╔═══════════════════════════════════════════════════════════════╗
║  "Một Senior không phải người BIẾT NHIỀU framework,          ║
║   mà là người HIỂU SÂU nền tảng."                           ║
║                                                               ║
║  Framework thay đổi mỗi 2-3 năm.                            ║
║  Nền tảng CS KHÔNG BAO GIỜ thay đổi.                        ║
╚═══════════════════════════════════════════════════════════════╝
```

### 0.1 Cách máy tính hoạt động

> 📖 [**Deep Dive →**](./phase-0-cs-fundamentals/0.1-Cach-May-Tinh-Hoat-Dong-Deep-Dive.md)

```
    ┌─────────────────────────────────────────────────┐
    │  BẠN CẦN HIỂU:                                 │
    │                                                  │
    │  1. CPU thực thi instructions như thế nào?       │
    │     → Fetch → Decode → Execute cycle            │
    │     → Tại sao CPU chỉ hiểu 0 và 1?             │
    │                                                  │
    │  2. RAM (Memory) hoạt động ra sao?              │
    │     → Stack vs Heap                             │
    │     → Tại sao truy cập Stack nhanh hơn Heap?   │
    │     → Memory alignment, padding                 │
    │                                                  │
    │  3. Disk (Storage) khác RAM ở đâu?              │
    │     → HDD vs SSD                                │
    │     → Tại sao đọc từ disk chậm gấp 1000x?     │
    │     → Sequential vs Random access               │
    │                                                  │
    │  4. Cache hierarchy                             │
    │     → L1, L2, L3 cache                          │
    │     → Cache line, cache miss                    │
    │     → Tại sao data locality quan trọng?         │
    └─────────────────────────────────────────────────┘

    Tốc độ truy cập (xấp xỉ):
    ┌───────────┬──────────────┬──────────────────────┐
    │ Cấp       │ Thời gian    │ So sánh              │
    ├───────────┼──────────────┼──────────────────────┤
    │ L1 Cache  │ ~1 ns        │ Lấy sách trên bàn   │
    │ L2 Cache  │ ~4 ns        │ Lấy sách trên kệ    │
    │ L3 Cache  │ ~10 ns       │ Đi đến tủ sách      │
    │ RAM       │ ~100 ns      │ Đi ra thư viện      │
    │ SSD       │ ~100,000 ns  │ Bay sang thành phố   │
    │ HDD       │ ~10,000,000  │ Bay vòng quanh TG   │
    │ Network   │ ~ms-s        │ Gửi thư quốc tế     │
    └───────────┴──────────────┴──────────────────────┘
```

### 0.2 Hệ điều hành (OS Fundamentals)

```
    ┌─────────────────────────────────────────────────┐
    │  BẠN CẦN HIỂU:                                 │
    │                                                  │
    │  1. Process & Thread                            │
    │     → Process là gì? Thread là gì?              │
    │     → Context switching tốn bao nhiêu?          │
    │     → Tại sao Go dùng Goroutine thay Thread?   │
    │                                                  │
    │  2. Memory Management                           │
    │     → Virtual Memory, Page Table                │
    │     → malloc/free, Garbage Collection            │
    │     → Memory leak xảy ra khi nào?               │
    │                                                  │
    │  3. File System                                 │
    │     → File descriptor, inode                    │
    │     → Buffered vs Unbuffered I/O                │
    │     → Tại sao "everything is a file" trên Unix? │
    │                                                  │
    │  4. I/O Models                                  │
    │     → Blocking vs Non-blocking                  │
    │     → Synchronous vs Asynchronous               │
    │     → epoll / kqueue / io_uring                 │
    │     → Multiplexing (select, poll)               │
    └─────────────────────────────────────────────────┘
```

### 0.3 Data Structures & Algorithms

```
╔═══════════════════════════════════════════════════════════╗
║  KHÔNG CẦN giải 1000 bài LeetCode!                      ║
║  CHỈ CẦN HIỂU SÂU các cấu trúc dữ liệu cốt lõi       ║
║  và BIẾT KHI NÀO dùng cái nào.                          ║
╚═══════════════════════════════════════════════════════════╝

    Mức độ ưu tiên:

    ★★★★★ PHẢI BIẾT (dùng hàng ngày):
    ┌─────────────────────────────────────────────────┐
    │  • Array / Slice      → Contiguous memory       │
    │  • Hash Map           → O(1) lookup             │
    │  • Linked List        → Dynamic size            │
    │  • Stack / Queue      → LIFO / FIFO             │
    │  • Tree (Binary, BST) → Hierarchical data       │
    └─────────────────────────────────────────────────┘

    ★★★★ NÊN BIẾT (dùng thường xuyên):
    ┌─────────────────────────────────────────────────┐
    │  • Heap / Priority Queue → Scheduling           │
    │  • Graph (BFS, DFS)      → Dependencies         │
    │  • Trie                  → Autocomplete          │
    └─────────────────────────────────────────────────┘

    ★★★ BIẾT THÌ TỐT (Senior nên biết):
    ┌─────────────────────────────────────────────────┐
    │  • B-Tree / B+ Tree   → Database indexes        │
    │  • Skip List          → Redis internals         │
    │  • Bloom Filter       → Probabilistic check     │
    │  • Consistent Hashing → Distributed systems     │
    └─────────────────────────────────────────────────┘

    Algorithms cần nắm:
    ┌─────────────────────────────────────────────────┐
    │  • Sorting      → Quick, Merge, Heap sort       │
    │  • Searching    → Binary search                 │
    │  • Recursion    → Call stack, base case          │
    │  • Big-O        → Time & Space complexity       │
    │  • Two Pointers → Sliding window                │
    │  • Dynamic Prog → Memoization, tabulation       │
    └─────────────────────────────────────────────────┘
```

### 0.4 Tài nguyên học Phase 0

```
    ┌─────────────────────────────────────────────────────┐
    │  📖 Sách:                                           │
    │  • "Computer Systems: A Programmer's Perspective"  │
    │    (CS:APP) — Hiểu máy tính từ gốc                 │
    │  • "Operating Systems: Three Easy Pieces" (OSTEP)  │
    │    — OS miễn phí, dễ hiểu                          │
    │  • "Grokking Algorithms" — DSA cho người mới       │
    │                                                     │
    │  🎬 Video:                                          │
    │  • MIT 6.004 — Computer Architecture               │
    │  • MIT 6.S081 — Operating Systems                  │
    │  • Abdul Bari — Algorithms (YouTube)               │
    └─────────────────────────────────────────────────────┘
```

---

## Phase 1: Ngôn Ngữ Lập Trình (Golang)

### Tại sao chọn Go cho Backend?

```
╔═══════════════════════════════════════════════════════════════╗
║  Go được thiết kế BỞI và CHO backend engineer:               ║
║                                                               ║
║  ✅ Compile nhanh     → Feedback loop ngắn                   ║
║  ✅ Static typing     → Bắt bug lúc compile                  ║
║  ✅ Built-in concurrency → Goroutine + Channel               ║
║  ✅ Standard library mạnh → net/http, database/sql           ║
║  ✅ Single binary     → Deploy đơn giản                      ║
║  ✅ Cross-compile     → Build cho mọi OS                     ║
║                                                               ║
║  Dùng bởi: Google, Uber, Docker, Kubernetes, CrowdStrike    ║
╚═══════════════════════════════════════════════════════════════╝
```

### 1.1 Fundamentals (Tuần 1–4)

> 📖 **Deep Dive Documents:**
>
> - [Ultimate Go — Triết Lý & Nguyên Tắc](./Ultimate-Go-Deep-Dive.md)
> - [Language Design Philosophy](./Language-Design-Philosophy-Deep-Dive.md)
> - [Simplicity Is Complicated](./Simplicity-Is-Complicated-Deep-Dive.md)
> - [Variables](./Variables-Deep-Dive.md)
> - [Stacks & Pointers](./Stacks-And-Pointers-Deep-Dive.md)
> - [Escape Analysis](./Escape-Analysis-Deep-Dive.md)
> - [Memory Profiling](./Memory-Profiling-Deep-Dive.md)
> - [Data & Semantics](./Data-And-Semantics-Deep-Dive.md)
> - [Methods, Interfaces & Embedded Types](./Methods-Interfaces-Embedded-Types-Deep-Dive.md)
> - [Composition With Go](./Composition-With-Go-Deep-Dive.md)
> - [Interface Semantics](./Interface-Semantics-Deep-Dive.md)
> - [Interface Values Are Valueless](./Interface-Values-Are-Valueless-Deep-Dive.md)
> - [Avoid Interface Pollution](./Avoid-Interface-Pollution-Deep-Dive.md)
> - [Reducing Type Hierarchies](./Reducing-Type-Hierarchies-Deep-Dive.md)
> - [Data-Oriented Design](./Data-Oriented-Design-Deep-Dive.md)
> - [Efficiency, Performance & Data Structures](./Efficiency-Performance-Data-Structures-Deep-Dive.md)

```
    Học theo thứ tự này:

    ┌─────────────────────────────────────────────────┐
    │  Week 1: Syntax & Types                         │
    │  ├── Variables, Constants, iota                 │
    │  ├── Basic types (int, float, string, bool)     │
    │  ├── Composite types (array, slice, map, struct)│
    │  ├── Zero values — Tại sao Go có zero values?  │
    │  └── Type conversion vs Type assertion          │
    │                                                  │
    │  Week 2: Control Flow & Functions               │
    │  ├── if/else, switch, for                       │
    │  ├── Functions: params, returns, variadic       │
    │  ├── Named returns, defer                       │
    │  ├── Error handling (error interface)            │
    │  └── Closures & Anonymous functions             │
    │                                                  │
    │  Week 3: Data Semantics                         │
    │  ├── Value semantics vs Pointer semantics       │
    │  ├── Stack vs Heap allocation                   │
    │  ├── Escape analysis                            │
    │  ├── Methods (value receiver vs pointer recv)   │
    │  └── Interfaces & Polymorphism                  │
    │                                                  │
    │  Week 4: Packages & Testing                     │
    │  ├── Package design, import/export              │
    │  ├── go mod, dependency management              │
    │  ├── Writing tests (testing package)            │
    │  ├── Table-driven tests                         │
    │  └── Benchmarks & Profiling basics              │
    └─────────────────────────────────────────────────┘
```

### 1.2 Intermediate (Tuần 5–8)

> 📖 **Deep Dive Documents:**
>
> - [Scheduling in Go — Part 1: OS Scheduler](./Scheduling-In-Go-Part1-OS-Scheduler-Deep-Dive.md)
> - [Scheduling in Go — Part 2: Go Scheduler](./Scheduling-In-Go-Part2-Go-Scheduler-Deep-Dive.md)
> - [Scheduling in Go — Part 3: Concurrency](./Scheduling-In-Go-Part3-Concurrency-Deep-Dive.md)
> - [Concurrency Is Not Parallelism](./Concurrency-Is-Not-Parallelism-Deep-Dive.md)
> - [Unsafe Benchmarking](./Unsafe-Benchmarking-Deep-Dive.md)

```
    ┌─────────────────────────────────────────────────┐
    │  Week 5-6: Concurrency                          │
    │  ├── Goroutines — lightweight threads           │
    │  ├── Channels — typed, directional              │
    │  ├── Select statement                           │
    │  ├── sync package (Mutex, WaitGroup, Once)      │
    │  ├── Context — cancellation, timeout, values    │
    │  ├── Race conditions & go race detector         │
    │  └── Concurrency patterns:                      │
    │       • Fan-in / Fan-out                        │
    │       • Worker pool                             │
    │       • Pipeline                                │
    │       • Semaphore                               │
    │                                                  │
    │  Week 7-8: Advanced Topics                      │
    │  ├── Generics (Go 1.18+)                        │
    │  ├── Reflection (reflect package)               │
    │  ├── unsafe package — khi nào dùng?             │
    │  ├── Code generation (go generate)              │
    │  ├── Build tags & cross-compilation             │
    │  └── Profiling (pprof, trace)                   │
    └─────────────────────────────────────────────────┘
```

### 1.3 Tài nguyên Phase 1

```
    ┌─────────────────────────────────────────────────────┐
    │  📖 Sách / Khóa học:                                │
    │  • "The Go Programming Language" (Donovan & K.)    │
    │  • "Ultimate Go" — Ardan Labs (Bill Kennedy)       │
    │  • "Concurrency in Go" (Katherine Cox-Buday)       │
    │  • Go Tour (tour.golang.org) — Interactive         │
    │  • Go by Example (gobyexample.com)                 │
    │                                                     │
    │  🔧 Thực hành:                                      │
    │  • Exercism.io — Go track                          │
    │  • Build CLI tools (file parser, HTTP client)      │
    └─────────────────────────────────────────────────────┘
```

---

## Phase 2: Networking & Protocols

### Tại sao học Networking TRƯỚC Database?

```
    ┌─────────────────────────────────────────────────────┐
    │  Backend = Máy tính NÓI CHUYỆN với nhau qua mạng   │
    │                                                      │
    │  Client ◄──── Network ────► Server ◄──── ► DB       │
    │                                                      │
    │  Nếu không hiểu networking:                         │
    │  ❌ Không hiểu tại sao API chậm                     │
    │  ❌ Không debug được connection issues               │
    │  ❌ Không thiết kế được distributed system           │
    │  ❌ Không hiểu database connection pooling           │
    └─────────────────────────────────────────────────────┘
```

### 2.1 Mô hình OSI / TCP-IP (High-Level → Low-Level)

```
    ┌──────────────────────────────────────────────────────┐
    │  Layer 7 — Application   │ HTTP, gRPC, WebSocket    │
    │  Layer 6 — Presentation  │ TLS/SSL, Encoding        │
    │  Layer 5 — Session       │ Session management       │
    │──────────────────────────│──────────────────────────│
    │  Layer 4 — Transport     │ TCP, UDP                 │
    │  Layer 3 — Network       │ IP, Routing              │
    │  Layer 2 — Data Link     │ Ethernet, MAC            │
    │  Layer 1 — Physical      │ Cables, Signals          │
    └──────────────────────────────────────────────────────┘

    Backend Engineer cần tập trung:
    ★★★★★ Layer 7 (HTTP, gRPC)    → Hàng ngày
    ★★★★★ Layer 4 (TCP/UDP)       → Debug, performance
    ★★★★  Layer 6 (TLS)           → Security
    ★★★   Layer 3 (IP, DNS)       → Infrastructure
```

### 2.2 TCP Deep Dive

```
    ┌─────────────────────────────────────────────────┐
    │  BẠN CẦN HIỂU THẬT SÂU:                        │
    │                                                  │
    │  1. Three-way Handshake (SYN → SYN-ACK → ACK)  │
    │     → Tại sao cần 3 bước?                       │
    │     → Chi phí của mỗi connection                │
    │                                                  │
    │  2. Reliability                                  │
    │     → Sequence numbers, ACK                     │
    │     → Retransmission                            │
    │     → Flow control (sliding window)             │
    │     → Congestion control (slow start, AIMD)     │
    │                                                  │
    │  3. Connection Lifecycle                         │
    │     → TIME_WAIT state — Tại sao tồn tại?       │
    │     → Connection pooling — Tại sao cần?         │
    │     → Keep-Alive                                │
    │                                                  │
    │  4. TCP vs UDP                                   │
    │     → Khi nào dùng TCP? (HTTP, DB connections)  │
    │     → Khi nào dùng UDP? (DNS, streaming, game)  │
    └─────────────────────────────────────────────────┘
```

### 2.3 HTTP Deep Dive

```
    ┌─────────────────────────────────────────────────┐
    │  HTTP/1.1 → HTTP/2 → HTTP/3                     │
    │                                                  │
    │  HTTP/1.1:                                       │
    │  ├── Request/Response model                     │
    │  ├── Methods: GET, POST, PUT, DELETE, PATCH     │
    │  ├── Status codes (2xx, 3xx, 4xx, 5xx)          │
    │  ├── Headers (Content-Type, Authorization...)   │
    │  ├── Cookies & Sessions                         │
    │  ├── Keep-Alive & Connection reuse              │
    │  └── Head-of-line blocking problem              │
    │                                                  │
    │  HTTP/2:                                         │
    │  ├── Binary framing (thay vì text)              │
    │  ├── Multiplexing (nhiều request trên 1 conn)   │
    │  ├── Header compression (HPACK)                 │
    │  ├── Server Push                                │
    │  └── Stream prioritization                      │
    │                                                  │
    │  HTTP/3 (QUIC):                                  │
    │  ├── Dựa trên UDP (thay vì TCP)                 │
    │  ├── 0-RTT connection setup                     │
    │  ├── Giải quyết head-of-line blocking hoàn toàn │
    │  └── Built-in TLS 1.3                           │
    └─────────────────────────────────────────────────┘
```

### 2.4 Security (TLS/HTTPS)

```
    ┌─────────────────────────────────────────────────┐
    │  1. TLS Handshake                               │
    │     → Certificate exchange                      │
    │     → Key exchange (Diffie-Hellman)              │
    │     → Symmetric encryption bắt đầu              │
    │                                                  │
    │  2. Certificates                                │
    │     → CA (Certificate Authority)                │
    │     → Self-signed vs CA-signed                  │
    │     → mTLS (mutual TLS) — service-to-service   │
    │                                                  │
    │  3. CORS, CSRF, XSS                             │
    │     → Same-Origin Policy                        │
    │     → Cách phòng tránh                          │
    └─────────────────────────────────────────────────┘
```

### 2.5 DNS

```
    ┌─────────────────────────────────────────────────┐
    │  DNS Resolution Flow:                            │
    │                                                  │
    │  Browser → Local Cache → OS Cache               │
    │    → Recursive Resolver → Root → TLD → Auth     │
    │                                                  │
    │  BẠN CẦN BIẾT:                                   │
    │  • A, AAAA, CNAME, MX, TXT records              │
    │  • TTL (Time To Live)                            │
    │  • DNS-based load balancing                      │
    │  • DNS propagation delay                         │
    └─────────────────────────────────────────────────┘
```

---

## Phase 3: Database & Data Storage

### 3.1 Relational Database (PostgreSQL)

```
╔═══════════════════════════════════════════════════════════════╗
║  Chọn PostgreSQL làm database chính để học.                  ║
║  Lý do: Feature-rich, ACID compliant, industry standard.     ║
╚═══════════════════════════════════════════════════════════════╝

    Thứ tự học:

    ★★★★★ LEVEL 1 — SQL Fundamentals:
    ┌─────────────────────────────────────────────────┐
    │  • CREATE, INSERT, SELECT, UPDATE, DELETE       │
    │  • WHERE, ORDER BY, GROUP BY, HAVING            │
    │  • JOINs (INNER, LEFT, RIGHT, FULL)             │
    │  • Subqueries, CTEs (WITH)                      │
    │  • Aggregations (COUNT, SUM, AVG, MAX, MIN)     │
    │  • UNION, INTERSECT, EXCEPT                     │
    └─────────────────────────────────────────────────┘

    ★★★★★ LEVEL 2 — Schema Design:
    ┌─────────────────────────────────────────────────┐
    │  • Normalization (1NF, 2NF, 3NF, BCNF)         │
    │  • Denormalization — Khi nào và tại sao?        │
    │  • Primary Key, Foreign Key, Unique, Check      │
    │  • Data types (SERIAL, UUID, JSONB, TIMESTAMP)  │
    │  • Migrations — quản lý schema changes          │
    └─────────────────────────────────────────────────┘

    ★★★★★ LEVEL 3 — Performance & Internals:
    ┌─────────────────────────────────────────────────┐
    │  • Indexes (B-Tree, Hash, GIN, GiST)            │
    │  • EXPLAIN ANALYZE — đọc query plan             │
    │  • Index scan vs Sequential scan                │
    │  • Connection pooling (PgBouncer)               │
    │  • Vacuum, Autovacuum, MVCC                     │
    │  • Table bloat, Dead tuples                     │
    └─────────────────────────────────────────────────┘

    ★★★★ LEVEL 4 — Transactions & Consistency:
    ┌─────────────────────────────────────────────────┐
    │  • ACID properties (chi tiết từng cái)          │
    │  • Isolation levels:                            │
    │    Read Uncommitted → Read Committed            │
    │    → Repeatable Read → Serializable             │
    │  • Locking: Row-level, Table-level              │
    │  • Deadlocks — phát hiện & giải quyết           │
    │  • Optimistic vs Pessimistic locking            │
    └─────────────────────────────────────────────────┘
```

### 3.2 NoSQL Databases

```
    ┌─────────────────────────────────────────────────────────┐
    │  LOẠI           │ ĐẠI DIỆN    │ KHI NÀO DÙNG?         │
    ├─────────────────┼─────────────┼────────────────────────┤
    │ Key-Value       │ Redis       │ Caching, session,      │
    │                 │             │ rate limiting           │
    ├─────────────────┼─────────────┼────────────────────────┤
    │ Document        │ MongoDB     │ Flexible schema,       │
    │                 │             │ content management     │
    ├─────────────────┼─────────────┼────────────────────────┤
    │ Column-Family   │ Cassandra   │ Time-series, IoT,      │
    │                 │             │ write-heavy            │
    ├─────────────────┼─────────────┼────────────────────────┤
    │ Graph           │ Neo4j       │ Social networks,       │
    │                 │             │ recommendations        │
    ├─────────────────┼─────────────┼────────────────────────┤
    │ Search Engine   │ Elasticsearch│ Full-text search,     │
    │                 │             │ log aggregation        │
    └─────────────────┴─────────────┴────────────────────────┘

    ⚠️ QUAN TRỌNG:
    "Đừng chọn NoSQL vì nó 'trendy'.
     Chọn vì nó GIẢI QUYẾT vấn đề mà SQL KHÔNG giải quyết tốt."
```

### 3.3 Redis Deep Dive

```
    ┌─────────────────────────────────────────────────┐
    │  Redis là PHẢI BIẾT cho Backend Engineer:        │
    │                                                  │
    │  1. Data Structures                             │
    │     → String, List, Set, Sorted Set, Hash       │
    │     → HyperLogLog, Stream, Bitmap               │
    │                                                  │
    │  2. Use Cases                                    │
    │     → Caching (Cache-Aside, Write-Through)      │
    │     → Session storage                           │
    │     → Rate limiting                             │
    │     → Distributed locks (Redlock)               │
    │     → Pub/Sub messaging                         │
    │     → Leaderboards (Sorted Set)                 │
    │                                                  │
    │  3. Internals                                    │
    │     → Single-threaded event loop                │
    │     → Persistence (RDB, AOF)                    │
    │     → Eviction policies (LRU, LFU, TTL)         │
    │     → Cluster mode, Sentinel                    │
    │                                                  │
    │  4. Gotchas                                      │
    │     → Cache invalidation strategies             │
    │     → Thundering herd problem                   │
    │     → Hot key problem                           │
    └─────────────────────────────────────────────────┘
```

---

## Phase 4: API Design & Web Server

### 4.1 RESTful API Design

> 📖 **Deep Dive Documents:**
>
> - [Application-Focused API Design](./Application-Focused-API-Design-Deep-Dive.md)
> - [Infrastructure Software](./Infrastructure-Software-Deep-Dive.md)

```
╔═══════════════════════════════════════════════════════════════╗
║  REST không phải framework — REST là KIẾN TRÚC.              ║
║  Hiểu đúng REST = thiết kế API mà developer MUỐN dùng.      ║
╚═══════════════════════════════════════════════════════════════╝

    Nguyên tắc thiết kế:

    ┌─────────────────────────────────────────────────┐
    │  1. Resource-based URLs (danh từ, không động từ)│
    │     ✅ GET  /users/123                          │
    │     ❌ GET  /getUser?id=123                     │
    │                                                  │
    │  2. HTTP Methods = Hành động                    │
    │     GET    → Đọc (idempotent)                   │
    │     POST   → Tạo mới                            │
    │     PUT    → Cập nhật toàn bộ (idempotent)      │
    │     PATCH  → Cập nhật một phần                  │
    │     DELETE → Xóa (idempotent)                   │
    │                                                  │
    │  3. Status Codes có nghĩa                       │
    │     200 OK, 201 Created, 204 No Content         │
    │     400 Bad Request, 401 Unauthorized           │
    │     403 Forbidden, 404 Not Found                │
    │     409 Conflict, 422 Unprocessable             │
    │     429 Too Many Requests                       │
    │     500 Internal Server Error                   │
    │                                                  │
    │  4. Pagination, Filtering, Sorting              │
    │     GET /users?page=2&limit=20&sort=-created_at │
    │                                                  │
    │  5. Versioning                                   │
    │     /api/v1/users  hoặc  Header: Accept-Version │
    │                                                  │
    │  6. Error Response chuẩn                         │
    │     { "error": { "code": "...", "message": "" }}│
    └─────────────────────────────────────────────────┘
```

### 4.2 gRPC & Protocol Buffers

```
    ┌─────────────────────────────────────────────────┐
    │  gRPC = Google Remote Procedure Call             │
    │                                                  │
    │  Khi nào dùng REST vs gRPC?                     │
    │                                                  │
    │  REST:                                           │
    │  ✅ Public APIs (client/browser gọi)            │
    │  ✅ Simple CRUD                                  │
    │  ✅ Cần human-readable (JSON)                   │
    │                                                  │
    │  gRPC:                                           │
    │  ✅ Service-to-service communication            │
    │  ✅ High performance (binary, HTTP/2)            │
    │  ✅ Streaming (unary, server, client, bidi)     │
    │  ✅ Strong typing (Protobuf schema)             │
    │  ✅ Code generation tự động                     │
    │                                                  │
    │  BẠN CẦN BIẾT:                                   │
    │  • Protobuf syntax (proto3)                     │
    │  • Service definition                           │
    │  • Interceptors (= middleware của gRPC)          │
    │  • Error handling (status codes)                │
    │  • Deadline / Timeout                           │
    │  • Load balancing (client-side vs proxy)        │
    └─────────────────────────────────────────────────┘
```

### 4.3 Authentication & Authorization

```
    ┌─────────────────────────────────────────────────┐
    │  Authentication (Xác thực: BẠN LÀ AI?)          │
    │  ├── Session-based (cookie + server state)      │
    │  ├── Token-based (JWT — stateless)              │
    │  ├── OAuth 2.0 / OpenID Connect                 │
    │  └── API Keys                                    │
    │                                                  │
    │  Authorization (Phân quyền: BẠN ĐƯỢC LÀM GÌ?)  │
    │  ├── RBAC (Role-Based Access Control)           │
    │  ├── ABAC (Attribute-Based)                     │
    │  └── Policy engines (OPA, Casbin)               │
    │                                                  │
    │  JWT Deep Dive:                                  │
    │  ├── Header.Payload.Signature                   │
    │  ├── Access Token vs Refresh Token              │
    │  ├── Token rotation strategy                    │
    │  ├── Token revocation (blacklist/whitelist)     │
    │  └── ⚠️ JWT KHÔNG phải session replacement,     │
    │        hiểu trade-offs!                          │
    └─────────────────────────────────────────────────┘
```

### 4.4 Middleware & Server Architecture

```
    ┌─────────────────────────────────────────────────┐
    │  Request lifecycle trong Go HTTP server:         │
    │                                                  │
    │  Request → Router → Middleware Chain → Handler  │
    │                                                  │
    │  Middleware phổ biến:                            │
    │  ├── Logging (structured logging)               │
    │  ├── Authentication                             │
    │  ├── Rate Limiting                              │
    │  ├── CORS                                        │
    │  ├── Request ID (tracing)                       │
    │  ├── Recovery (panic handling)                   │
    │  ├── Compression (gzip)                          │
    │  └── Timeout                                     │
    │                                                  │
    │  Project Structure (Clean / Hexagonal):          │
    │  ├── cmd/          → entrypoints                │
    │  ├── internal/                                   │
    │  │   ├── handler/  → HTTP/gRPC handlers         │
    │  │   ├── service/  → business logic             │
    │  │   ├── repo/     → data access                │
    │  │   └── model/    → domain entities            │
    │  ├── pkg/          → shared libraries           │
    │  └── migrations/   → DB migrations              │
    └─────────────────────────────────────────────────┘
```

---

## Phase 5: Software Design Patterns (BIG TECH MUST-KNOW)

### Tại sao Phase này quan trọng cho Big Tech?

```
╔═══════════════════════════════════════════════════════════════╗
║  Big Tech Senior được đánh giá qua khả năng THIẾT KẾ CODE   ║
║  không chỉ "code chạy được" mà phải:                         ║
║  • Extensible (mở rộng dễ)                                   ║
║  • Testable (test dễ)                                         ║
║  • Maintainable (bảo trì dễ)                                 ║
║                                                               ║
║  Google, Uber, Stripe đều hỏi sâu về design patterns        ║
║  trong coding interview và system design interview.           ║
╚═══════════════════════════════════════════════════════════════╝
```

### 5.1 SOLID Principles (Áp dụng trong Go)

> 📖 **Deep Dive Documents:**
>
> - [Design Philosophy — Integrity](./Design-Philosophy-Integrity-Deep-Dive.md)
> - [Style Guide Philosophy](./Style-Guide-Philosophy-Deep-Dive.md)
> - [Code Must Never Lie](./Code-Must-Never-Lie-Deep-Dive.md)
> - [Psychology of Code Readability](./Psychology-Code-Readability-Deep-Dive.md)
> - [Miller's Law](./Millers-Law-Deep-Dive.md)
> - [Prototype Your Design](./Prototype-Your-Design-Deep-Dive.md)
> - [Technical Debt Quadrant](./Technical-Debt-Quadrant-Deep-Dive.md)
> - [Bugs Per LOC](./Bugs-Per-LOC-Deep-Dive.md)
> - [Normalization of Deviance](./Normalization-of-Deviance-Deep-Dive.md)
> - [Reading Postmortems](./Reading-Postmortems-Deep-Dive.md)

```
    ┌─────────────────────────────────────────────────────┐
    │  S — Single Responsibility                          │
    │     → Mỗi struct/package CHỈ làm MỘT việc          │
    │     → Go: Mỗi package có 1 mục đích rõ ràng       │
    │     → Anti-pattern: "god package" chứa mọi thứ    │
    │                                                      │
    │  O — Open/Closed                                    │
    │     → Mở cho extension, đóng cho modification      │
    │     → Go: Dùng interfaces + composition            │
    │     → Thêm behavior MỚI mà KHÔNG sửa code cũ     │
    │                                                      │
    │  L — Liskov Substitution                            │
    │     → Go: Interface compliance                      │
    │     → Mọi implementation phải thay thế được        │
    │     → Testing: Mock phải behave giống real          │
    │                                                      │
    │  I — Interface Segregation                          │
    │     → Go: "The bigger the interface, the weaker     │
    │            the abstraction" — Rob Pike              │
    │     → io.Reader (1 method) > io.ReadWriteCloser    │
    │     → Accept interfaces, return structs            │
    │                                                      │
    │  D — Dependency Inversion                           │
    │     → High-level modules không phụ thuộc low-level │
    │     → Cả hai phụ thuộc vào ABSTRACTION             │
    │     → Go: Inject dependencies qua constructor      │
    └─────────────────────────────────────────────────────┘
```

### 5.2 Domain-Driven Design (DDD)

```
    ┌─────────────────────────────────────────────────────┐
    │  DDD — Tại sao Big Tech dùng?                       │
    │  → Giúp model business logic PHỨC TẠP              │
    │  → Ubiquitous Language: Dev và Business nói         │
    │    CÙNG ngôn ngữ                                    │
    │                                                      │
    │  Strategic Patterns:                                │
    │  ├── Bounded Context                                │
    │  │   → Ranh giới rõ ràng giữa các domains          │
    │  │   → Mỗi context có model RIÊNG                  │
    │  │   → Map trực tiếp sang microservices            │
    │  │                                                   │
    │  ├── Context Mapping                                │
    │  │   → Anti-Corruption Layer                        │
    │  │   → Shared Kernel                                │
    │  │   → Customer-Supplier                            │
    │  │                                                   │
    │  └── Aggregate                                      │
    │      → Cluster of domain objects                    │
    │      → Transaction boundary                         │
    │      → Aggregate Root = entry point duy nhất       │
    │                                                      │
    │  Tactical Patterns:                                 │
    │  ├── Entity (có identity)                           │
    │  ├── Value Object (immutable, no identity)          │
    │  ├── Repository (persistence abstraction)           │
    │  ├── Domain Service (logic không thuộc Entity)      │
    │  ├── Domain Event (thông báo thay đổi)             │
    │  └── Factory (complex object creation)              │
    └─────────────────────────────────────────────────────┘
```

### 5.3 Architecture Patterns

```
    ┌─────────────────────────────────────────────────────┐
    │  1. Clean Architecture / Hexagonal                  │
    │     ┌─────────────────────────────────────┐         │
    │     │         Frameworks & Drivers        │         │
    │     │  ┌───────────────────────────────┐  │         │
    │     │  │     Interface Adapters        │  │         │
    │     │  │  ┌─────────────────────────┐  │  │         │
    │     │  │  │    Application Layer    │  │  │         │
    │     │  │  │  ┌───────────────────┐  │  │  │         │
    │     │  │  │  │  Domain / Entity  │  │  │  │         │
    │     │  │  │  └───────────────────┘  │  │  │         │
    │     │  │  └─────────────────────────┘  │  │         │
    │     │  └───────────────────────────────┘  │         │
    │     └─────────────────────────────────────┘         │
    │     → Dependencies point INWARD                    │
    │     → Domain KHÔNG biết về DB, HTTP, etc.          │
    │                                                      │
    │  2. CQRS (Command Query Responsibility Segreg.)    │
    │     → Tách read model và write model               │
    │     → Write: normalized, consistency               │
    │     → Read: denormalized, performance              │
    │                                                      │
    │  3. Event Sourcing                                  │
    │     → Store EVENTS, không store state              │
    │     → Replay events = rebuild state                │
    │     → Audit trail miễn phí                         │
    │     → Dùng ở: Banking, E-commerce, Audit systems  │
    │                                                      │
    │  4. Strangler Fig Pattern                           │
    │     → Migrate monolith → microservices từ từ       │
    │     → Không big-bang rewrite                       │
    └─────────────────────────────────────────────────────┘
```

### 5.4 Design Patterns trong Go

```
    ┌─────────────────────────────────────────────────────┐
    │  ★★★★★ Creational:                                  │
    │  ├── Functional Options Pattern (Go-idiomatic)     │
    │  ├── Builder Pattern                               │
    │  └── Factory Pattern                               │
    │                                                      │
    │  ★★★★★ Structural:                                  │
    │  ├── Decorator (middleware = decorator!)            │
    │  ├── Adapter (anti-corruption layer)               │
    │  └── Facade (simplify complex subsystems)          │
    │                                                      │
    │  ★★★★★ Behavioral:                                  │
    │  ├── Strategy (swap algorithms via interfaces)     │
    │  ├── Observer (event-driven, pub/sub)              │
    │  ├── Chain of Responsibility (middleware chain)    │
    │  └── Command (encapsulate requests as objects)     │
    │                                                      │
    │  ★★★★★ Concurrency (Go-specific):                   │
    │  ├── Fan-In / Fan-Out                              │
    │  ├── Pipeline                                       │
    │  ├── Worker Pool                                    │
    │  ├── Semaphore (via buffered channel)              │
    │  ├── Error Group (golang.org/x/sync/errgroup)     │
    │  └── Context-based cancellation propagation        │
    └─────────────────────────────────────────────────────┘
```

---

## Phase 6: Advanced Testing & Quality (BIG TECH MUST-KNOW)

### Tại sao testing cực kỳ quan trọng ở Big Tech?

```
╔═══════════════════════════════════════════════════════════════╗
║  Google: "Nếu không có test, code không tồn tại."            ║
║  → 80%+ code coverage là MINIMUM                             ║
║  → Testing là KỸ NĂNG, không phải "task phụ"                ║
║                                                               ║
║  Senior Big Tech = viết test TỐT HƠN viết code              ║
╚═══════════════════════════════════════════════════════════════╝
```

### 6.1 Testing Pyramid

```
    ┌─────────────────────────────────────────────────────┐
    │                    ╱╲                                 │
    │                   ╱  ╲    E2E Tests                  │
    │                  ╱    ╲   (ít, chậm, đắt)           │
    │                 ╱──────╲                              │
    │                ╱        ╲  Integration Tests         │
    │               ╱          ╲ (vừa phải)                │
    │              ╱────────────╲                           │
    │             ╱              ╲ Unit Tests               │
    │            ╱                ╲ (nhiều, nhanh, rẻ)     │
    │           ╱══════════════════╲                        │
    │                                                      │
    │  Unit Tests (70%):                                   │
    │  ├── Table-driven tests (Go idiom)                  │
    │  ├── Test edge cases, error paths                   │
    │  ├── Mock dependencies (interfaces)                 │
    │  ├── Golden files (snapshot testing)                │
    │  └── Benchmarks (testing.B)                         │
    │                                                      │
    │  Integration Tests (20%):                            │
    │  ├── Database tests (testcontainers-go)             │
    │  ├── API tests (httptest package)                   │
    │  ├── Message queue tests                            │
    │  └── External service tests                         │
    │                                                      │
    │  E2E Tests (10%):                                    │
    │  ├── Full flow tests                                │
    │  ├── Contract tests (consumer-driven)               │
    │  └── Smoke tests in production                      │
    └─────────────────────────────────────────────────────┘
```

### 6.2 Advanced Testing Techniques (Senior Level)

```
    ┌─────────────────────────────────────────────────────┐
    │  1. Property-Based Testing                          │
    │     → Thay vì test case cụ thể, test THUỘC TÍNH   │
    │     → Go: gopter, rapid                            │
    │     → VD: "Sort output luôn có len = input len"    │
    │                                                      │
    │  2. Fuzzing (Go 1.18+ built-in)                     │
    │     → Tự động generate random input                │
    │     → Tìm crash, panic, edge cases                 │
    │     → go test -fuzz=FuzzFunctionName               │
    │     → ĐÃ phát hiện bugs THẬT trong Go stdlib      │
    │                                                      │
    │  3. Mutation Testing                                │
    │     → Tự động thay đổi code, test phải FAIL       │
    │     → Nếu test PASS khi code bị mutate = test YẾU │
    │     → Đo chất lượng THỰC SỰ của test suite        │
    │                                                      │
    │  4. Chaos Engineering                               │
    │     → Inject failures vào production               │
    │     → Netflix Chaos Monkey philosophy              │
    │     → Test: network partition, disk full, OOM      │
    │     → Tools: Litmus, Chaos Mesh (K8s)              │
    │     → "Nếu bạn không test failure, failure test bạn"│
    │                                                      │
    │  5. Load & Stress Testing                           │
    │     → Tools: k6, vegeta (Go-based), wrk            │
    │     → Latency percentiles (p50, p95, p99, p99.9)   │
    │     → Saturation testing (tìm breaking point)      │
    │     → Soak testing (long-running stability)        │
    │                                                      │
    │  6. Contract Testing                                │
    │     → Consumer-driven contracts (Pact)             │
    │     → API compatibility testing                     │
    │     → Schema evolution testing (Protobuf/Avro)     │
    └─────────────────────────────────────────────────────┘
```

### 6.3 Code Quality & Static Analysis

```
    ┌─────────────────────────────────────────────────────┐
    │  Tools PHẢI dùng:                                    │
    │  ├── golangci-lint (meta-linter, 50+ linters)      │
    │  ├── staticcheck (advanced static analysis)        │
    │  ├── go vet (built-in, detect subtle bugs)         │
    │  ├── gosec (security scanner)                       │
    │  ├── govulncheck (vulnerability scanner)            │
    │  └── gofumpt (stricter gofmt)                      │
    │                                                      │
    │  Practices:                                         │
    │  ├── Pre-commit hooks                              │
    │  ├── CI gate (block merge if lint fails)           │
    │  ├── Code coverage thresholds (80%+)               │
    │  ├── Cyclomatic complexity limits                  │
    │  └── Dependency audit (go mod tidy, verify)        │
    └─────────────────────────────────────────────────────┘
```

---

## Phase 7: System Design & Architecture

### Tại sao Phase này là THEN CHỐT cho Big Tech Senior?

```
╔═══════════════════════════════════════════════════════════════╗
║  System Design Interview chiếm 50% quyết định hire/no-hire  ║
║  tại Google, Meta, Amazon, Uber, Stripe.                     ║
║                                                               ║
║  Ở level Senior (L5/L6):                                     ║
║  → Bạn phải LEAD system design cho team                     ║
║  → Bạn phải DEFEND trade-offs trước Staff Engineers          ║
║  → Bạn phải ANTICIPATE failure scenarios                     ║
╚═══════════════════════════════════════════════════════════════╝
```

### 7.1 Kiến trúc ứng dụng

```
╔═══════════════════════════════════════════════════════════════╗
║  Đây là phase PHÂN BIỆT Junior và Senior.                    ║
║  Junior viết code chạy được.                                 ║
║  Senior THIẾT KẾ SYSTEM chạy tốt ở scale lớn.               ║
╚═══════════════════════════════════════════════════════════════╝

    Học theo thứ tự:

    1. Monolith → Modular Monolith → Microservices
    ┌──────────────────────────────────────────────────┐
    │  Monolith:                                       │
    │  ✅ Đơn giản, dễ deploy, dễ debug               │
    │  ❌ Scale khó, coupling cao                      │
    │                                                   │
    │  Modular Monolith:                               │
    │  ✅ Monolith nhưng tách module rõ ràng           │
    │  ✅ Dễ chuyển sang microservices sau             │
    │  → ĐÂY LÀ LỰA CHỌN TỐT NHẤT CHO 90% dự án!  │
    │                                                   │
    │  Microservices:                                   │
    │  ✅ Scale độc lập, team độc lập                  │
    │  ❌ Phức tạp: network, data consistency, debug   │
    │  → CHỈ dùng khi THẬT SỰ cần                     │
    └──────────────────────────────────────────────────┘
```

### 7.2 Message Queue & Event-Driven

```
    ┌─────────────────────────────────────────────────┐
    │  Message Queue (Kafka, RabbitMQ, NATS):          │
    │                                                  │
    │  Producer ──► Queue ──► Consumer                │
    │                                                  │
    │  Use cases:                                      │
    │  • Async processing (email, notification)       │
    │  • Decoupling services                          │
    │  • Load leveling (absorb traffic spikes)        │
    │  • Event sourcing                               │
    │                                                  │
    │  Kafka PHẢI BIẾT:                                │
    │  ├── Topics, Partitions, Offsets                │
    │  ├── Consumer Groups                            │
    │  ├── Exactly-once vs At-least-once delivery     │
    │  ├── Schema Registry (Avro/Protobuf)            │
    │  └── Log compaction                             │
    │                                                  │
    │  Patterns:                                       │
    │  • Pub/Sub vs Point-to-Point                    │
    │  • Saga Pattern (distributed transactions)      │
    │  • Outbox Pattern (transactional messaging)     │
    │  • CQRS (Command Query Responsibility Segreg.)  │
    │  • Event Sourcing                               │
    └─────────────────────────────────────────────────┘
```

### 7.3 Caching Strategies

```
    ┌─────────────────────────────────────────────────┐
    │  "Hai bài toán khó nhất trong CS:               │
    │   Cache invalidation và Naming things."          │
    │   — Phil Karlton                                │
    │                                                  │
    │  Caching Patterns:                               │
    │  ├── Cache-Aside (Lazy Loading)                 │
    │  │   App kiểm tra cache → miss → query DB      │
    │  │   → ghi vào cache → return                   │
    │  │                                               │
    │  ├── Write-Through                               │
    │  │   Ghi vào cache VÀ DB cùng lúc              │
    │  │                                               │
    │  ├── Write-Behind (Write-Back)                   │
    │  │   Ghi cache trước, async ghi DB sau          │
    │  │                                               │
    │  └── Read-Through                                │
    │      Cache tự query DB khi miss                 │
    │                                                  │
    │  Multi-level caching:                            │
    │  L1 (in-process) → L2 (Redis) → DB             │
    │                                                  │
    │  Invalidation strategies:                        │
    │  • TTL-based (simple nhưng stale)               │
    │  • Event-based (chính xác nhưng phức tạp)       │
    │  • Version-based (cache key chứa version)       │
    └─────────────────────────────────────────────────┘
```

### 7.4 Scalability Patterns

```
    ┌─────────────────────────────────────────────────┐
    │  Load Balancing:                                 │
    │  ├── Layer 4 (TCP) vs Layer 7 (HTTP)            │
    │  ├── Algorithms: Round Robin, Least Conn,       │
    │  │   IP Hash, Weighted                          │
    │  └── Tools: Nginx, HAProxy, Cloud LB            │
    │                                                  │
    │  Database Scaling:                               │
    │  ├── Read Replicas (scale reads)                │
    │  ├── Sharding (scale writes)                    │
    │  │   → Range-based, Hash-based, Directory-based│
    │  ├── Connection Pooling                         │
    │  └── Partitioning (table-level)                 │
    │                                                  │
    │  Application Scaling:                            │
    │  ├── Horizontal (thêm instances)                │
    │  ├── Vertical (thêm resource)                   │
    │  └── Auto-scaling (metrics-based)               │
    │                                                  │
    │  Distributed Patterns:                           │
    │  ├── CAP Theorem (hiểu trade-offs)              │
    │  ├── Consistent Hashing                         │
    │  ├── Circuit Breaker                            │
    │  ├── Retry with Exponential Backoff             │
    │  ├── Bulkhead Pattern                           │
    │  └── Rate Limiting (Token Bucket, Leaky Bucket) │
    └─────────────────────────────────────────────────┘
```

### 7.5 System Design Interview Checklist

```
    ┌─────────────────────────────────────────────────────┐
    │  Quy trình System Design Interview:                 │
    │                                                      │
    │  Step 1: CLARIFY Requirements (5 phút)              │
    │  ├── Functional vs Non-functional                   │
    │  ├── Scale: users, QPS, data size                   │
    │  └── Constraints: latency, availability              │
    │                                                      │
    │  Step 2: HIGH-LEVEL Design (10 phút)                │
    │  ├── Core components & data flow                    │
    │  ├── API design                                      │
    │  └── Database schema                                │
    │                                                      │
    │  Step 3: DEEP DIVE (20 phút)                        │
    │  ├── Bottleneck identification                      │
    │  ├── Scaling strategy                               │
    │  ├── Caching, queuing decisions                     │
    │  └── Trade-offs discussion                          │
    │                                                      │
    │  Step 4: WRAP UP (5 phút)                           │
    │  ├── Monitoring & alerting                          │
    │  ├── Failure scenarios                              │
    │  └── Future improvements                            │
    └─────────────────────────────────────────────────────┘

    Bài tập System Design phải làm:
    ┌─────────────────────────────────────────────────────┐
    │  ★★★★★ URL Shortener (TinyURL)                     │
    │  ★★★★★ Rate Limiter                                │
    │  ★★★★★ Chat System (WhatsApp)                      │
    │  ★★★★  Notification Service                        │
    │  ★★★★  News Feed (Facebook/Twitter)                │
    │  ★★★★  Distributed Cache                           │
    │  ★★★   Video Streaming (YouTube)                   │
    │  ★★★   Search Autocomplete                         │
    │  ★★★   Payment System                              │
    └─────────────────────────────────────────────────────┘
```

---

## Phase 8: Distributed Systems Deep Dive (BIG TECH CORE)

### Tại sao đây là "con boss cuối" của Backend?

```
╔═══════════════════════════════════════════════════════════════╗
║  MỌI hệ thống ở Big Tech đều là DISTRIBUTED SYSTEM.          ║
║                                                               ║
║  Google Spanner, Amazon DynamoDB, Uber RingPop,               ║
║  Meta TAO, Netflix Zuul — tất cả đều yêu cầu bạn            ║
║  HIỂU SÂU distributed systems theory.                         ║
║                                                               ║
║  Đây là khoảng cách LỚN NHẤT giữa Senior thường             ║
║  và Senior Big Tech.                                          ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.1 Consistency Models

```
    ┌─────────────────────────────────────────────────────┐
    │  Từ mạnh → yếu:                                    │
    │                                                      │
    │  1. Linearizability (Strongest)                     │
    │     → Mọi operation có vẻ xảy ra tại 1 thời điểm │
    │     → Real-time ordering                            │
    │     → Chi phí cực cao (coordination overhead)      │
    │     → VD: Google Spanner (TrueTime API)            │
    │                                                      │
    │  2. Sequential Consistency                          │
    │     → Mọi process thấy CÙNG thứ tự operations     │
    │     → Nhưng không cần real-time                     │
    │                                                      │
    │  3. Causal Consistency                              │
    │     → Causally-related operations = ordered        │
    │     → Concurrent operations = unordered            │
    │     → VD: CRDT-based systems                       │
    │                                                      │
    │  4. Eventual Consistency (Weakest practical)        │
    │     → "Cuối cùng" tất cả replicas sẽ giống nhau  │
    │     → Không đảm bảo KHI NÀO                        │
    │     → VD: DNS, Amazon S3                            │
    │                                                      │
    │  ⚠️ BIG TECH yêu cầu bạn BIẾT trade-off           │
    │  giữa từng model và KHI NÀO chọn model nào!       │
    └─────────────────────────────────────────────────────┘
```

### 8.2 Consensus Algorithms

```
    ┌─────────────────────────────────────────────────────┐
    │  ★★★★★ Raft (PHẢI hiểu sâu)                        │
    │  ├── Leader Election                                │
    │  ├── Log Replication                                │
    │  ├── Safety guarantees                              │
    │  ├── Membership changes                             │
    │  └── Dùng ở: etcd, CockroachDB, Consul            │
    │                                                      │
    │  ★★★★ Paxos (Biết concept)                          │
    │  ├── Proposer, Acceptor, Learner                    │
    │  ├── Multi-Paxos                                    │
    │  └── Dùng ở: Google Chubby, Megastore              │
    │                                                      │
    │  ★★★ PBFT & BFT (Biết tồn tại)                     │
    │  └── Byzantine Fault Tolerance                      │
    │                                                      │
    │  Concepts liên quan:                                │
    │  ├── FLP Impossibility Theorem                      │
    │  │   → Impossibility of consensus in async system  │
    │  │   → với ≥1 faulty process                       │
    │  ├── Split-Brain problem                            │
    │  └── Quorum (majority-based decisions)              │
    └─────────────────────────────────────────────────────┘
```

### 8.3 Distributed Transactions

```
    ┌─────────────────────────────────────────────────────┐
    │  1. Two-Phase Commit (2PC)                          │
    │     → Prepare phase → Commit phase                 │
    │     → Blocking protocol (coordinator fails = stuck)│
    │     → Dùng trong: Database clusters                │
    │                                                      │
    │  2. Three-Phase Commit (3PC)                        │
    │     → Non-blocking nhưng phức tạp hơn              │
    │                                                      │
    │  3. Saga Pattern (Microservices preferred)           │
    │     → Choreography: events giữa services           │
    │     → Orchestration: central coordinator            │
    │     → Compensating transactions (rollback)         │
    │     → VD: Order → Payment → Inventory → Shipping  │
    │           Nếu Inventory fail → refund Payment     │
    │                                                      │
    │  4. Outbox Pattern                                  │
    │     → Ghi event vào DB table (outbox) cùng TX     │
    │     → Worker poll outbox → publish to message queue│
    │     → Đảm bảo at-least-once delivery               │
    │     → CDC (Change Data Capture) alternative: Debezium│
    └─────────────────────────────────────────────────────┘
```

### 8.4 Clocks & Ordering

```
    ┌─────────────────────────────────────────────────────┐
    │  "There is no global clock in distributed systems"  │
    │                                                      │
    │  1. Physical Clocks                                 │
    │     → NTP (Network Time Protocol) — not exact      │
    │     → Clock skew, clock drift                      │
    │     → Google TrueTime — bounded uncertainty        │
    │                                                      │
    │  2. Logical Clocks                                  │
    │     → Lamport Timestamps                            │
    │        • Monotonically increasing counter           │
    │        • Establishes partial ordering               │
    │     → Vector Clocks                                 │
    │        • Detect concurrent events                   │
    │        • Conflict resolution                        │
    │     → Hybrid Logical Clocks (HLC)                   │
    │        • Combine physical + logical                 │
    │        • CockroachDB uses this                      │
    │                                                      │
    │  3. Ordering Guarantees                             │
    │     → Total order vs Partial order                  │
    │     → Happens-before relationship                   │
    │     → Causal ordering                               │
    └─────────────────────────────────────────────────────┘
```

### 8.5 Failure Modes & Resilience

```
    ┌─────────────────────────────────────────────────────┐
    │  Types of Failures:                                  │
    │  ├── Crash failure (process dies)                   │
    │  ├── Omission failure (message lost)               │
    │  ├── Timing failure (too slow)                      │
    │  ├── Byzantine failure (arbitrary behavior)         │
    │  └── Network partition (nodes can't communicate)   │
    │                                                      │
    │  CAP Theorem — Deep Understanding:                  │
    │  ├── Consistency + Availability + Partition tol.   │
    │  ├── Network partition WILL happen                  │
    │  ├── → Choose CP or AP                              │
    │  ├── CP: Banking, inventory (correctness first)    │
    │  └── AP: Social media, cache (availability first)  │
    │                                                      │
    │  PACELC Theorem (extends CAP):                      │
    │  ├── If Partition → choose A or C                   │
    │  └── Else → choose Latency or Consistency          │
    │                                                      │
    │  Resilience Patterns:                               │
    │  ├── Circuit Breaker (states: closed/open/half)    │
    │  ├── Bulkhead (resource isolation)                  │
    │  ├── Retry + Exponential Backoff + Jitter          │
    │  ├── Timeout budgets (deadline propagation)         │
    │  ├── Graceful degradation                           │
    │  └── Health checking (deep vs shallow)              │
    └─────────────────────────────────────────────────────┘
```

### 8.6 Tài nguyên Phase 8

```
    ┌─────────────────────────────────────────────────────┐
    │  📖 BẮT BUỘC:                                       │
    │  • "Designing Data-Intensive Applications" (DDIA)  │
    │    — Martin Kleppmann (Chapters 5-9 là VÀNG)       │
    │  • "Understanding Distributed Systems" — Vitillo   │
    │                                                      │
    │  📝 Papers:                                          │
    │  • Raft paper (In Search of an Understandable      │
    │    Consensus Algorithm)                              │
    │  • Google Spanner paper                             │
    │  • Amazon Dynamo paper                              │
    │  • MapReduce paper                                  │
    │                                                      │
    │  🔧 Hands-on:                                        │
    │  • Implement Raft from scratch (thử với Go)        │
    │  • MIT 6.824 Distributed Systems labs               │
    │  • Fly.io distributed systems challenges            │
    └─────────────────────────────────────────────────────┘
```

---

## Phase 9: Go Runtime & Performance Engineering

### Tại sao Big Tech cần bạn hiểu Runtime?

```
╔═══════════════════════════════════════════════════════════════╗
║  "Senior Big Tech không chỉ DÙNG ngôn ngữ.                   ║
║   Họ HIỂU ngôn ngữ hoạt động như thế nào bên trong."         ║
║                                                               ║
║  Uber, Google, CrowdStrike yêu cầu Go Senior hiểu:          ║
║  • Tại sao service bị latency spike?                         ║
║  • GC pause ảnh hưởng p99 như thế nào?                       ║
║  • Memory leak xảy ra ở đâu trong Go?                        ║
║  • Goroutine leak pattern nào phổ biến?                       ║
╚═══════════════════════════════════════════════════════════════╝
```

### 9.1 Go Garbage Collector Deep Dive

```
    ┌─────────────────────────────────────────────────────┐
    │  Go GC — Tri-color Mark & Sweep:                    │
    │                                                      │
    │  1. Phases:                                          │
    │     → Mark Setup (STW — Stop The World)             │
    │     → Marking (concurrent with mutator)             │
    │     → Mark Termination (STW)                        │
    │     → Sweeping (concurrent)                         │
    │                                                      │
    │  2. Write Barrier                                    │
    │     → Ensures correctness during concurrent marking│
    │     → Performance overhead during mark phase       │
    │                                                      │
    │  3. GOGC & GOMEMLIMIT                               │
    │     → GOGC controls GC frequency (default=100)     │
    │     → GOMEMLIMIT (Go 1.19+) — soft memory limit   │
    │     → Tuning: latency vs throughput trade-off      │
    │                                                      │
    │  4. GC Pacing                                       │
    │     → How Go decides WHEN to run GC               │
    │     → Heap growth trigger                           │
    │     → CPU utilization target (25%)                  │
    │                                                      │
    │  5. Common GC Issues:                               │
    │     → Large heap → long mark phase                 │
    │     → Pointer-heavy data → GC overhead             │
    │     → Fix: reduce allocations, use value types     │
    │     → Fix: sync.Pool for temporary objects         │
    │     → Fix: arena (Go 1.20+ experimental)           │
    └─────────────────────────────────────────────────────┘
```

### 9.2 Go Scheduler (GMP Model)

```
    ┌─────────────────────────────────────────────────────┐
    │  G (Goroutine) — M (Machine/OS Thread) — P (Proc)  │
    │                                                      │
    │  ┌───┐  ┌───┐  ┌───┐  ┌───┐                        │
    │  │ G │  │ G │  │ G │  │ G │  ← Goroutines          │
    │  └─┬─┘  └─┬─┘  └─┬─┘  └─┬─┘                        │
    │    │      │      │      │                            │
    │  ┌─▼──────▼─┐  ┌─▼──────▼─┐                        │
    │  │    P      │  │    P      │  ← Logical Processors │
    │  └─────┬─────┘  └─────┬─────┘    (GOMAXPROCS)       │
    │        │              │                              │
    │  ┌─────▼─────┐  ┌─────▼─────┐                       │
    │  │     M      │  │     M      │ ← OS Threads        │
    │  └───────────┘  └───────────┘                       │
    │                                                      │
    │  Key Concepts:                                       │
    │  ├── Work Stealing (idle P steals from busy P)     │
    │  ├── Preemption (cooperative → async since 1.14)   │
    │  ├── System calls → M detaches, new M spins up    │
    │  ├── Netpoller (non-blocking I/O integration)      │
    │  └── GOMAXPROCS — default = num CPU cores          │
    │                                                      │
    │  Debugging:                                          │
    │  ├── GODEBUG=schedtrace=1000 (scheduler trace)     │
    │  ├── runtime.NumGoroutine()                        │
    │  └── Goroutine dump (SIGQUIT / runtime.Stack)      │
    └─────────────────────────────────────────────────────┘
```

### 9.3 Memory Model & Data Races

```
    ┌─────────────────────────────────────────────────────┐
    │  Go Memory Model (QUAN TRỌNG cho Big Tech):         │
    │                                                      │
    │  1. Happens-Before Relationship                     │
    │     → goroutine creation happens-before execution  │
    │     → channel send happens-before receive          │
    │     → sync.Mutex unlock happens-before lock        │
    │                                                      │
    │  2. Data Race = UNDEFINED BEHAVIOR trong Go         │
    │     → Hai goroutines access cùng variable          │
    │     → Ít nhất 1 là write                           │
    │     → Không có synchronization                      │
    │     → go test -race (LUÔN BẬT trong CI)            │
    │                                                      │
    │  3. Synchronization Primitives (khi nào dùng gì):  │
    │     ├── Channel: communication, coordination       │
    │     ├── sync.Mutex: protect shared state           │
    │     ├── sync.RWMutex: multiple readers OK          │
    │     ├── sync.Once: one-time initialization         │
    │     ├── sync.Map: concurrent map (specific cases)  │
    │     ├── atomic: low-level, lock-free operations    │
    │     └── sync.Cond: conditional waiting             │
    │                                                      │
    │  4. Common Mistakes:                                │
    │     → Closing channel from receiver side           │
    │     → Goroutine leak (no cancellation)             │
    │     → Variable capture in loop (pre-1.22)          │
    │     → Shared slice/map without sync               │
    └─────────────────────────────────────────────────────┘
```

### 9.4 Profiling & Performance Optimization

```
    ┌─────────────────────────────────────────────────────┐
    │  Profiling Tools (Go built-in):                     │
    │  ├── CPU profiling (pprof)                         │
    │  │   → go tool pprof http://host/debug/pprof/      │
    │  │   → Flame graphs (top-down, bottom-up)          │
    │  │                                                   │
    │  ├── Memory profiling (heap, allocs)                │
    │  │   → Find allocation hotspots                    │
    │  │   → Inuse vs Alloc objects/space                │
    │  │                                                   │
    │  ├── Goroutine profiling                            │
    │  │   → Detect goroutine leaks                      │
    │  │   → Blocked goroutines analysis                 │
    │  │                                                   │
    │  ├── Block profiling                               │
    │  │   → Time spent waiting on sync primitives       │
    │  │                                                   │
    │  ├── Mutex profiling                               │
    │  │   → Contention analysis                         │
    │  │                                                   │
    │  └── Execution tracer (go tool trace)              │
    │      → Complete timeline of execution              │
    │      → GC events, goroutine scheduling             │
    │                                                      │
    │  Optimization Techniques:                           │
    │  ├── Reduce allocations (stack > heap)              │
    │  ├── sync.Pool for temporary objects               │
    │  ├── Pre-allocate slices (make([]T, 0, cap))       │
    │  ├── strings.Builder instead of + concatenation    │
    │  ├── Avoid interface{} boxing                      │
    │  ├── Struct field ordering (padding reduction)      │
    │  ├── Buffer reuse patterns                         │
    │  └── Connection pooling (DB, HTTP, gRPC)           │
    └─────────────────────────────────────────────────────┘
```

---

## Phase 10: Production & Operations (at Scale)

### 10.1 Containerization & Orchestration

```
    ┌─────────────────────────────────────────────────┐
    │  Docker (PHẢI BIẾT):                             │
    │  ├── Dockerfile best practices                  │
    │  │   → Multi-stage builds                       │
    │  │   → Layer caching                            │
    │  │   → Non-root user                            │
    │  │   → .dockerignore                            │
    │  ├── Docker Compose (local dev)                 │
    │  ├── Networking (bridge, host, overlay)          │
    │  └── Volume management                          │
    │                                                  │
    │  Kubernetes (NÊN BIẾT):                          │
    │  ├── Pod, Deployment, Service, Ingress          │
    │  ├── ConfigMap, Secret                          │
    │  ├── Health checks (liveness, readiness)        │
    │  ├── Resource limits (CPU, Memory)              │
    │  ├── HPA (Horizontal Pod Autoscaler)            │
    │  └── Helm charts                                │
    └─────────────────────────────────────────────────┘
```

### 10.2 CI/CD Pipeline

```
    ┌─────────────────────────────────────────────────┐
    │  Pipeline chuẩn:                                 │
    │                                                  │
    │  Code → Lint → Test → Build → Deploy            │
    │                                                  │
    │  ├── Linting: golangci-lint                     │
    │  ├── Testing: unit, integration, e2e            │
    │  ├── Build: Docker image, binary                │
    │  ├── Deploy strategies:                         │
    │  │   • Rolling update                           │
    │  │   • Blue-Green deployment                    │
    │  │   • Canary deployment                        │
    │  │   • Feature flags                            │
    │  └── Tools: GitHub Actions, GitLab CI           │
    │                                                  │
    │  Git workflow:                                   │
    │  ├── Trunk-based development                    │
    │  ├── Feature branches + PR review               │
    │  ├── Semantic versioning                        │
    │  └── Conventional commits                       │
    └─────────────────────────────────────────────────┘
```

### 10.3 Observability (Tam giác vàng)

```
    ┌─────────────────────────────────────────────────┐
    │  3 trụ cột của Observability:                    │
    │                                                  │
    │       ┌──────────┐                               │
    │       │  Logs    │  ← Cái gì đã XẢY RA?        │
    │       └────┬─────┘                               │
    │            │                                      │
    │  ┌────────┴────────┐                             │
    │  │                 │                              │
    │  ▼                 ▼                              │
    │ ┌──────────┐  ┌──────────┐                       │
    │ │ Metrics  │  │ Traces   │                       │
    │ │          │  │          │                        │
    │ │ Bao nhiêu│  │ TẠI SAO │                        │
    │ │ & Khi nào│  │ & Ở ĐÂU │                        │
    │ └──────────┘  └──────────┘                       │
    │                                                  │
    │  Logging:                                        │
    │  ├── Structured logging (JSON)                  │
    │  ├── Log levels (DEBUG → INFO → WARN → ERROR)   │
    │  ├── Correlation IDs (request tracing)           │
    │  └── Tools: ELK Stack, Loki + Grafana           │
    │                                                  │
    │  Metrics:                                        │
    │  ├── RED method: Rate, Errors, Duration         │
    │  ├── USE method: Utilization, Saturation, Errors│
    │  ├── Custom business metrics                    │
    │  └── Tools: Prometheus + Grafana                │
    │                                                  │
    │  Distributed Tracing:                            │
    │  ├── OpenTelemetry (standard)                   │
    │  ├── Span, Trace, Context propagation            │
    │  └── Tools: Jaeger, Zipkin, Datadog             │
    └─────────────────────────────────────────────────┘
```

---

### 10.4 Site Reliability Engineering (SRE)

```
    ┌─────────────────────────────────────────────────────┐
    │  SRE — Google's approach to operations:             │
    │                                                      │
    │  1. SLI / SLO / SLA                                 │
    │     → SLI: Service Level Indicator (metric)        │
    │       • Latency, error rate, throughput             │
    │     → SLO: Service Level Objective (target)        │
    │       • "99.9% requests < 200ms"                   │
    │     → SLA: Service Level Agreement (contract)      │
    │       • SLO + consequences (refund, penalty)       │
    │                                                      │
    │  2. Error Budgets                                   │
    │     → 99.9% uptime = 8.76 hours downtime/year     │
    │     → Error budget = 100% - SLO                    │
    │     → Budget remaining → deploy freely             │
    │     → Budget exhausted → focus on reliability      │
    │                                                      │
    │  3. Incident Management                             │
    │     → On-call rotation                             │
    │     → Incident response process                    │
    │     → Postmortem (blameless, actionable)            │
    │     → Runbooks (automated response)                │
    │                                                      │
    │  4. Toil Reduction                                  │
    │     → Automate repetitive operational work         │
    │     → Target: < 50% time on toil                   │
    │     → Infrastructure as Code (Terraform, Pulumi)   │
    └─────────────────────────────────────────────────────┘
```

### 10.5 Security Engineering

```
    ┌─────────────────────────────────────────────────────┐
    │  Big Tech Senior PHẢI hiểu Security:                │
    │                                                      │
    │  1. Authentication & Authorization                  │
    │     → OAuth 2.0 / OIDC flows (implicit, auth code)│
    │     → mTLS (service mesh)                          │
    │     → RBAC vs ABAC vs ReBAC                        │
    │     → Zero Trust Architecture                      │
    │                                                      │
    │  2. Application Security                            │
    │     → OWASP Top 10 (SQL injection, XSS, CSRF)     │
    │     → Input validation / Output encoding            │
    │     → Parameterized queries (never concatenate)    │
    │     → Rate limiting + DDoS protection              │
    │                                                      │
    │  3. Infrastructure Security                         │
    │     → Secret management (Vault, AWS Secrets Mgr)   │
    │     → Container security scanning                  │
    │     → Network policies (K8s)                       │
    │     → Supply chain security (SBOM, Sigstore)       │
    │                                                      │
    │  4. Data Security                                   │
    │     → Encryption at rest & in transit               │
    │     → Data classification                          │
    │     → PII handling & compliance (GDPR, SOC2)       │
    │     → Audit logging                                 │
    └─────────────────────────────────────────────────────┘
```

---

## Phase 11: Big Tech Interview & Senior Mindset

### Tại sao cần Phase riêng cho Interview?

```
╔═══════════════════════════════════════════════════════════════╗
║  Biết kiến thức ≠ Pass interview.                             ║
║                                                               ║
║  Big Tech interview có FORMAT riêng:                          ║
║  • Coding Interview (2-3 rounds)                             ║
║  • System Design Interview (1-2 rounds)                      ║
║  • Behavioral Interview (1-2 rounds)                         ║
║  • Cross-functional / Bar Raiser (Amazon)                    ║
║                                                               ║
║  Bạn cần LUYỆN TẬP format này cụ thể!                       ║
╚═══════════════════════════════════════════════════════════════╝
```

### 11.1 Coding Interview Strategy

```
    ┌─────────────────────────────────────────────────────┐
    │  Big Tech Coding Assessment:                        │
    │                                                      │
    │  1. Problem-Solving Framework (45 min/round):       │
    │     → Clarify (2-3 min): constraints, edge cases   │
    │     → Plan (5 min): approach, time complexity       │
    │     → Code (25 min): clean, modular code           │
    │     → Test (5-10 min): examples, edge cases        │
    │     → Optimize (if time): better complexity        │
    │                                                      │
    │  2. Topics theo tần suất (Backend Senior):          │
    │  ┌──────────────────┬──────────────────────────────┐│
    │  │ ★★★★★ Rất thường │ Hash Maps, Trees, Graphs,   ││
    │  │                  │ BFS/DFS, Sliding Window,     ││
    │  │                  │ Binary Search, Two Pointers  ││
    │  ├──────────────────┼──────────────────────────────┤│
    │  │ ★★★★ Thường      │ Dynamic Programming,        ││
    │  │                  │ Stack/Queue, Heap,           ││
    │  │                  │ Trie, Topological Sort       ││
    │  ├──────────────────┼──────────────────────────────┤│
    │  │ ★★★ Đôi khi      │ Union Find, Segment Tree,   ││
    │  │                  │ Bit Manipulation,            ││
    │  │                  │ Monotonic Stack/Queue        ││
    │  └──────────────────┴──────────────────────────────┘│
    │                                                      │
    │  3. Luyện tập:                                      │
    │     → LeetCode: 200-300 bài (quality > quantity)   │
    │     → Blind 75 → NeetCode 150 → Company tagged    │
    │     → 1 bài/ngày, tối thiểu 3-6 tháng             │
    │     → Mock interview (Pramp, Interviewing.io)      │
    └─────────────────────────────────────────────────────┘
```

### 11.2 System Design Interview

```
    ┌─────────────────────────────────────────────────────┐
    │  Quy trình System Design Interview (45-60 min):     │
    │                                                      │
    │  Step 1: REQUIREMENTS (5 min)                       │
    │  ├── Functional: "What does the system do?"        │
    │  ├── Non-functional: scale, latency, consistency   │
    │  ├── Back-of-envelope: QPS, storage, bandwidth     │
    │  └── Constraints: budget, timeline, team size      │
    │                                                      │
    │  Step 2: HIGH-LEVEL DESIGN (10-15 min)              │
    │  ├── Core components & data flow                    │
    │  ├── API design (REST/gRPC endpoints)               │
    │  ├── Database schema (SQL vs NoSQL choice)         │
    │  └── Major architectural decisions                  │
    │                                                      │
    │  Step 3: DEEP DIVE (20-25 min)                      │
    │  ├── Bottleneck identification                      │
    │  ├── Scaling strategy (sharding, caching, CDN)     │
    │  ├── Data partitioning & replication                │
    │  ├── Consistency vs Availability trade-offs        │
    │  └── Failure handling & recovery                   │
    │                                                      │
    │  Step 4: WRAP UP (5 min)                            │
    │  ├── Monitoring & alerting strategy                 │
    │  ├── Future improvements & tech debt               │
    │  └── Cost optimization opportunities               │
    │                                                      │
    │  ★ KEY: Luôn DISCUSS TRADE-OFFS, không chỉ solution│
    └─────────────────────────────────────────────────────┘

    Bài tập System Design PHẢI làm (Big Tech level):
    ┌─────────────────────────────────────────────────────┐
    │  ★★★★★ ESSENTIAL (mọi company đều hỏi):            │
    │  • URL Shortener / TinyURL                         │
    │  • Rate Limiter                                     │
    │  • Chat System (WhatsApp/Messenger)                │
    │  • News Feed / Timeline (Facebook/Twitter)         │
    │                                                      │
    │  ★★★★★ COMMON (>60% interviews):                    │
    │  • Web Crawler                                      │
    │  • Notification Service (push, email, SMS)         │
    │  • Distributed Cache (Memcached/Redis cluster)     │
    │  • Key-Value Store (DynamoDB-like)                  │
    │  • Search Autocomplete / Typeahead                  │
    │                                                      │
    │  ★★★★ ADVANCED (Senior level):                      │
    │  • Video Streaming (YouTube/Netflix)                │
    │  • Payment System (Stripe-like)                    │
    │  • Ride-sharing (Uber/Lyft)                        │
    │  • Distributed Message Queue (Kafka-like)          │
    │  • Google Maps / Location Service                  │
    │  • Online Code Editor (LeetCode-like)              │
    │  • Distributed Task Scheduler                      │
    └─────────────────────────────────────────────────────┘
```

### 11.3 Behavioral Interview (STAR Method)

```
    ┌─────────────────────────────────────────────────────┐
    │  STAR Framework:                                    │
    │  ├── S: Situation (context cụ thể)                 │
    │  ├── T: Task (nhiệm vụ/vai trò của bạn)           │
    │  ├── A: Action (bạn ĐÃ LÀM gì—chi tiết)          │
    │  └── R: Result (kết quả đo lường được)             │
    │                                                      │
    │  Câu hỏi phổ biến ở Big Tech:                      │
    │  ├── "Tell me about a time you resolved a          │
    │  │    conflict with a teammate."                    │
    │  ├── "Describe a project you led from start        │
    │  │    to finish."                                   │
    │  ├── "Tell me about a time you had to make a      │
    │  │    decision with incomplete information."        │
    │  ├── "Describe a technical mistake you made        │
    │  │    and how you recovered."                       │
    │  ├── "How do you handle disagreements with your    │
    │  │    manager?"                                     │
    │  └── "Tell me about a time you mentored someone." │
    │                                                      │
    │  Tips:                                              │
    │  ├── Chuẩn bị 8-10 stories (cover mọi category)   │
    │  ├── Mỗi story dùng cho 2-3 câu hỏi khác nhau    │
    │  ├── Quantify results ("giảm 40% latency")         │
    │  └── Show senior signals:                          │
    │       • Ownership & Accountability                 │
    │       • Cross-team collaboration                   │
    │       • Technical decision-making                  │
    │       • Mentorship & knowledge sharing             │
    └─────────────────────────────────────────────────────┘
```

### 11.4 Big Tech Leveling Guide

```
    ┌─────────────────────────────────────────────────────┐
    │  Leveling tại Big Tech (Senior = L5/E5):            │
    │                                                      │
    │  ┌─────────┬──────────┬──────────────────────────┐  │
    │  │ Level   │ Title    │ Expectations              │  │
    │  ├─────────┼──────────┼──────────────────────────┤  │
    │  │ L3/E3   │ Junior   │ Deliver tasks with       │  │
    │  │         │          │ guidance                 │  │
    │  ├─────────┼──────────┼──────────────────────────┤  │
    │  │ L4/E4   │ Mid      │ Deliver features         │  │
    │  │         │          │ independently            │  │
    │  ├─────────┼──────────┼──────────────────────────┤  │
    │  │ L5/E5   │ Senior   │ Own subsystems,          │  │
    │  │         │          │ mentor others,           │  │
    │  │         │          │ drive technical          │  │
    │  │         │          │ direction                │  │
    │  ├─────────┼──────────┼──────────────────────────┤  │
    │  │ L6/E6   │ Staff    │ Org-wide technical       │  │
    │  │         │          │ leadership               │  │
    │  └─────────┴──────────┴──────────────────────────┘  │
    │                                                      │
    │  Senior (L5) Signals:                               │
    │  ├── Makes CORRECT trade-offs independently        │
    │  ├── Anticipates problems BEFORE they happen       │
    │  ├── Designs systems that SCALE                     │
    │  ├── Writes code that others can MAINTAIN          │
    │  ├── Mentors juniors EFFECTIVELY                    │
    │  ├── Communicates CLEARLY (written & verbal)       │
    │  └── Delivers IMPACT, not just code                │
    └─────────────────────────────────────────────────────┘
```

### 11.5 Senior Kỹ năng mềm & Leadership

```
    ┌─────────────────────────────────────────────────────┐
    │  1. Technical Decision Making                       │
    │     → Trade-off analysis ("why NOT X?")            │
    │     → Architecture Decision Records (ADRs)         │
    │     → RFC/Design docs TRƯỚC khi code               │
    │     → Prototype → Validate → Implement             │
    │                                                      │
    │  2. Code Review Excellence                          │
    │     → Review thiết kế, not chỉ syntax              │
    │     → Feedback: constructive, not toxic            │
    │     → Focus: correctness, readability, performance │
    │                                                      │
    │  3. Mentoring & Growth                              │
    │     → Pair programming sessions                    │
    │     → Tech talks & knowledge sharing               │
    │     → Create opportunities for junior growth       │
    │                                                      │
    │  4. Communication                                   │
    │     → Giải thích kỹ thuật cho non-tech             │
    │     → Status updates clear & concise               │
    │     → Async communication (clear writing)          │
    │                                                      │
    │  5. Business Awareness                              │
    │     → Hiểu product, user, revenue model            │
    │     → ROI của technical decisions                   │
    │     → "Kỹ thuật phục vụ business, not ngược lại" │
    └─────────────────────────────────────────────────────┘
```

### 11.6 Continuous Learning Path (Big Tech Level)

```
    ┌─────────────────────────────────────────────────────┐
    │  📖 BẮT BUỘC đọc:                                   │
    │  • "Designing Data-Intensive Applications"         │
    │    — Martin Kleppmann (★★★★★ BẮT BUỘC)            │
    │  • "System Design Interview" Vol 1 & 2 — Alex Xu  │
    │  • "Clean Architecture" — Robert C. Martin         │
    │  • "Database Internals" — Alex Petrov              │
    │  • "Site Reliability Engineering" — Google         │
    │  • "Staff Engineer" — Will Larson                  │
    │                                                      │
    │  📖 Go Specific:                                     │
    │  • "The Go Programming Language"                   │
    │  • "Concurrency in Go" — Katherine Cox-Buday      │
    │  • "100 Go Mistakes" — Teiva Harsanyi              │
    │  • "Let's Go" & "Let's Go Further" — Alex Edwards│
    │  • "Efficient Go" — Bartłomiej Płotka              │
    │                                                      │
    │  📝 Papers (Big Tech engineer đọc papers):          │
    │  • Google: MapReduce, GFS, Bigtable, Spanner       │
    │  • Amazon: Dynamo                                   │
    │  • Facebook: TAO, Memcached at Scale               │
    │  • LinkedIn: Kafka                                  │
    │                                                      │
    │  🎙️ Engineering Blogs:                               │
    │  • Uber Engineering, Netflix, Stripe, Cloudflare   │
    │  • Google Research Blog, Meta Engineering          │
    │  • The Go Blog (go.dev/blog)                       │
    │  • Martin Fowler's Blog                            │
    │                                                      │
    │  🔧 Hands-on Challenges:                             │
    │  • CodeCrafters (Build your own Redis, Docker, Git)│
    │  • Fly.io Distributed Systems challenges           │
    │  • MIT 6.824 Labs (Distributed Systems)            │
    │  • Contribute to Go open source projects           │
    └─────────────────────────────────────────────────────┘
```

---

## Timeline Tham Khảo (Big Tech Level)

```
╔═══════════════════════════════════════════════════════════════════════╗
║  ⚠️ TIMELINE NÀY CHỈ LÀ THAM KHẢO                                    ║
║  Mỗi người học với tốc độ khác nhau.                                   ║
║  Quan trọng là HIỂU SÂU, không phải học nhanh.                        ║
║  Để đạt Big Tech Senior cần TỔNG 2-4 NĂM tập trung.                  ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                        ║
║  ── FOUNDATION (6 tháng) ─────────────────────────────────────────    ║
║  Tháng 1-2    │ Phase 0: CS Fundamentals                              ║
║  Tháng 2-4    │ Phase 1: Golang Mastery                               ║
║  Tháng 4-6    │ Phase 2: Networking & Protocols                       ║
║                                                                        ║
║  ── INTERMEDIATE (8 tháng) ───────────────────────────────────────    ║
║  Tháng 6-9    │ Phase 3: Database & Storage                           ║
║  Tháng 9-11   │ Phase 4: API Design & Web Server                     ║
║  Tháng 11-13  │ Phase 5: Software Design Patterns                    ║
║  Tháng 13-14  │ Phase 6: Advanced Testing & Quality                  ║
║                                                                        ║
║  ── ADVANCED (10 tháng) ──────────────────────────────────────────    ║
║  Tháng 14-18  │ Phase 7: System Design (học liên tục)                ║
║  Tháng 18-22  │ Phase 8: Distributed Systems (core)                  ║
║  Tháng 20-24  │ Phase 9: Go Runtime & Performance                    ║
║                                                                        ║
║  ── BIG TECH READY (song song) ──────────────────────────────────    ║
║  Tháng 20-26  │ Phase 10: Production & Operations                    ║
║  Tháng 22-28+ │ Phase 11: Interview Prep (3-6 tháng tập trung)      ║
║                                                                        ║
║  ★ BUILD PROJECTS ở mỗi phase!                                        ║
║  ★ Không chỉ đọc — phải CODE và DEPLOY!                              ║
║  ★ LeetCode 1 bài/ngày từ tháng 12 trở đi!                          ║
║                                                                        ║
╚═══════════════════════════════════════════════════════════════════════╝

    Project milestones (Big Tech level):

    ┌─────────────────────────────────────────────────────┐
    │  🏗️ Project 1 (sau Phase 1-2):                     │
    │     CLI tool — HTTP client, file parser             │
    │                                                      │
    │  🏗️ Project 2 (sau Phase 3-4):                     │
    │     REST API + PostgreSQL + Redis                   │
    │     (User management, Auth, CRUD, Tests)            │
    │                                                      │
    │  🏗️ Project 3 (sau Phase 5-6):                     │
    │     Refactor Project 2 với Clean Architecture,      │
    │     DDD patterns, 80%+ test coverage               │
    │                                                      │
    │  🏗️ Project 4 (sau Phase 7):                       │
    │     URL Shortener / Chat system                     │
    │     (Distributed design, caching, queuing)          │
    │                                                      │
    │  🏗️ Project 5 (sau Phase 8-9):                     │
    │     Build distributed KV store hoặc Raft impl      │
    │     + Performance profiling & optimization          │
    │                                                      │
    │  🏗️ Project 6 (sau Phase 10):                      │
    │     Deploy Project 4 với full CI/CD pipeline        │
    │     Docker + K8s + Monitoring + SRE practices       │
    │                                                      │
    │  🏗️ Project 7 (Big Tech Ready):                    │
    │     Open source contribution (Go ecosystem)         │
    │     hoặc Build your own Redis/Docker (CodeCrafters) │
    └─────────────────────────────────────────────────────┘
```

---

## Kết Luận

```
╔═══════════════════════════════════════════════════════════════════════╗
║                                                                        ║
║  "Không có đường tắt đến Senior Big Tech.                             ║
║   Chỉ có CON ĐƯỜNG ĐÚNG — học từ gốc rễ,                             ║
║   xây từng tầng kiến thức,                                            ║
║   và KHÔNG BAO GIỜ ngừng học."                                        ║
║                                                                        ║
║  Big Tech Senior Principles:                                           ║
║  ┌───────────────────────────────────────────────────────────┐         ║
║  │  1. HIỂU SÂU > Biết nhiều                               │         ║
║  │  2. BUILD THINGS > Đọc sách                              │         ║
║  │  3. FUNDAMENTALS > Frameworks                            │         ║
║  │  4. TRADE-OFFS > "Best practices"                        │         ║
║  │  5. CONSISTENCY > Intensity                              │         ║
║  │  6. IMPACT > Lines of code                               │         ║
║  │  7. TEACH OTHERS > Work alone                            │         ║
║  └───────────────────────────────────────────────────────────┘         ║
║                                                                        ║
║  "The best engineers I've worked with aren't the ones who              ║
║   know the most. They're the ones who can make the best                ║
║   decisions with incomplete information."                              ║
║                                                     — Staff @Google   ║
║                                                                        ║
╚═══════════════════════════════════════════════════════════════════════╝
```
