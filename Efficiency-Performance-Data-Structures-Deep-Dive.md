# Efficiency with Algorithms, Performance with Data Structures — Hiệu Suất Qua Thuật Toán & Cấu Trúc Dữ Liệu

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài talk "Efficiency with Algorithms, Performance with Data Structures" — Chandler Carruth (Google, CppCon)
> **Góc nhìn:** Efficiency vs Performance, Cache Locality, Mechanical Sympathy — áp dụng cho Go
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                               | Mô tả                                     |
| --- | ------------------------------------ | ----------------------------------------- |
| §1  | Tại sao Performance quan trọng?      | Battery, Data Center, Compute per Watt    |
| §2  | Efficiency vs Performance            | Hai mặt hoàn toàn khác nhau               |
| §3  | Efficiency qua Algorithms            | Làm ÍT VIỆC hơn, substring searching      |
| §4  | Thói quen tốt — Luôn làm ít việc hơn | Reserve, cache lookup, API design         |
| §5  | CPU Memory Hierarchy                 | L1, L2, L3, Main Memory — latency table   |
| §6  | Performance qua Data Structures      | Linked List vs Contiguous, Cache Locality |
| §7  | Map & Hash Table — Thực tế đau lòng  | std::map, unordered_map, open addressing  |
| §8  | Áp dụng cho Go                       | Slice, Map, Mechanical Sympathy           |
| §9  | Tổng kết & Câu hỏi phỏng vấn Senior  | Ôn tập & thực hành                        |

---

## §1. Tại Sao Performance Quan Trọng?

```
╔═══════════════════════════════════════════════════════════════╗
║   TẠI SAO CẦN QUAN TÂM PERFORMANCE?                          ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Niklaus Wirth (tác giả Pascal):                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Software is getting SLOWER more rapidly     │             ║
║  │  than hardware becomes FASTER."              │             ║
║  │                                              │             ║
║  │ → Phần mềm CHẬM đi nhanh hơn              │             ║
║  │   phần cứng NHANH lên!                      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  LÝ DO #1: BATTERY — PIN!                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Kỹ thuật tiết kiệm pin #1:                │             ║
║  │  "RACE TO SLEEP" — chạy xong, TẮT CPU!     │             ║
║  │                                              │             ║
║  │  ┌─────────┐  Code chậm:                    │             ║
║  │  │ ████████│  CPU BẬT lâu → tốn PIN!      │             ║
║  │  │ ████████│                                 │             ║
║  │  │ ████████│  CPU ON ────────────────→       │             ║
║  │  └─────────┘                                 │             ║
║  │                                              │             ║
║  │  ┌─────────┐  Code nhanh:                    │             ║
║  │  │ ████    │  CPU BẬT ngắn → TIẾT KIỆM!   │             ║
║  │  │         │                                 │             ║
║  │  │  sleep… │  CPU ON ──→ OFF (sleep!)       │             ║
║  │  └─────────┘                                 │             ║
║  │                                              │             ║
║  │  → Code NHANH hơn = PIN lâu hơn!           │             ║
║  │  → CPU tiết kiệm bằng cách TẮT mình đi!  │             ║
║  │  → Muốn tắt sớm = phải chạy NHANH!        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  LÝ DO #2: DATA CENTER — TIỀN ĐIỆN!                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Data Center Google:                          │             ║
║  │  → Chức năng: biến ĐIỆN thành NHIỆT!       │             ║
║  │  → Điện = TỐN TIỀN + HỮU HẠN!             │             ║
║  │  → Compute per Watt = thước đo!             │             ║
║  │                                              │             ║
║  │  Code nhanh hơn = CÙNG điện, NHIỀU hơn work!│             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  KẾT LUẬN:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Ngôn ngữ (C++, Go, Java) KHÔNG CHO bạn     │             ║
║  │ performance! Nó cho bạn QUYỀN KIỂM SOÁT!   │             ║
║  │                                              │             ║
║  │ → Go cho bạn kiểm soát memory layout!       │             ║
║  │ → Go cho bạn kiểm soát goroutine!           │             ║
║  │ → Bạn phải HIỂU để kiểm soát được!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Efficiency vs Performance

```
╔═══════════════════════════════════════════════════════════════╗
║   HAI MẶT CỦA ĐỒNG XU — EFFICIENCY vs PERFORMANCE           ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  EFFICIENCY (Hiệu quả):                      │             ║
║  │  ┌────────────────────────────────┐          │             ║
║  │  │ "How much WORK is required     │          │             ║
║  │  │  by a task?"                   │          │             ║
║  │  │                                │          │             ║
║  │  │ → Bao nhiêu CÔNG VIỆC cần    │          │             ║
║  │  │   để hoàn thành task?          │          │             ║
║  │  │ → Cải thiện = LÀM ÍT HƠN!   │          │             ║
║  │  │ → Đạt được qua ALGORITHMS!    │          │             ║
║  │  │ → O(n²) → O(n log n) → O(n) │          │             ║
║  │  └────────────────────────────────┘          │             ║
║  │                                              │             ║
║  │  ════════════════════════════════             │             ║
║  │                                              │             ║
║  │  PERFORMANCE (Hiệu suất):                    │             ║
║  │  ┌────────────────────────────────┐          │             ║
║  │  │ "How QUICKLY a program does    │          │             ║
║  │  │  its work?"                    │          │             ║
║  │  │                                │          │             ║
║  │  │ → CÙNG lượng work, làm       │          │             ║
║  │  │   NHANH HƠN!                  │          │             ║
║  │  │ → Đạt được qua DATA STRUCTURES│          │             ║
║  │  │ → Cache locality, contiguous  │          │             ║
║  │  │   memory, CPU-friendly layout  │          │             ║
║  │  └────────────────────────────────┘          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BẢNG SO SÁNH:                                                 ║
║  ┌───────────────┬────────────────┬────────────────┐          ║
║  │               │ EFFICIENCY     │ PERFORMANCE    │          ║
║  ├───────────────┼────────────────┼────────────────┤          ║
║  │ Câu hỏi     │ Bao nhiêu work?│ Nhanh bao nhiêu?│         ║
║  │ Cải thiện   │ Làm ÍT hơn    │ Làm NHANH hơn  │          ║
║  │ Công cụ     │ Algorithms     │ Data Structures │          ║
║  │ Đo lường    │ Big-O          │ ns/op, cache hit│          ║
║  │ Ví dụ       │ O(n²)→O(n)    │ List → Array   │          ║
║  │ Có giới hạn?│ CÓ (optimal)  │ KHÔNG (lý thuyết)│         ║
║  └───────────────┴────────────────┴────────────────┘          ║
║                                                               ║
║  Carruth:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Efficiency comes through ALGORITHMS.        │             ║
║  │  Performance comes through DATA STRUCTURES." │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Efficiency Qua Algorithms — Làm Ít Việc Hơn

```
╔═══════════════════════════════════════════════════════════════╗
║   ALGORITHMS — LÀM ÍT VIỆC HƠN!                             ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  VÍ DỤ: SUBSTRING SEARCHING                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Bài toán: Tìm "abc" trong "xyzabcdef..."  │             ║
║  │                                              │             ║
║  │  NAIVE — O(n × m):                           │             ║
║  │  text:    [x][y][z][a][b][c][d][e][f]        │             ║
║  │  pattern: [a][b][c]                           │             ║
║  │  → So sánh MỖI vị trí × MỖI ký tự!        │             ║
║  │  → 2 vòng lặp = O(n × m)!                   │             ║
║  │  → LÃNG PHÍ — lặp pattern nhiều lần!       │             ║
║  │                                              │             ║
║  │  ════════════════════════════════             │             ║
║  │                                              │             ║
║  │  KMP — O(n + m):                              │             ║
║  │  → Tiền xử lý pattern 1 lần!               │             ║
║  │  → Khi mismatch: NHẢY AHEAD!                │             ║
║  │  → Không quay lại vị trí cũ!               │             ║
║  │  → LÀM ÍT VIỆC HƠN!                        │             ║
║  │                                              │             ║
║  │  ════════════════════════════════             │             ║
║  │                                              │             ║
║  │  BOYER-MOORE — O(n/m) best case:             │             ║
║  │  → Bắt đầu từ CUỐI pattern!                │             ║
║  │  → Mismatch → nhảy NGUYÊN ĐỘ DÀI pattern! │             ║
║  │  → SKIP cả đoạn text!                       │             ║
║  │  → CÒN ÍT VIỆC HƠN NỮA!                   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ÁP DỤNG GO:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ // ❌ Tìm phần tử trong unsorted slice      │             ║
║  │ // O(n) MỖI LẦN tìm!                        │             ║
║  │ for _, v := range items {                     │             ║
║  │     if v == target { ... }                    │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ // ✅ Dùng map = O(1) lookup!                │             ║
║  │ m := make(map[string]Item)                    │             ║
║  │ val, ok := m[key] // O(1)!                    │             ║
║  │                                              │             ║
║  │ // ✅ Sort + binary search = O(log n)!       │             ║
║  │ sort.Strings(items)                           │             ║
║  │ i := sort.SearchStrings(items, target)        │             ║
║  │                                              │             ║
║  │ → Chọn algorithm ĐÚNG = ít việc hơn!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Thói Quen Tốt — Luôn Làm Ít Việc Hơn

```
╔═══════════════════════════════════════════════════════════════╗
║   THÓI QUEN HIỆU QUẢ — ALWAYS DO LESS WORK!                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  VÍ DỤ 1: PRE-ALLOCATE SLICE                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ // ❌ Append liên tục — grow nhiều lần!     │             ║
║  │ var result []Item                             │             ║
║  │ for _, v := range input {                     │             ║
║  │     result = append(result, transform(v))     │             ║
║  │ }                                             │             ║
║  │ // → Go phải grow slice: 2→4→8→16→...      │             ║
║  │ // → Mỗi lần grow = allocate + COPY ALL!    │             ║
║  │                                              │             ║
║  │ // ✅ Pre-allocate — biết trước size!        │             ║
║  │ result := make([]Item, 0, len(input))         │             ║
║  │ for _, v := range input {                     │             ║
║  │     result = append(result, transform(v))     │             ║
║  │ }                                             │             ║
║  │ // → 0 grow! 0 extra copy!                   │             ║
║  │ // → Code RÕ RÀNG HƠN: explicit intent!    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VÍ DỤ 2: TRÁNH LOOKUP LẶP LẠI                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ // ❌ Lookup MAP 4 lần cho CÙNG key!         │             ║
║  │ func process(cache map[string]*Data, key string) {│        ║
║  │     if cache[key] == nil {           // lookup 1│          ║
║  │         cache[key] = &Data{}         // lookup 2│          ║
║  │     }                                         │             ║
║  │     cache[key].Process()             // lookup 3│          ║
║  │     fmt.Println(cache[key].Result)   // lookup 4│          ║
║  │ }                                             │             ║
║  │ // → Hash key 4 lần = LÃNG PHÍ!             │             ║
║  │                                              │             ║
║  │ // ✅ Lookup 1 lần, giữ reference!           │             ║
║  │ func process(cache map[string]*Data, key string) {│        ║
║  │     d, ok := cache[key]              // 1 lần! │          ║
║  │     if !ok {                                  │             ║
║  │         d = &Data{}                           │             ║
║  │         cache[key] = d                        │             ║
║  │     }                                         │             ║
║  │     d.Process()                               │             ║
║  │     fmt.Println(d.Result)                     │             ║
║  │ }                                             │             ║
║  │ // → 1 lookup! Code RÕ RÀNG hơn!            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Carruth:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Always do less work. Not sometimes.         │             ║
║  │  Not after profiling. ALWAYS."               │             ║
║  │                                              │             ║
║  │ → Không phải micro-optimization!             │             ║
║  │ → Là viết code ELEGANT + HIỆU QUẢ!        │             ║
║  │ → Không phức tạp hơn, còn RÕ RÀNG hơn!    │             ║
║  │ → Profiler SẼ KHÔNG tìm ra những lỗi này! │             ║
║  │ → Phải làm BẢN NĂNG, thói quen!            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. CPU Memory Hierarchy — Bảng Latency Huyền Thoại

```
╔═══════════════════════════════════════════════════════════════╗
║   JEFF DEAN'S LATENCY NUMBERS — BẢNG QUAN TRỌNG NHẤT!        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌────────────────────────────────────┬────────────┐          ║
║  │ Operation                          │ Time       │          ║
║  ├────────────────────────────────────┼────────────┤          ║
║  │ L1 cache reference                 │ ~1 ns      │          ║
║  │ L2 cache reference                 │ ~4 ns      │          ║
║  │ L3 cache reference                 │ ~10 ns     │          ║
║  │ Main memory reference              │ ~100 ns    │          ║
║  │ SSD random read                    │ ~16,000 ns │          ║
║  │ Disk seek                          │ ~2 ms      │          ║
║  │ Network round trip (same DC)       │ ~500,000 ns│          ║
║  └────────────────────────────────────┴────────────┘          ║
║                                                               ║
║  SƠ ĐỒ MEMORY HIERARCHY:                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │        ┌────────┐                            │             ║
║  │        │   L1   │  ~1ns     ~64KB            │             ║
║  │        │ CACHE  │  SIÊU NHANH! SIÊU NHỎ!   │             ║
║  │        └───┬────┘                            │             ║
║  │        ┌───┴────┐                            │             ║
║  │        │   L2   │  ~4ns     ~256KB           │             ║
║  │        │ CACHE  │  Nhanh! Nhỏ!             │             ║
║  │        └───┬────┘                            │             ║
║  │        ┌───┴────┐                            │             ║
║  │        │   L3   │  ~10ns    ~8MB             │             ║
║  │        │ CACHE  │  Khá nhanh!               │             ║
║  │        └───┬────┘                            │             ║
║  │    ┌───────┴───────┐                         │             ║
║  │    │  MAIN MEMORY  │  ~100ns   ~16GB         │             ║
║  │    │     (RAM)     │  100x chậm hơn L1!    │             ║
║  │    └───────────────┘                         │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VẤN ĐỀ LỚN:                                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  CPU hiện đại: ~36 BILLION instructions/sec! │             ║
║  │                                              │             ║
║  │  NHƯNG: ~50% thời gian = CHỜ DATA!         │             ║
║  │  (Có khi 75%!)                                │             ║
║  │                                              │             ║
║  │  → 1 cycle = CPU thực thi ~36 operations!   │             ║
║  │  → 1 main memory access = 100 CYCLES!       │             ║
║  │  → CPU NGỒI CHỜ 100 cycles để lấy 1 byte! │             ║
║  │                                              │             ║
║  │  → Giải pháp DUY NHẤT: giữ data TRONG CACHE!│             ║
║  │  → Data phải CONTIGUOUS (liên tục)!         │             ║
║  │  → Data phải NHỎ GỌN!                      │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CACHE LINE:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Cache hoạt động theo LINE (= 64 bytes):     │             ║
║  │                                              │             ║
║  │ Truy cập 1 byte → CPU load NGUYÊN 64 bytes! │             ║
║  │ ┌─────────────────────────────────────┐      │             ║
║  │ │ byte 0 │ byte 1 │ ... │ byte 63    │      │             ║
║  │ └─────────────────────────────────────┘      │             ║
║  │ ← ─ ─ ─ ─ 64 bytes cache line ─ ─ ─ ─ →    │             ║
║  │                                              │             ║
║  │ → Nếu data LIÊN TỤC: load 64 bytes =       │             ║
║  │   nhiều phần tử FREE!                       │             ║
║  │ → Nếu data RẢI RÁC: mỗi phần tử =        │             ║
║  │   1 cache miss = 100ns penalty!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Performance Qua Data Structures — Cache Locality

```
╔═══════════════════════════════════════════════════════════════╗
║   DATA STRUCTURES & CACHE LOCALITY                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Carruth:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Discontinuous data structures are the ROOT  │             ║
║  │  OF ALL performance evil!"                   │             ║
║  │                                              │             ║
║  │ → Linked List = KẺ THÙ SỐ 1 của            │             ║
║  │   performance!                                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  LINKED LIST vs ARRAY:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ❌ LINKED LIST — DỮ LIỆU RẢI RÁC:         │             ║
║  │                                              │             ║
║  │  ┌───┐    ┌───┐    ┌───┐    ┌───┐           │             ║
║  │  │ A │───→│ B │───→│ C │───→│ D │           │             ║
║  │  └───┘    └───┘    └───┘    └───┘           │             ║
║  │   0x100   0x8F0    0x350    0xA20            │             ║
║  │   ↑        ↑        ↑        ↑               │             ║
║  │  cache    cache    cache    cache             │             ║
║  │  miss!    miss!    miss!    miss!             │             ║
║  │                                              │             ║
║  │  → Mỗi node = ADDRESS NGẪU NHIÊN!          │             ║
║  │  → Mỗi lần traverse = CACHE MISS!           │             ║
║  │  → ~100ns × N nodes = CHẬM KINH KHỦNG!     │             ║
║  │                                              │             ║
║  │  ════════════════════════════════             │             ║
║  │                                              │             ║
║  │  ✅ ARRAY / SLICE — DỮ LIỆU LIÊN TỤC:      │             ║
║  │                                              │             ║
║  │  ┌───┬───┬───┬───┬───┬───┬───┬───┐         │             ║
║  │  │ A │ B │ C │ D │ E │ F │ G │ H │         │             ║
║  │  └───┴───┴───┴───┴───┴───┴───┴───┘         │             ║
║  │   0x100  0x108  0x110  0x118 ...             │             ║
║  │   ←── cache line 1 ──→←── cache line 2 ──→  │             ║
║  │   cache hit! hit! hit!  hit! hit! hit!       │             ║
║  │                                              │             ║
║  │  → Tất cả LIỀN NHAU trong memory!          │             ║
║  │  → CPU load 1 cache line = NHIỀU phần tử!  │             ║
║  │  → Cache hit LIÊN TỤC!                      │             ║
║  │  → CPU prefetch dự đoán = CÒN NHANH HƠN!  │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  HIỆU SUẤT THỰC TẾ:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Linked List traverse: ~100ns × N             │             ║
║  │ Array traverse:       ~1ns × N               │             ║
║  │                                              │             ║
║  │ → Array NHANH HƠN ~100 LẦN cho traverse!   │             ║
║  │ → Vì CACHE = tất cả!                        │             ║
║  │                                              │             ║
║  │ Khi nào dùng Linked List?                    │             ║
║  │ → KHI HIẾM KHI traverse!                    │             ║
║  │ → KHI chỉ insert/delete ở random position! │             ║
║  │ → Trong thực tế: HIẾM HƠN bạn tưởng!     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Map & Hash Table — Thực Tế Đau Lòng

```
╔═══════════════════════════════════════════════════════════════╗
║   MAP & HASH TABLE — THIẾT KẾ QUAN TRỌNG                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  TREE-BASED MAP (std::map ≈ Red-Black Tree):                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ❌ = LINKED LIST BỊ BIẾN TƯỚNG!            │             ║
║  │                                              │             ║
║  │         ┌───┐                                │             ║
║  │         │30 │  ← root                       │             ║
║  │        /     \                               │             ║
║  │     ┌───┐   ┌───┐                           │             ║
║  │     │15 │   │45 │  ← child nodes            │             ║
║  │    /  \     /  \                             │             ║
║  │  ...  ...  ...  ...                          │             ║
║  │                                              │             ║
║  │  → Mỗi node = separate allocation!          │             ║
║  │  → Traverse = chase pointers = CACHE MISS!  │             ║
║  │  → Insert/Delete cũng traverse!             │             ║
║  │  → Rebalance = MORE traversals!              │             ║
║  │  → "Tệ hơn linked list!"                   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BUCKET-BASED HASH MAP (std::unordered_map):                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ❌ = HASH TABLE + LINKED LIST bên trong!    │             ║
║  │                                              │             ║
║  │  Hash Table:                                  │             ║
║  │  ┌────┬────┬────┬────┬────┐                 │             ║
║  │  │ *  │ *  │null│ *  │null│                 │             ║
║  │  └─┬──┴─┬──┴────┴─┬──┴────┘                 │             ║
║  │    │    │         │                          │             ║
║  │    ▼    ▼         ▼                          │             ║
║  │  ┌──┐ ┌──┐     ┌──┐                         │             ║
║  │  │KV│→│KV│     │KV│  ← LINKED LIST!         │             ║
║  │  └──┘ └──┘     └──┘                         │             ║
║  │                                              │             ║
║  │  → Pointer chase = CACHE MISS!              │             ║
║  │  → Mỗi lookup = hash + chase pointer!       │             ║
║  │  → Linked list trong BUCKET!                │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ✅ OPEN ADDRESSING HASH TABLE (Swiss Table):                │             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Flat Table — DỮ LIỆU LIÊN TỤC!            │             ║
║  │  ┌────┬────┬────┬────┬────┬────┬────┐       │             ║
║  │  │ KV │ KV │    │ KV │    │ KV │ KV │       │             ║
║  │  └────┴────┴────┴────┴────┴────┴────┘       │             ║
║  │  ← ── contiguous memory! ── →                │             ║
║  │                                              │             ║
║  │  → Hash key → index vào table!              │             ║
║  │  → Collision? LOCAL PROBING (adjacent slot)! │             ║
║  │  → Adjacent = CÙNG cache line = NHANH!      │             ║
║  │  → Không pointer! Không linked list!         │             ║
║  │  → CACHE FRIENDLY!                           │             ║
║  │                                              │             ║
║  │  → 10x nhanh hơn bucket-based!              │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §8. Áp Dụng Cho Go — Mechanical Sympathy

```
╔═══════════════════════════════════════════════════════════════╗
║   ÁP DỤNG CHO GO — MECHANICAL SYMPATHY                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  GO SLICE = ARRAY LIÊN TỤC = ✅ CACHE FRIENDLY!             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Go slice internal:                           │             ║
║  │  ┌─────────────────────────────────┐         │             ║
║  │  │ ptr ──→ [0][1][2][3][4][5]...   │         │             ║
║  │  │ len: 6                          │         │             ║
║  │  │ cap: 8                          │         │             ║
║  │  └─────────────────────────────────┘         │             ║
║  │                                              │             ║
║  │  → Data = contiguous array!                  │             ║
║  │  → for range = sequential access!            │             ║
║  │  → CPU prefetch = SIÊU NHANH!              │             ║
║  │  → GIỮ DATA TRONG L1/L2!                    │             ║
║  │                                              │             ║
║  │  BEST PRACTICES:                              │             ║
║  │  → Pre-allocate: make([]T, 0, n)             │             ║
║  │  → Dùng slice thay linked list!              │             ║
║  │  → Struct of Arrays > Array of Structs       │             ║
║  │    (khi chỉ cần 1-2 fields từ struct)       │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GO MAP = HASH TABLE!                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Go runtime map (từ Go 1.x):                │             ║
║  │  → Dùng BUCKET-based hash table!            │             ║
║  │  → Mỗi bucket chứa 8 key-value pairs!      │             ║
║  │  → Overflow bucket = linked (pointer chase)! │             ║
║  │                                              │             ║
║  │  Go 1.24+ Swiss Table:                        │             ║
║  │  → Go đã chuyển sang Swiss Table!            │             ║
║  │  → Open addressing, cache friendly!           │             ║
║  │  → Nhanh hơn đáng kể!                      │             ║
║  │                                              │             ║
║  │  BEST PRACTICES:                              │             ║
║  │  → Pre-allocate: make(map[K]V, n)             │             ║
║  │  → Giữ key NHỎ (string ngắn, int)!         │             ║
║  │  → Giữ value NHỎ (pointer nếu value lớn)!  │             ║
║  │  → Tránh lookup LẶP LẠI cùng key!          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  STRUCT LAYOUT — DATA-ORIENTED DESIGN:                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ // ❌ Struct quá lớn = ít phần tử trong cache│             ║
║  │ type Monster struct {                         │             ║
║  │     Name    string       // 16 bytes          │             ║
║  │     Texture [1024]byte   // 1024 bytes!       │             ║
║  │     X, Y    float64      // 16 bytes          │             ║
║  │     Health  int          // 8 bytes            │             ║
║  │ }                                             │             ║
║  │ // 1 Monster = 1064 bytes!                    │             ║
║  │ // 1 cache line = 0.06 Monster! LÃNG PHÍ!   │             ║
║  │                                              │             ║
║  │ // ✅ Tách data HOT ra riêng                 │             ║
║  │ type MonsterPos struct {                      │             ║
║  │     X, Y float64   // 16 bytes                │             ║
║  │ }                                             │             ║
║  │ positions := make([]MonsterPos, n)            │             ║
║  │ // 1 cache line = 4 positions! YAY!           │             ║
║  │ // Iterate positions = CACHE FRIENDLY!        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  MECHANICAL SYMPATHY — William Kennedy (Go):                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ "If you don't understand the DATA,           │             ║
║  │  you don't understand the PROBLEM.           │             ║
║  │  If you don't understand the COST of         │             ║
║  │  solving the problem, you can't reason       │             ║
║  │  about the problem."                         │             ║
║  │                                              │             ║
║  │ → Hiểu DATA = hiểu PROBLEM!                │             ║
║  │ → Hiểu COST = reasoning được!              │             ║
║  │ → Hiểu HARDWARE = viết code NHANH!         │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §9. Tổng Kết & Câu Hỏi Phỏng Vấn Senior Golang

### 9.1 Câu hỏi phỏng vấn

```
╔═══════════════════════════════════════════════════════════════╗
║   CÂU HỎI PHỎNG VẤN SENIOR GOLANG                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Q1: "Efficiency và Performance khác gì nhau?"               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Efficiency = làm ÍT VIỆC hơn!               │             ║
║  │ → Algorithms! O(n²) → O(n log n)            │             ║
║  │ → Ví dụ: sort.Search thay vì linear scan!   │             ║
║  │                                              │             ║
║  │ Performance = làm CÙNG việc NHANH HƠN!      │             ║
║  │ → Data structures! Cache locality!            │             ║
║  │ → Ví dụ: slice thay vì linked list!          │             ║
║  │                                              │             ║
║  │ Cả hai QUAN TRỌNG NHƯ NHAU và LIÊN QUAN!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Cache locality trong Go hoạt động thế nào?"           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ CPU cache hierarchy: L1(1ns) → L2(4ns)      │             ║
║  │ → L3(10ns) → Main Memory(100ns)              │             ║
║  │                                              │             ║
║  │ Go slice = contiguous array = CACHE FRIENDLY!│             ║
║  │ → for range slice = sequential = prefetch!   │             ║
║  │ → Struct nhỏ = nhiều phần tử trong cache!   │             ║
║  │                                              │             ║
║  │ Go map = hash table (Swiss Table từ 1.24)    │             ║
║  │ → Pre-allocate make(map[K]V, n)               │             ║
║  │ → Giữ key/value nhỏ gọn!                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Tại sao nên pre-allocate slice?"                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Không pre-allocate:                           │             ║
║  │ → append grow: 2→4→8→16→32 (amortized)     │             ║
║  │ → Mỗi grow = allocate + COPY ALL elements!  │             ║
║  │ → Có thể gây GC pressure!                   │             ║
║  │                                              │             ║
║  │ Pre-allocate: make([]T, 0, n)                 │             ║
║  │ → 0 grows! 0 extra copies!                   │             ║
║  │ → Code RÕ RÀNG hơn: intent explicit!        │             ║
║  │ → Profiler KHÔNG tìm ra vấn đề này!        │             ║
║  │ → Phải làm BẢN NĂNG, always!                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Mechanical Sympathy là gì? Liên quan Go?"             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = HIỂU hardware DƯỚI phần mềm!              │             ║
║  │ → Hiểu CPU cache → chọn data structure đúng!│             ║
║  │ → Hiểu memory layout → thiết kế struct đúng!│             ║
║  │ → Go cho KIỂM SOÁT memory layout!           │             ║
║  │ → Value semantics = data trên stack/inline!  │             ║
║  │ → Pointer semantics = data trên heap!        │             ║
║  │ → Slice of values = contiguous = NHANH!      │             ║
║  │ → Slice of pointers = scattered = CHẬM!     │             ║
║  │                                              │             ║
║  │ Kennedy: "Sympathize with the machine!"       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q5: "Worst is Better — cho ví dụ?"                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Bubble sort = O(n²) = CHẬM theo lý thuyết!  │             ║
║  │ → NHƯNG với <64 phần tử nhỏ = NHANH NHẤT! │             ║
║  │ → Vì data fit L1 cache = 0 cache miss!       │             ║
║  │ → Simple = ít overhead!                      │             ║
║  │                                              │             ║
║  │ Cuckoo hashing = O(1) guaranteed!             │             ║
║  │ → NHƯNG probing = random = CACHE MISS!       │             ║
║  │ → Chậm hơn linear probing trên CPU thật!    │             ║
║  │                                              │             ║
║  │ → Algorithm tốt + data structure tệ = CHẬM! │             ║
║  │ → Phải XÉT CẢ HAI cùng lúc!                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 9.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: EFFICIENCY & PERFORMANCE                    │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Performance QUAN TRỌNG:                │            │
    │  │    → Battery (race to sleep!)             │            │
    │  │    → Data center (compute per watt!)      │            │
    │  │    → Ngôn ngữ cho CONTROL, ko cho perf!  │            │
    │  │                                          │            │
    │  │ 2. EFFICIENCY = Algorithms:               │            │
    │  │    → Làm ÍT VIỆC hơn!                   │            │
    │  │    → Chọn algorithm đúng Big-O!          │            │
    │  │    → Thói quen: ALWAYS do less work!      │            │
    │  │                                          │            │
    │  │ 3. PERFORMANCE = Data Structures:         │            │
    │  │    → Làm CÙNG việc NHANH hơn!            │            │
    │  │    → Cache locality = TẤT CẢ!            │            │
    │  │    → Contiguous > Linked!                 │            │
    │  │    → Slice > Linked List!                 │            │
    │  │                                          │            │
    │  │ 4. CPU Memory Hierarchy:                  │            │
    │  │    → L1(1ns) → RAM(100ns) = 100x!        │            │
    │  │    → 50% CPU time = CHỜ DATA!            │            │
    │  │    → Data nhỏ + liên tục = NHANH!        │            │
    │  │                                          │            │
    │  │ 5. Algorithms + Data Structures = CẶP ĐÔI│            │
    │  │    → Không tách rời được!                │            │
    │  │    → Data structure → inform algorithm!   │            │
    │  │    → Phải xét CẢ HAI cùng lúc!          │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  Chandler Carruth:                                       │
    │  "Efficiency comes through ALGORITHMS.                   │
    │   Performance comes through DATA STRUCTURES.             │
    │   Discontinuous data structures are the ROOT             │
    │   OF ALL performance evil."                              │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
