# Design Philosophy On Integrity — Triết Lý Thiết Kế Về Tính Toàn Vẹn

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Design Philosophy On Integrity" của William Kennedy (2017)
> **Góc nhìn:** Integrity = nguyên tắc #1 trong thiết kế phần mềm Go
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                              | Mô tả                                   |
| --- | ----------------------------------- | --------------------------------------- |
| §1  | Integrity là gì?                    | Định nghĩa & tầm quan trọng             |
| §2  | Micro vs Macro Integrity            | 2 cấp độ toàn vẹn trong code            |
| §3  | 92% Critical Failures               | Nghiên cứu lỗi hệ thống phân tán        |
| §4  | Ma trận Developer Behavior          | Ignorance vs Carelessness × Deliberate  |
| §5  | Write Less Code — Viết ít code hơn  | Giảm bug bằng cách viết ít hơn          |
| §6  | Go & Integrity — Tại sao Go đúng    | Error handling, Type system, Simplicity |
| §7  | Áp dụng thực tế cho Go services     | Patterns & practices                    |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior | Ôn tập & thực hành                      |

---

## §1. Integrity Là Gì?

### 1.1 Định nghĩa của William Kennedy

```
╔═══════════════════════════════════════════════════════════════╗
║   INTEGRITY — TÍNH TOÀN VẸN                                  ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  William Kennedy:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "The ACCURACY and CONSISTENCY of the code    │             ║
║  │  performing every read, write and the        │             ║
║  │  execution of every instruction.             │             ║
║  │                                              │             ║
║  │  Just as important, writing LESS CODE and    │             ║
║  │  knowing the ERROR HANDLING code IS the      │             ║
║  │  main code."                                 │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  DỊCH:                                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Sự CHÍNH XÁC và NHẤT QUÁN của code trong   │             ║
║  │  mọi thao tác đọc, ghi và thực thi.         │             ║
║  │                                              │             ║
║  │  Quan trọng không kém: viết ÍT CODE hơn     │             ║
║  │  và hiểu rằng code XỬ LÝ LỖI CHÍNH LÀ     │             ║
║  │  code chính."                                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  2 TRỤ CỘT CỦA INTEGRITY:                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌────────────────┐   ┌────────────────┐    │             ║
║  │  │ TRỤ 1:         │   │ TRỤ 2:         │    │             ║
║  │  │ Mọi thao tác   │   │ Viết ÍT code   │    │             ║
║  │  │ đọc/ghi        │   │ Error handling  │    │             ║
║  │  │ CHÍNH XÁC      │   │ = MAIN CODE    │    │             ║
║  │  │ NHẤT QUÁN      │   │                 │    │             ║
║  │  │ HIỆU QUẢ       │   │                 │    │             ║
║  │  └────────────────┘   └────────────────┘    │             ║
║  │       │                      │               │             ║
║  │       ▼                      ▼               │             ║
║  │  ┌──────────────────────────────────────┐    │             ║
║  │  │ RESILIENT (bền bỉ) +                 │    │             ║
║  │  │ RELIABLE (đáng tin cậy) SOFTWARE    │    │             ║
║  │  └──────────────────────────────────────┘    │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  "Integrity is much more than just a BUZZWORD,               ║
║   it is a DRIVING PRINCIPLE in everything I do,              ║
║   both in CODE and in LIFE." — Kennedy                       ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Micro vs Macro Integrity

### 2.1 Hai cấp độ toàn vẹn

```
╔═══════════════════════════════════════════════════════════════╗
║   MICRO vs MACRO INTEGRITY                                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Mỗi dòng code bạn viết thực hiện 1 trong 2:                ║
║                                                               ║
║  ┌─────────────────────────────────────────────┐              ║
║  │ MICRO LEVEL — Từng dòng code                 │              ║
║  │ (Allocating, Reading, Writing memory)        │              ║
║  ├─────────────────────────────────────────────┤              ║
║  │                                              │              ║
║  │ → Mỗi dòng = cấp phát, đọc, hoặc ghi      │              ║
║  │   vào bộ nhớ                                 │              ║
║  │ → Variables và Statements điều khiển         │              ║
║  │ → TYPE SYSTEM = công cụ bảo vệ!            │              ║
║  │                                              │              ║
║  │ Ví dụ Go:                                     │              ║
║  │   var count int    // allocate 8 bytes       │              ║
║  │   count = 42       // write to memory        │              ║
║  │   fmt.Println(count) // read from memory     │              ║
║  │                                              │              ║
║  │ Type system CHẶN lỗi micro:                  │              ║
║  │   var count int                               │              ║
║  │   count = "hello" // ← COMPILE ERROR!       │              ║
║  │   → Go KHÔNG cho viết sai kiểu vào memory! │              ║
║  └─────────────────────────────────────────────┘              ║
║                                                               ║
║  ┌─────────────────────────────────────────────┐              ║
║  │ MACRO LEVEL — Luồng dữ liệu                 │              ║
║  │ (Data Transformations)                       │              ║
║  ├─────────────────────────────────────────────┤              ║
║  │                                              │              ║
║  │ → Functions nhận INPUT → trả OUTPUT          │              ║
║  │ → Data transformations phải CHÍNH XÁC       │              ║
║  │ → VIẾT ÍT CODE + ERROR HANDLING = công cụ! │              ║
║  │                                              │              ║
║  │ Ví dụ Go:                                     │              ║
║  │   func transform(input []byte) (Result, error)│             ║
║  │   → Input → Validate → Process → Output    │              ║
║  │   → MỖI BƯỚC có thể lỗi = MỖI BƯỚC check! │              ║
║  └─────────────────────────────────────────────┘              ║
║                                                               ║
║  TÓM TẮT:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ MICRO: Type System → chính xác từng byte    │             ║
║  │ MACRO: Error Handling → chính xác từng step  │             ║
║  │                                              │             ║
║  │ → CẢ HAI đều CẦN cho Integrity!            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 2.2 Go Type System — Micro Integrity

```go
// ═══ GO TYPE SYSTEM — BẢO VỆ MICRO INTEGRITY ═══

// 1. STRONG TYPING — Không tự chuyển đổi kiểu!
var age int = 25
var name string = "Go"
// age = name  ← COMPILE ERROR! Không cho trộn kiểu!

// 2. NAMED TYPES — Tạo ý nghĩa cho dữ liệu!
type UserID int64
type OrderID int64

func GetUser(id UserID) (*User, error) { /* ... */ }

var uid UserID = 123
var oid OrderID = 456
// GetUser(oid) ← COMPILE ERROR!
// → Dù cùng int64, KHÔNG THỂ trộn UserID với OrderID!
// → Type system BẢO VỆ khỏi logic errors!

// 3. STRUCT — Nhóm dữ liệu có ý nghĩa!
type Money struct {
    Amount   int64  // cents, không dùng float!
    Currency string // "USD", "VND"
}
// → Không thể cộng USD + VND!
// → Type system ENFORCE business rules!

// 4. INTERFACE — Contract rõ ràng!
type Reader interface {
    Read(p []byte) (n int, err error)
}
// → BẮT BUỘC implement đúng signature!
// → Compile-time verification!
```

---

## §3. 92% Critical Failures — Nghiên Cứu Quan Trọng

### 3.1 Dữ liệu gây sốc

```
╔═══════════════════════════════════════════════════════════════╗
║   92% CRITICAL FAILURES — TỪ BAD ERROR HANDLING              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Nghiên cứu: Ding Yuan et al.                                ║
║  Hệ thống: Cassandra, HBase, HDFS, MapReduce, Redis         ║
║  Tìm thấy: 48 CRITICAL FAILURES                              ║
║                                                               ║
║  KẾT QUẢ:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ╔══════════════════════════════════════╗     │             ║
║  │  ║  92% từ BAD ERROR HANDLING!         ║     │             ║
║  │  ╚══════════════════════════════════════╝     │             ║
║  │                                              │             ║
║  │  Chia nhỏ 92%:                               │             ║
║  │  ┌────────────────────────────────────┐      │             ║
║  │  │ 35% │ INCORRECT handling           │      │             ║
║  │  │     │ ┌──────────────────────────┐ │      │             ║
║  │  │     │ │ 25% ignore error hoàn toàn│ │      │             ║
║  │  │     │ │  8% catch wrong exception │ │      │             ║
║  │  │     │ │  2% incomplete TODOs      │ │      │             ║
║  │  │     │ └──────────────────────────┘ │      │             ║
║  │  ├─────┼──────────────────────────────┤      │             ║
║  │  │ 57% │ SYSTEM SPECIFIC              │      │             ║
║  │  │     │ ┌──────────────────────────┐ │      │             ║
║  │  │     │ │ 23% easily detectable    │ │      │             ║
║  │  │     │ │ 34% complex bugs         │ │      │             ║
║  │  │     │ └──────────────────────────┘ │      │             ║
║  │  └─────┴──────────────────────────────┘      │             ║
║  │                                              │             ║
║  │  8% │ LATENT HUMAN ERRORS                   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "92% of the critical failures could have     │             ║
║  │  been AVOIDED if the developers made         │             ║
║  │  INTEGRITY their FIRST PRIORITY."            │             ║
║  │                                              │             ║
║  │ → Nếu integrity là ƯU TIÊN SỐ 1            │             ║
║  │ → 92% critical failures TRÁNH ĐƯỢC!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VỀ EXCEPTION HANDLING:                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Exception handling LENDS ITSELF to making   │             ║
║  │  mistakes. It doesn't force you to think     │             ║
║  │  about error handling as part of the MAIN    │             ║
║  │  CODE but as an EXCEPTION."                  │             ║
║  │                                              │             ║
║  │ → Exception = "xử lý SAU, ở chỗ KHÁC"     │             ║
║  │ → Disconnect = LỖI!                         │             ║
║  │                                              │             ║
║  │ GO: Error handling = NGAY TẠI CHỖ!          │             ║
║  │ → if err != nil = MAIN CODE!                │             ║
║  │ → Không có "đâu đó khác" xử lý giùm!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  "Error handling in Go is TEDIOUS and                        ║
║   INCONVENIENT — BUT THAT'S THE POINT.                       ║
║   This is Go pushing you towards the one                     ║
║   thing you must NEVER ignore: INTEGRITY."                   ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Ma Trận Developer Behavior

### 4.1 Ignorance vs Carelessness

```
╔═══════════════════════════════════════════════════════════════╗
║   DEVELOPER BEHAVIOR MATRIX — MA TRẬN HÀNH VI               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Kennedy: "Developers operate in a matrix of                 ║
║  CARELESSNESS vs IGNORANCE vs INTENT."                       ║
║                                                               ║
║  (Twist trên Martin Fowler's Technical Debt Quadrant)        ║
║                                                               ║
║         NON-DELIBERATE          DELIBERATE                    ║
║         (Không cố ý)            (Cố ý)                       ║
║              │                      │                         ║
║  ┌───────────┼──────────┬───────────┼──────────┐             ║
║  │           │          │           │          │             ║
║  │  Learning /          │  Hacking /           │             ║
║  │  Prototyping         │  Guessing            │  IGNORANCE  ║
║  │                      │                      │  (Thiếu     ║
║  │  "Đang học,          │  "Hack cho nó       │   hiểu      ║
║  │   thử nghiệm"       │   chạy, không hiểu  │   biết)     ║
║  │                      │   hậu quả"           │             ║
║  │  → CHẤP NHẬN ĐƯỢC!  │  → CẦN LOẠI BỎ!    │             ║
║  │                      │                      │             ║
║  ├──────────────────────┼──────────────────────┤             ║
║  │                      │                      │             ║
║  │  Education           │  DANGEROUS           │  CARELESS-  ║
║  │  (Cần đào tạo)      │  SITUATION! ⚠️       │  NESS       ║
║  │                      │                      │  (Bất cẩn)  ║
║  │  "Vô tình bất cẩn,  │  "BIẾT là sai,      │             ║
║  │   cần học thêm"     │   VẪN CỐ Ý LÀM!"   │             ║
║  │                      │                      │             ║
║  │  → ĐÀO TẠO NGAY!   │  → KHÔNG THỂ        │             ║
║  │                      │    CHẤP NHẬN!        │             ║
║  └──────────────────────┴──────────────────────┘             ║
║                                                               ║
║  PHÂN TÍCH TỪNG GÓC:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Learning/Prototyping (OK):                 │             ║
║  │    → Junior đang học Go error handling       │             ║
║  │    → Prototype chưa cần production-ready     │             ║
║  │    → CẦN hướng dẫn, KHÔNG phải blame       │             ║
║  │                                              │             ║
║  │ 2. Hacking/Guessing (CẦN FIX):              │             ║
║  │    → Copy-paste từ StackOverflow không hiểu │             ║
║  │    → "Chạy rồi mà!" mà không hiểu tại sao │             ║
║  │    → CẦN code review + mentoring            │             ║
║  │                                              │             ║
║  │ 3. Education (ĐÀO TẠO):                      │             ║
║  │    → Vô tình quên check error               │             ║
║  │    → Không biết best practice                │             ║
║  │    → CẦN linter + pair programming          │             ║
║  │                                              │             ║
║  │ 4. DANGEROUS (⚠️ NGHIÊM TRỌNG):            │             ║
║  │    → BIẾT phải check error, CỐ Ý BỎ QUA!  │             ║
║  │    → "Deadline gấp, skip error handling!"    │             ║
║  │    → Kennedy: "BEYOND ME!" — Không thể      │             ║
║  │      hiểu nổi tại sao lại cố ý làm vậy!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  "We must ELIMINATE carelessness from                         ║
║   EVERYTHING we do." — Kennedy                               ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Write Less Code — Viết Ít Code Hơn

### 5.1 Bug rate theo dòng code

```
╔═══════════════════════════════════════════════════════════════╗
║   WRITE LESS CODE — ÍT CODE = ÍT BUG                        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  THỐNG KÊ NGÀNH:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Trung bình: 15–50 bugs / 1000 dòng code     │             ║
║  │                                              │             ║
║  │ → 10,000 dòng = 150–500 bugs!               │             ║
║  │ → 100,000 dòng = 1,500–5,000 bugs!          │             ║
║  │                                              │             ║
║  │ CÁCH GIẢM BUG ĐƠN GIẢN NHẤT:               │             ║
║  │ → VIẾT ÍT CODE HƠN!                        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Bjarne Stroustrup (C++ creator):                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Viết NHIỀU code hơn cần = code:              │             ║
║  │                                              │             ║
║  │  UGLY  → Tạo chỗ cho bugs ẨN NÁU          │             ║
║  │  LARGE → Đảm bảo tests KHÔNG ĐẦY ĐỦ       │             ║
║  │  SLOW  → Khuyến khích dùng SHORTCUTS        │             ║
║  │           và DIRTY TRICKS                     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GO ĐÃ THIẾT KẾ ĐỂ VIẾT ÍT:                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ✅ gofmt: 1 style duy nhất → 0 tranh luận  │             ║
║  │ ✅ Unused import = COMPILE ERROR!            │             ║
║  │ ✅ Unused variable = COMPILE ERROR!          │             ║
║  │ ✅ Không có generics phức tạp (trước 1.18)  │             ║
║  │ ✅ Không có inheritance → ít abstraction     │             ║
║  │ ✅ Composition over Inheritance              │             ║
║  │                                              │             ║
║  │ → Go ÉP bạn viết ít, đơn giản, rõ ràng!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SO SÁNH:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Java: 50 dòng cho "Hello World" web server  │             ║
║  │ Go:   15 dòng cho "Hello World" web server  │             ║
║  │                                              │             ║
║  │ → Ít code = ít bugs = ít bảo trì           │             ║
║  │ → Ít code = dễ đọc = dễ review             │             ║
║  │ → Ít code = INTEGRITY cao hơn!             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Go & Integrity — Tại Sao Go Đúng

### 6.1 Error handling = Integrity enforcement

```
╔═══════════════════════════════════════════════════════════════╗
║   GO ERROR HANDLING = INTEGRITY ENFORCEMENT                   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Kennedy:                                                      ║
║  "Error handling in Go is TEDIOUS and INCONVENIENT            ║
║   — but that's the POINT."                                    ║
║                                                               ║
║  SO SÁNH EXCEPTION vs GO ERROR:                               ║
║                                                               ║
║  EXCEPTION (Java/Python/C#):                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ try {                                         │             ║
║  │     step1();   // happy path                 │             ║
║  │     step2();   // happy path                 │             ║
║  │     step3();   // happy path                 │             ║
║  │ } catch (Exception e) {                       │             ║
║  │     // ĐÂU ĐÓ KHÁC, LÚC KHÁC!             │             ║
║  │     // → Disconnect giữa code và error!     │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ VẤN ĐỀ:                                      │             ║
║  │ → Error handling TÁCH RỜI khỏi code chính  │             ║
║  │ → Dễ QUÊN exception nào có thể xảy ra      │             ║
║  │ → Catch sai exception (8% failures!)        │             ║
║  │ → "Exception" = "ngoại lệ" = KHÔNG QUAN    │             ║
║  │   TRỌNG trong tâm trí dev!                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GO ERROR:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ r1, err := step1()                            │             ║
║  │ if err != nil {                               │             ║
║  │     return fmt.Errorf("step1: %w", err)      │             ║
║  │ }                                             │             ║
║  │ // Error handling NGAY TẠI CHỖ!             │             ║
║  │                                              │             ║
║  │ r2, err := step2(r1)                          │             ║
║  │ if err != nil {                               │             ║
║  │     return fmt.Errorf("step2: %w", err)      │             ║
║  │ }                                             │             ║
║  │ // NGAY TẠI CHỖ!                            │             ║
║  │                                              │             ║
║  │ ƯU ĐIỂM:                                     │             ║
║  │ → Error = part of MAIN CODE, NOT exception  │             ║
║  │ → VISIBLE: nhìn code = thấy error paths     │             ║
║  │ → FORCED: khó bỏ qua err (linters bắt)     │             ║
║  │ → CONTEXTUAL: wrap error = debug nhanh      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  "Integrity and security go HAND IN HAND."                   ║
║  → Integrity cần sự TEDIOUS giống security!                  ║
║  → Cả 2 không thể "skip cho nhanh"!                         ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 6.2 Go patterns cho Integrity

```go
// ═══ GO INTEGRITY PATTERNS ═══

// 1. ERROR CONTEXT WRAPPING — Debug nhanh!
func ProcessPayment(ctx context.Context, orderID string) error {
    order, err := fetchOrder(ctx, orderID)
    if err != nil {
        return fmt.Errorf("process payment: fetch order %s: %w",
            orderID, err)
    }
    // → Error message: "process payment: fetch order 123:
    //   connection refused"
    // → BIẾT NGAY lỗi ở đâu, trong context nào!

    if err := validatePayment(order); err != nil {
        return fmt.Errorf("process payment: validate %s: %w",
            orderID, err)
    }

    if err := chargeCard(ctx, order); err != nil {
        return fmt.Errorf("process payment: charge %s: %w",
            orderID, err)
    }
    return nil
}

// 2. NAMED RETURN + DEFER — Resource integrity!
func ReadConfig(path string) (cfg Config, err error) {
    f, err := os.Open(path)
    if err != nil {
        return cfg, fmt.Errorf("open config: %w", err)
    }
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = fmt.Errorf("close config: %w", cerr)
        }
    }()
    // → KHÔNG BAO GIỜ quên close file!
    // → Kể cả khi có error ở giữa!

    if err := json.NewDecoder(f).Decode(&cfg); err != nil {
        return cfg, fmt.Errorf("decode config: %w", err)
    }
    return cfg, nil
}

// 3. ZERO VALUE USABILITY — Integrity by default!
type Buffer struct {
    data []byte
    // Zero value = empty buffer, SẴN SÀNG dùng!
    // KHÔNG cần constructor!
}

var buf Buffer          // Zero value = usable!
buf.Write([]byte("hi")) // Works immediately!
// → Không thể "quên initialize" → Integrity!

// 4. SENTINEL ERRORS — Error identity!
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrTimeout      = errors.New("timeout")
)

func handleError(err error) {
    switch {
    case errors.Is(err, ErrNotFound):
        // 404
    case errors.Is(err, ErrUnauthorized):
        // 401
    case errors.Is(err, ErrTimeout):
        // 503
    default:
        // 500
    }
    // → CHÍNH XÁC loại error! Không catch sai!
    // → Giải quyết 8% "wrong exception" failures!
}
```

---

## §7. Áp Dụng Thực Tế Cho Go Services

### 7.1 Integrity checklist

```
╔═══════════════════════════════════════════════════════════════╗
║   INTEGRITY CHECKLIST CHO GO SERVICES                        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  MICRO INTEGRITY (Type System):                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ □ Named types cho domain concepts            │             ║
║  │   (UserID, OrderID, Money)                   │             ║
║  │ □ Struct validation methods                   │             ║
║  │ □ Enums qua const + iota                     │             ║
║  │ □ Interface contracts cho dependencies       │             ║
║  │ □ No raw string/int cho business data        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  MACRO INTEGRITY (Error Handling):                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ □ EVERY error checked (errcheck linter)      │             ║
║  │ □ Error wrapping với context                  │             ║
║  │ □ Sentinel errors cho error identity         │             ║
║  │ □ defer cho resource cleanup                  │             ║
║  │ □ context.Context cho timeout/cancel         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  WRITE LESS CODE:                                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ □ Không code mà CHƯA CẦN (YAGNI)           │             ║
║  │ □ Composition thay vì complex inheritance    │             ║
║  │ □ Table-driven tests thay vì copy-paste     │             ║
║  │ □ Standard library trước third-party        │             ║
║  │ □ Unused code = XÓA! (Go enforce!)          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CI/CD ENFORCEMENT:                                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ □ go vet (bắt suspicious constructs)        │             ║
║  │ □ golangci-lint (bắt anti-patterns)         │             ║
║  │ □ errcheck (bắt unchecked errors!)          │             ║
║  │ □ go test -race (bắt data races)            │             ║
║  │ □ go test -count=1 (no test caching)        │             ║
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
║  Q1: "Integrity trong thiết kế phần mềm là gì?"             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = Sự CHÍNH XÁC, NHẤT QUÁN của code trong    │             ║
║  │   mọi thao tác đọc, ghi, thực thi.          │             ║
║  │                                              │             ║
║  │ 2 cấp độ:                                    │             ║
║  │ • Micro: Type system → chính xác từng byte  │             ║
║  │ • Macro: Error handling → chính xác từng    │             ║
║  │   bước data transformation                   │             ║
║  │                                              │             ║
║  │ + Viết ÍT code + Error handling = MAIN code │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Tại sao error handling Go tốt hơn exceptions?"        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Exceptions: error TÁCH RỜI khỏi main code   │             ║
║  │ → "Xử lý sau, ở chỗ khác"                   │             ║
║  │ → 8% failures do catch sai exception!        │             ║
║  │ → 25% failures do ignore error!              │             ║
║  │                                              │             ║
║  │ Go errors: error LÀ main code!               │             ║
║  │ → Xử lý NGAY TẠI CHỖ                        │             ║
║  │ → errors.Is/As → check chính xác            │             ║
║  │ → errcheck linter → bắt ignored errors      │             ║
║  │                                              │             ║
║  │ "Tedious and inconvenient — but that's       │             ║
║  │  the POINT." — Kennedy                       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Viết ít code hơn giúp gì?"                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Thống kê: 15–50 bugs / 1000 dòng code       │             ║
║  │ → Ít dòng = ÍT BUGS (tuyến tính!)          │             ║
║  │                                              │             ║
║  │ Stroustrup: code thừa = Ugly (bugs ẩn),     │             ║
║  │ Large (tests thiếu), Slow (shortcuts)        │             ║
║  │                                              │             ║
║  │ Go enforce: unused import/var = ERROR!        │             ║
║  │ YAGNI: đừng code thứ chưa cần!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Developer matrix — khi nào acceptable?"                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Learning/Prototyping: OK, cần mentor         │             ║
║  │ Hacking/Guessing: CẦN FIX, code review      │             ║
║  │ Education: CẦN ĐÀO TẠO NGAY               │             ║
║  │ Dangerous: KHÔNG CHẤP NHẬN! Biết sai mà    │             ║
║  │ vẫn cố ý deploy → vi phạm integrity!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: DESIGN PHILOSOPHY ON INTEGRITY                │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Integrity = ƯU TIÊN SỐ 1!            │            │
    │  │    → Chính xác, nhất quán, hiệu quả     │            │
    │  │    → Trong MỌI thao tác đọc/ghi/thực thi│            │
    │  │                                          │            │
    │  │ 2. Error handling IS the main code!       │            │
    │  │    → 92% critical failures từ đây        │            │
    │  │    → Go if err != nil = đúng triết lý   │            │
    │  │    → "Tedious" = BY DESIGN!              │            │
    │  │                                          │            │
    │  │ 3. Write LESS code!                       │            │
    │  │    → 15-50 bugs / 1000 LOC               │            │
    │  │    → Ít code = ít bugs = ít bảo trì     │            │
    │  │    → Go unused = compile error!           │            │
    │  │                                          │            │
    │  │ 4. Type system = Micro Integrity!         │            │
    │  │    → Named types cho domain              │            │
    │  │    → Compile-time guarantees              │            │
    │  │                                          │            │
    │  │ 5. ELIMINATE Carelessness!                 │            │
    │  │    → Linters, CI/CD, code review          │            │
    │  │    → "Biết sai mà vẫn làm" = DANGEROUS  │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "If we are to become SERIOUS about                     │
    │   RELIABILITY, we must become serious                   │
    │   about INTEGRITY and therefore                         │
    │   ERROR HANDLING."                                      │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
