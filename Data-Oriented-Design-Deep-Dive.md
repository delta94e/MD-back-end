# Data-Oriented Design — Thiết Kế Hướng Dữ Liệu

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài talk "Data-Oriented Design and C++" — Mike Acton (Insomniac Games, CppCon 2014)
> **Góc nhìn:** Data là trung tâm, Hardware là nền tảng, Solve the real problem
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                              | Mô tả                                       |
| --- | ----------------------------------- | ------------------------------------------- |
| §1  | Mục đích của MỌI chương trình       | Transform DATA — không phải viết code!      |
| §2  | Ba lời nói dối lớn                  | Software ≠ Platform, Code ≠ Data            |
| §3  | Hiểu Data = Hiểu Problem            | Nếu không hiểu data, không hiểu vấn đề      |
| §4  | Where There Is One, There Are Many  | Không bao giờ chỉ có 1, luôn có NHIỀU       |
| §5  | Hardware Là Nền Tảng                | CPU, Cache, Memory — phải hiểu machine      |
| §6  | Data-Oriented Design trong thực tế  | Struct of Arrays, Data Transforms, Batching |
| §7  | Áp Dụng Cho Go                      | Value semantics, slice, struct layout       |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior | Ôn tập & thực hành                          |

---

## §1. Mục Đích Của MỌI Chương Trình

```
╔═══════════════════════════════════════════════════════════════╗
║   MỤC ĐÍCH = TRANSFORM DATA!                                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Mike Acton:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "The purpose of ALL programs, and ALL parts  │             ║
║  │  of those programs, is to TRANSFORM DATA     │             ║
║  │  from one form to another."                  │             ║
║  │                                              │             ║
║  │ → MỌI chương trình = biến đổi DATA!        │             ║
║  │ → Input data → xử lý → Output data!        │             ║
║  │ → Code chỉ là CÔNG CỤ biến đổi data!      │             ║
║  │ → Data mới là NHÂN VẬT CHÍNH!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ:                                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌──────────┐    ┌──────────┐    ┌────────┐ │             ║
║  │  │ INPUT    │    │ TRANSFORM│    │ OUTPUT │ │             ║
║  │  │ DATA     │───→│ (code)   │───→│ DATA   │ │             ║
║  │  │          │    │          │    │        │ │             ║
║  │  └──────────┘    └──────────┘    └────────┘ │             ║
║  │                                              │             ║
║  │  VÍ DỤ:                                      │             ║
║  │  → Web server: HTTP Request → Response      │             ║
║  │  → Game render: 3D models → Pixels          │             ║
║  │  → Database: Query → Result set             │             ║
║  │  → API: JSON input → JSON output            │             ║
║  │  → Compiler: Source code → Machine code     │             ║
║  │                                              │             ║
║  │  TẤT CẢ đều là TRANSFORM DATA!             │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  HỆ QUẢ:                                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Muốn giải quyết vấn đề?                 │             ║
║  │   HIỂU DATA TRƯỚC, viết code SAU!          │             ║
║  │ → Data QUYẾT ĐỊNH mọi thứ:                 │             ║
║  │   • Data format → algorithm                  │             ║
║  │   • Data size → data structure               │             ║
║  │   • Data access pattern → memory layout      │             ║
║  │   • Data lifetime → allocation strategy      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Ba Lời Nói Dối Lớn (Three Big Lies)

```
╔═══════════════════════════════════════════════════════════════╗
║   THREE BIG LIES — BA LỜI NÓI DỐI LỚN                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  LIE #1: "SOFTWARE IS THE PLATFORM"                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ ❌ SAI: "Tôi viết code cho OS, cho runtime, │             ║
║  │    cho framework, cho virtual machine"        │             ║
║  │                                              │             ║
║  │ ✅ ĐÚNG: "Tôi viết code cho HARDWARE!"      │             ║
║  │                                              │             ║
║  │  ┌─────────────────────────────────┐         │             ║
║  │  │ Your Code                       │         │             ║
║  │  ├─────────────────────────────────┤         │             ║
║  │  │ Go Runtime                      │         │             ║
║  │  ├─────────────────────────────────┤         │             ║
║  │  │ Operating System                │         │             ║
║  │  ├─────────────────────────────────┤         │             ║
║  │  │ ██████ HARDWARE ██████████████  │ ← ĐÂY! │             ║
║  │  │ CPU, Cache, Memory, Disk        │         │             ║
║  │  └─────────────────────────────────┘         │             ║
║  │                                              │             ║
║  │  → Hardware là PLATFORM THẬT!               │             ║
║  │  → Không hiểu hardware = code CHẬM!        │             ║
║  │  → CPU cache, memory layout, pipeline!      │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  LIE #2: "CODE DESIGNS DATA"                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ ❌ SAI: Thiết kế class/struct trước,        │             ║
║  │    rồi data phải "fit" vào design            │             ║
║  │                                              │             ║
║  │ ✅ ĐÚNG: DATA quyết định CODE!              │             ║
║  │                                              │             ║
║  │  ❌ OOP cổ điển:                             │             ║
║  │  "Tôi có class Animal → Dog, Cat..."         │             ║
║  │  → Thiết kế hierarchy trước!                │             ║
║  │  → Data phải nhồi vào cho vừa!             │             ║
║  │                                              │             ║
║  │  ✅ Data-Oriented:                            │             ║
║  │  "Tôi có 10,000 entities cần render"          │             ║
║  │  → Data: positions[], colors[], meshIDs[]    │             ║
║  │  → Code: batch process arrays!               │             ║
║  │  → Data QUYẾT ĐỊNH code structure!          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  LIE #3: "CODE IS MORE IMPORTANT THAN DATA"                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ ❌ SAI: "Code architecture quan trọng nhất!  │             ║
║  │    Design patterns! Clean code! SOLID!"       │             ║
║  │                                              │             ║
║  │ ✅ ĐÚNG: DATA quan trọng hơn code!          │             ║
║  │                                              │             ║
║  │  → Code có thể sai → fix!                  │             ║
║  │  → Data sai → MẤT DATA = game over!         │             ║
║  │  → Code đẹp + data layout tệ = CHẬM!      │             ║
║  │  → Code "xấu" + data layout tốt = NHANH!   │             ║
║  │                                              │             ║
║  │  → Ưu tiên: DATA > CODE > mọi thứ khác!   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Hiểu Data = Hiểu Problem

```
╔═══════════════════════════════════════════════════════════════╗
║   HIỂU DATA = HIỂU PROBLEM!                                  ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Mike Acton:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "If you don't understand the DATA,           │             ║
║  │  you don't understand the PROBLEM."          │             ║
║  │                                              │             ║
║  │ "If you don't understand the COST of         │             ║
║  │  solving the problem, you can't REASON       │             ║
║  │  about the problem."                         │             ║
║  │                                              │             ║
║  │ "If you don't understand the HARDWARE,       │             ║
║  │  you can't reason about the COST."           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CHUỖI SUY LUẬN:                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Hiểu Hardware                               │             ║
║  │      ↓                                       │             ║
║  │  Hiểu Cost (cache miss, allocation, etc.)    │             ║
║  │      ↓                                       │             ║
║  │  Hiểu Data (layout, access pattern, size)    │             ║
║  │      ↓                                       │             ║
║  │  Hiểu Problem (thực sự cần gì?)            │             ║
║  │      ↓                                       │             ║
║  │  Viết SOLUTION đúng!                         │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CÂU HỎI CẦN TRẢ LỜI TRƯỚC KHI VIẾT CODE:                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  1. Data ĐẦU VÀO là gì? Format? Size?      │             ║
║  │  2. Data ĐẦU RA cần gì? Format? Size?      │             ║
║  │  3. Có bao nhiêu phần tử? (1? 100? 1M?)    │             ║
║  │  4. Access pattern? (Sequential? Random?)    │             ║
║  │  5. Hot path? (90% thời gian chạy ở đâu?)  │             ║
║  │  6. Data thay đổi bao lâu 1 lần?           │             ║
║  │  7. Data nào ĐI CÙNG nhau? (locality!)      │             ║
║  │  8. Data nào KHÔNG cần? (loại bỏ!)          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VÍ DỤ THỰC TẾ — GO API SERVER:                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Trước khi code, HỎI:                        │             ║
║  │                                              │             ║
║  │ → Request body trung bình BAO LỚN?          │             ║
║  │   100B? 10KB? 1MB?                            │             ║
║  │   → Quyết định buffer size!                  │             ║
║  │                                              │             ║
║  │ → Có bao nhiêu concurrent requests?          │             ║
║  │   10? 1000? 100K?                             │             ║
║  │   → Quyết định goroutine pool!               │             ║
║  │                                              │             ║
║  │ → Database result set BAO LỚN?              │             ║
║  │   10 rows? 10K rows?                          │             ║
║  │   → Quyết định pre-allocate slice!           │             ║
║  │                                              │             ║
║  │ → 90% requests hit ENDPOINT NÀO?             │             ║
║  │   → Optimize HOT PATH đó!                   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Where There Is One, There Are Many

```
╔═══════════════════════════════════════════════════════════════╗
║   "WHERE THERE IS ONE, THERE ARE MANY"                        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  NGUYÊN TẮC:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Nếu có 1 thứ, sẽ có NHIỀU thứ giống nó!" │             ║
║  │                                              │             ║
║  │ → 1 entity → HÀNG NGHÌN entities!           │             ║
║  │ → 1 request → TRIỆU requests!               │             ║
║  │ → 1 record → TRIỆU records!                 │             ║
║  │                                              │             ║
║  │ → THIẾT KẾ CHO NHIỀU, không cho 1!          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ❌ THIẾT KẾ CHO 1 (Object-Oriented):                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ type Enemy struct {                           │             ║
║  │     Name     string                           │             ║
║  │     Texture  *Texture     // 8 bytes pointer  │             ║
║  │     X, Y, Z  float64     // 24 bytes          │             ║
║  │     Health   int          // 8 bytes           │             ║
║  │     Speed    float64      // 8 bytes           │             ║
║  │     AI       *AIBehavior  // 8 bytes pointer  │             ║
║  │     Sounds   []Sound      // 24 bytes slice   │             ║
║  │     // ... 100+ bytes mỗi enemy!            │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ // Xử lý từng enemy MỘT!                   │             ║
║  │ for _, e := range enemies {                   │             ║
║  │     e.Update()        // Virtual dispatch!    │             ║
║  │     e.Render()        // Cache miss!          │             ║
║  │     e.PlaySound()     // Random access!       │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ → Memory: 100 bytes × 10,000 = RẢI RÁC!    │             ║
║  │ → Mỗi enemy = cache miss!                   │             ║
║  │ → CHẬM!                                      │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ✅ THIẾT KẾ CHO NHIỀU (Data-Oriented):                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ // TÁCH DATA theo access pattern!            │             ║
║  │ type Positions struct {                       │             ║
║  │     X []float64    // contiguous!             │             ║
║  │     Y []float64    // contiguous!             │             ║
║  │     Z []float64    // contiguous!             │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ type Healths []int        // contiguous!      │             ║
║  │ type Speeds  []float64    // contiguous!      │             ║
║  │                                              │             ║
║  │ // Batch process!                             │             ║
║  │ func UpdatePositions(pos *Positions,          │             ║
║  │     speeds []float64, dt float64) {           │             ║
║  │     for i := range pos.X {                    │             ║
║  │         pos.X[i] += speeds[i] * dt            │             ║
║  │     }                                         │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ → Memory: mỗi array = LIÊN TỤC!            │             ║
║  │ → CPU prefetch = siêu nhanh!                │             ║
║  │ → Chỉ load data CẦN cho task đang chạy!    │             ║
║  │ → Update position? CHỈ load positions+speeds!│             ║
║  │ → KHÔNG load texture, sounds, AI!            │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ SO SÁNH MEMORY:                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ ❌ Array of Structs (AoS):                   │             ║
║  │ [Name|Tex|X|Y|Z|HP|Spd|AI|Snd]              │             ║
║  │ [Name|Tex|X|Y|Z|HP|Spd|AI|Snd]              │             ║
║  │ [Name|Tex|X|Y|Z|HP|Spd|AI|Snd]              │             ║
║  │ → Load CẢ struct dù chỉ cần X,Y,Z!        │             ║
║  │ → Lãng phí cache!                           │             ║
║  │                                              │             ║
║  │ ✅ Struct of Arrays (SoA):                   │             ║
║  │ X:   [x1|x2|x3|x4|x5|x6|x7|x8|...]         │             ║
║  │ Y:   [y1|y2|y3|y4|y5|y6|y7|y8|...]          │             ║
║  │ Z:   [z1|z2|z3|z4|z5|z6|z7|z8|...]          │             ║
║  │ HP:  [h1|h2|h3|h4|h5|h6|h7|h8|...]          │             ║
║  │ → CHỈ load arrays CẦN THIẾT!               │             ║
║  │ → Sequential access = cache friendly!        │             ║
║  │ → CPU prefetch hoàn hảo!                    │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Hardware Là Nền Tảng

```
╔═══════════════════════════════════════════════════════════════╗
║   HARDWARE = PLATFORM THẬT SỰ!                               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Mike Acton:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "The HARDWARE is the platform.               │             ║
║  │  NOT the software."                          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  THỰC TẾ CPU:                                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  CPU CAN:                                     │             ║
║  │  → Execute ~4 instructions per cycle!        │             ║
║  │  → Pipeline: fetch-decode-execute song song! │             ║
║  │  → Prefetch: dự đoán data tiếp theo!       │             ║
║  │  → Branch prediction: dự đoán if/else!      │             ║
║  │                                              │             ║
║  │  CPU CAN'T:                                   │             ║
║  │  → Lấy data từ RAM nhanh! (~100 cycles!)    │             ║
║  │  → Predict random access patterns!            │             ║
║  │  → Hide cache miss latency!                  │             ║
║  │                                              │             ║
║  │  → HIỂU CPU = viết code CPU YÊU THÍCH!     │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  DATA ACCESS PATTERNS:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ✅ SEQUENTIAL (tuần tự):                    │             ║
║  │  → for i := range data { data[i]... }        │             ║
║  │  → CPU prefetch DỰ ĐOÁN được!              │             ║
║  │  → Cache line load = nhiều phần tử FREE!    │             ║
║  │  → NHANH NHẤT!                               │             ║
║  │                                              │             ║
║  │  ⚠️ STRIDED (bước nhảy cố định):             │             ║
║  │  → data[0], data[4], data[8], ...             │             ║
║  │  → CPU CÓ THỂ predict stride!               │             ║
║  │  → Kém hơn sequential, nhưng OK!            │             ║
║  │                                              │             ║
║  │  ❌ RANDOM (ngẫu nhiên):                     │             ║
║  │  → data[7], data[100], data[3], data[999]    │             ║
║  │  → CPU KHÔNG predict được!                  │             ║
║  │  → MỖI access = cache miss = 100ns!          │             ║
║  │  → Linked list, hash map = RANDOM!            │             ║
║  │  → CHẬM NHẤT!                                │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SOLVE FOR THE COMMON CASE:                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ "Solve for the 90% case FIRST."              │             ║
║  │                                              │             ║
║  │ → 90% requests = READ → optimize READ!       │             ║
║  │ → 90% entities = ALIVE → optimize ALIVE!     │             ║
║  │ → 90% data = SMALL → optimize SMALL!         │             ║
║  │                                              │             ║
║  │ → ĐỪNG optimize edge cases trước!           │             ║
║  │ → Profile → tìm hot path → optimize ĐÓ!   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Data-Oriented Design Trong Thực Tế

```
╔═══════════════════════════════════════════════════════════════╗
║   DATA-ORIENTED DESIGN — THỰC HÀNH                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  NGUYÊN TẮC 1: TÁCH HOT DATA & COLD DATA                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ HOT data = truy cập THƯỜNG XUYÊN            │             ║
║  │ COLD data = truy cập HIẾM KHI               │             ║
║  │                                              │             ║
║  │ ❌ Gộp chung:                                 │             ║
║  │ type User struct {                            │             ║
║  │     ID        int64     // HOT               │             ║
║  │     Name      string    // HOT               │             ║
║  │     Email     string    // HOT               │             ║
║  │     Bio       string    // COLD (hiếm dùng) │             ║
║  │     Avatar    []byte    // COLD (lớn!)      │             ║
║  │     Settings  Settings  // COLD              │             ║
║  │     CreatedAt time.Time // COLD              │             ║
║  │ }                                             │             ║
║  │ // → Load CẢ struct dù chỉ cần ID+Name!   │             ║
║  │                                              │             ║
║  │ ✅ Tách riêng:                               │             ║
║  │ type UserCore struct {   // HOT — nhỏ gọn! │             ║
║  │     ID    int64                               │             ║
║  │     Name  string                              │             ║
║  │     Email string                              │             ║
║  │ }                                             │             ║
║  │ type UserProfile struct { // COLD — load khi cần│          ║
║  │     Bio       string                          │             ║
║  │     Avatar    []byte                          │             ║
║  │     Settings  Settings                        │             ║
║  │     CreatedAt time.Time                       │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NGUYÊN TẮC 2: BATCH PROCESSING                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ ❌ Xử lý TỪNG MỘT:                          │             ║
║  │ for _, user := range users {                  │             ║
║  │     db.UpdateOne(user) // 1 query per user!  │             ║
║  │ }                                             │             ║
║  │ // 10,000 users = 10,000 queries!             │             ║
║  │                                              │             ║
║  │ ✅ Batch:                                     │             ║
║  │ db.UpdateBatch(users) // 1 query for ALL!    │             ║
║  │ // 10,000 users = 1 query!                    │             ║
║  │                                              │             ║
║  │ → Batch = ít overhead per item!              │             ║
║  │ → Network round trip: 1 thay vì 10,000!     │             ║
║  │ → Data locality trong 1 operation!           │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NGUYÊN TẮC 3: KHÔNG CÓ GIẢI PHÁP LÝ TƯỞNG                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ Mike Acton:                                   │             ║
║  │ "There is NO idealized, abstract solution.   │             ║
║  │  Different problems require DIFFERENT         │             ║
║  │  solutions."                                  │             ║
║  │                                              │             ║
║  │ → Data KHÁC = Problem KHÁC = Solution KHÁC! │             ║
║  │ → Không có "one size fits all"!              │             ║
║  │ → 100 users vs 1M users = BÀI TOÁN KHÁC!   │             ║
║  │ → Đo lường → quyết định → tối ưu!        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Áp Dụng Cho Go

```
╔═══════════════════════════════════════════════════════════════╗
║   DATA-ORIENTED DESIGN TRONG GO                               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  GO CÓ LỢI THẾ TỰ NHIÊN:                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ 1. Value Semantics:                           │             ║
║  │    → Struct value = data TRÊN STACK!          │             ║
║  │    → Contiguous memory = cache friendly!     │             ║
║  │    → Không pointer chasing!                  │             ║
║  │                                              │             ║
║  │ 2. Slice = Contiguous Array:                  │             ║
║  │    → []int = BACKED by contiguous array!     │             ║
║  │    → for range = sequential access!           │             ║
║  │    → CPU prefetch = NHANH!                   │             ║
║  │                                              │             ║
║  │ 3. Struct Layout Kiểm Soát Được:            │             ║
║  │    → Field order = memory order!              │             ║
║  │    → Có thể tối ưu padding!                 │             ║
║  │                                              │             ║
║  │ 4. Không có OOP Inheritance:                  │             ║
║  │    → Tự nhiên hướng đến composition!        │             ║
║  │    → Không virtual dispatch overhead!         │             ║
║  │    → Interface dispatch = 1 indirect call!   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SLICE OF VALUES vs SLICE OF POINTERS:                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ ✅ []Point (values — CONTIGUOUS!):           │             ║
║  │ ┌──────┬──────┬──────┬──────┬──────┐        │             ║
║  │ │{X,Y} │{X,Y} │{X,Y} │{X,Y} │{X,Y} │        │             ║
║  │ └──────┴──────┴──────┴──────┴──────┘        │             ║
║  │ → ALL data liền nhau!                       │             ║
║  │ → 1 cache line = nhiều Points!               │             ║
║  │ → for range = sequential = NHANH!            │             ║
║  │                                              │             ║
║  │ ❌ []*Point (pointers — SCATTERED!):         │             ║
║  │ ┌──┬──┬──┬──┬──┐    Heap:                   │             ║
║  │ │ * │ * │ * │ * │ * │    {X,Y} ← 0x100     │             ║
║  │ └─┬┴─┬┴─┬┴─┬┴─┬┘    {X,Y} ← 0x8F0     │             ║
║  │   │  │  │  │  │      {X,Y} ← 0x350     │             ║
║  │   ↓  ↓  ↓  ↓  ↓      → RANDOM addresses! │             ║
║  │ → Mỗi deref = cache miss!                  │             ║
║  │ → CHẬM hơn nhiều!                          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GO STRUCT PADDING:                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ // ❌ Bad layout — 32 bytes (có padding!)    │             ║
║  │ type Bad struct {                             │             ║
║  │     a bool    // 1 byte  + 7 padding         │             ║
║  │     b int64   // 8 bytes                      │             ║
║  │     c bool    // 1 byte  + 7 padding         │             ║
║  │     d int64   // 8 bytes                      │             ║
║  │ }                                             │             ║
║  │ // Total: 32 bytes!                           │             ║
║  │                                              │             ║
║  │ // ✅ Good layout — 24 bytes (ít padding!)   │             ║
║  │ type Good struct {                            │             ║
║  │     b int64   // 8 bytes                      │             ║
║  │     d int64   // 8 bytes                      │             ║
║  │     a bool    // 1 byte                       │             ║
║  │     c bool    // 1 byte  + 6 padding         │             ║
║  │ }                                             │             ║
║  │ // Total: 24 bytes! Tiết kiệm 25%!          │             ║
║  │                                              │             ║
║  │ → Sắp xếp fields LỚN TRƯỚC, NHỎ SAU!      │             ║
║  │ → Dùng unsafe.Sizeof() để kiểm tra!        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GO PRE-ALLOCATION PATTERN:                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ // LUÔN pre-allocate khi biết size!          │             ║
║  │ users := make([]User, 0, expectedCount)       │             ║
║  │ cache := make(map[string]*Data, expectedSize) │             ║
║  │ buf := make([]byte, 0, 4096)                  │             ║
║  │                                              │             ║
║  │ // LUÔN dùng sync.Pool cho hot objects!      │             ║
║  │ var bufPool = sync.Pool{                      │             ║
║  │     New: func() interface{} {                 │             ║
║  │         return make([]byte, 0, 4096)          │             ║
║  │     },                                        │             ║
║  │ }                                             │             ║
║  │ buf := bufPool.Get().([]byte)                 │             ║
║  │ defer bufPool.Put(buf[:0])                    │             ║
║  │                                              │             ║
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
║  Q1: "Data-Oriented Design là gì?"                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = Thiết kế phần mềm XOAY QUANH DATA!        │             ║
║  │ → Purpose = transform data!                   │             ║
║  │ → Hiểu data TRƯỚC, viết code SAU!           │             ║
║  │ → Data layout quyết định performance!        │             ║
║  │ → Khác OOP: OOP xoay quanh OBJECTS           │             ║
║  │   DOD xoay quanh DATA TRANSFORMATIONS!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Array of Structs vs Struct of Arrays?"                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ AoS: [{X,Y,HP}, {X,Y,HP}, ...] = 1 struct   │             ║
║  │ → Load CẢ struct dù chỉ cần 1 field!       │             ║
║  │ → Lãng phí cache!                           │             ║
║  │                                              │             ║
║  │ SoA: {X:[], Y:[], HP:[]}                     │             ║
║  │ → CHỈ load array cần thiết!                  │             ║
║  │ → Sequential = cache friendly!                │             ║
║  │                                              │             ║
║  │ Go: dùng AoS cho small struct thường xuyên!  │             ║
║  │ → SoA khi cần batch process 1-2 fields!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Struct padding trong Go?"                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Go align fields theo natural alignment!    │             ║
║  │ → int64 align 8 bytes, int32 align 4 bytes!  │             ║
║  │ → bool giữa 2 int64 = 7 bytes padding!      │             ║
║  │ → Fix: sắp xếp LỚN → NHỎ!                 │             ║
║  │ → Kiểm tra: unsafe.Sizeof(MyStruct{})        │             ║
║  │ → 10,000 structs × 8 bytes padding =         │             ║
║  │   80KB LÃNG PHÍ!                             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "[]T vs []*T — khi nào dùng gì?"                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ []T (values):                                 │             ║
║  │ → Contiguous! Cache friendly!                 │             ║
║  │ → Tốt cho small-medium structs (<= 64 bytes)│             ║
║  │ → Iterate NHANH!                              │             ║
║  │                                              │             ║
║  │ []*T (pointers):                              │             ║
║  │ → Khi struct RẤT LỚN (> 256 bytes)          │             ║
║  │ → Khi cần share reference giữa collections  │             ║
║  │ → Khi cần polymorphism (interface)           │             ║
║  │ → Cache miss mỗi dereference!               │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q5: "Three Big Lies giải thích?"                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Software is the platform → SAI!           │             ║
║  │    HARDWARE là platform! Hiểu CPU + cache!  │             ║
║  │                                              │             ║
║  │ 2. Code designs data → SAI!                  │             ║
║  │    DATA quyết định code! Hiểu data trước!  │             ║
║  │                                              │             ║
║  │ 3. Code > Data → SAI!                        │             ║
║  │    DATA quan trọng hơn! Layout > Architecture!│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: DATA-ORIENTED DESIGN                        │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Purpose = TRANSFORM DATA:              │            │
    │  │    → Mọi program = input → output!       │            │
    │  │    → Data là NHÂN VẬT CHÍNH!             │            │
    │  │                                          │            │
    │  │ 2. Three Big Lies:                        │            │
    │  │    → Hardware IS the platform!             │            │
    │  │    → Data designs CODE!                   │            │
    │  │    → Data > Code!                         │            │
    │  │                                          │            │
    │  │ 3. Hiểu Data = Hiểu Problem:             │            │
    │  │    → Size? Pattern? Frequency? Hot path? │            │
    │  │    → Hỏi TRƯỚC, code SAU!                │            │
    │  │                                          │            │
    │  │ 4. Where There Is One, There Are Many:    │            │
    │  │    → Thiết kế cho BATCH, không cho 1!    │            │
    │  │    → SoA > AoS cho batch processing!      │            │
    │  │                                          │            │
    │  │ 5. Áp dụng Go:                           │            │
    │  │    → Value semantics = contiguous!         │            │
    │  │    → []T > []*T cho cache!                │            │
    │  │    → Struct padding = sắp xếp fields!    │            │
    │  │    → Pre-allocate + sync.Pool!            │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  Mike Acton:                                             │
    │  "If you don't understand the DATA,                      │
    │   you don't understand the PROBLEM."                     │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
