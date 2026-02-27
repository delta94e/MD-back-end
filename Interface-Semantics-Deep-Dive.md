# Interface Semantics — Ngữ Nghĩa Interface Trong Go

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Interface Semantics" của William Kennedy (2017)
> **Góc nhìn:** Interface + Value/Pointer Semantics, Method Sets, Integrity
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                  | Mô tả                                           |
| --- | --------------------------------------- | ----------------------------------------------- |
| §1  | Interface lưu trữ Value vs Pointer      | Hai cách lưu data trong interface               |
| §2  | Ví dụ minh họa: printer                 | Value copy vs Address share — kết quả khác nhau |
| §3  | Method Sets — Quy tắc Integrity         | Luật chọn value/pointer receiver                |
| §4  | Tại sao Pointer Semantics cấm lưu Value | Hai lý do về integrity                          |
| §5  | Interfaces Are Valueless                | Interface "vô giá trị" — luôn về data bên trong |
| §6  | So sánh Interface Values                | Data bên trong quyết định bằng nhau             |
| §7  | Tổng hợp Rules & Best Practices         | Bảng quy tắc chọn semantic                      |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior     | Ôn tập & thực hành                              |

---

## §1. Interface Lưu Trữ Value vs Pointer

### 1.1 Hai cách lưu data

```
╔═══════════════════════════════════════════════════════════════╗
║   INTERFACE — HAI CÁCH LƯU DATA                              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Interface trong Go có thể lưu data bằng 2 cách:            ║
║                                                               ║
║  1. VALUE SEMANTICS — lưu BẢN SAO!                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ var p printer = u   // u = user value        │             ║
║  │                                              │             ║
║  │ Interface Value:                              │             ║
║  │ ┌───────────────┐                            │             ║
║  │ │ [type info]   │→ *user type metadata       │             ║
║  │ │ [data ptr]    │→ COPY of u (bản sao!)     │             ║
║  │ └───────────────┘                            │             ║
║  │                                              │             ║
║  │ → Interface chứa BẢN SAO riêng!             │             ║
║  │ → Thay đổi u gốc = KHÔNG ảnh hưởng!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  2. POINTER SEMANTICS — lưu ĐỊA CHỈ!                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ var p printer = &u  // &u = address of u     │             ║
║  │                                              │             ║
║  │ Interface Value:                              │             ║
║  │ ┌───────────────┐                            │             ║
║  │ │ [type info]   │→ *user type metadata       │             ║
║  │ │ [data ptr]    │→ ADDRESS of u gốc!        │             ║
║  │ └───────────────┘                            │             ║
║  │                                              │             ║
║  │ → Interface CHIA SẺ value gốc!              │             ║
║  │ → Thay đổi u gốc = ẢNH HƯỞNG!             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Ví Dụ Minh Họa: printer

### 2.1 Code ví dụ

```go
// ═══ VALUE vs POINTER trong Interface ═══

type printer interface {
    print()
}

type user struct {
    name string
}

// Value receiver → implement bằng VALUE SEMANTICS!
func (u user) print() {
    fmt.Println("User Name:", u.name)
}

func main() {
    u := user{"Bill"}

    entities := []printer{
        u,    // Index 0: VALUE semantics (COPY!)
        &u,   // Index 1: POINTER semantics (SHARE!)
    }

    // Thay đổi u sau khi đã lưu vào interface!
    u.name = "Bill_CHG"

    for _, e := range entities {
        e.print()
    }
}
```

### 2.2 Kết quả và giải thích

```
╔═══════════════════════════════════════════════════════════════╗
║   KẾT QUẢ — VALUE vs POINTER TRONG INTERFACE                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  OUTPUT:                                                       ║
║  User Name: Bill       ← Index 0 (VALUE — bản sao!)         ║
║  User Name: Bill_CHG   ← Index 1 (POINTER — thấy thay đổi!)║
║                                                               ║
║  SƠ ĐỒ MEMORY:                                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  u := user{"Bill"}   →  u.name = "Bill_CHG" │             ║
║  │  ┌──────────────┐       ┌──────────────┐     │             ║
║  │  │ user{"Bill"} │  →→→  │ user         │     │             ║
║  │  └──────────────┘       │ {"Bill_CHG"} │     │             ║
║  │                         └──────────────┘     │             ║
║  │                              ▲                │             ║
║  │                              │ POINTER        │             ║
║  │  entities[]:                 │                │             ║
║  │  ┌────────────────────┐     │                │             ║
║  │  │ [0] = COPY of u   │     │                │             ║
║  │  │ ┌────────────────┐ │     │                │             ║
║  │  │ │ user{"Bill"}   │ │     │ BẢN SAO riêng!│             ║
║  │  │ │ KHÔNG ĐỔI!    │ │     │ KHÔNG bị ảnh  │             ║
║  │  │ └────────────────┘ │     │ hưởng!         │             ║
║  │  ├────────────────────┤     │                │             ║
║  │  │ [1] = ADDRESS of u│─────┘                │             ║
║  │  │ ┌────────────────┐ │                      │             ║
║  │  │ │ ptr → u gốc   │ │     TRỎ TỚI u gốc! │             ║
║  │  │ │ THẤY "Bill_CHG"│ │     THẤY thay đổi! │             ║
║  │  │ └────────────────┘ │                      │             ║
║  │  └────────────────────┘                      │             ║
║  │                                              │             ║
║  │  → Index 0: lưu COPY → mãi mãi "Bill"!    │             ║
║  │  → Index 1: lưu ADDRESS → thấy "Bill_CHG"! │             ║
║  │                                              │             ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Method Sets — Quy Tắc Integrity

### 3.1 Bảng Method Set Rules

```
╔═══════════════════════════════════════════════════════════════╗
║   METHOD SET RULES — QUY TẮC INTEGRITY                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Quy tắc Method Set quyết định: DATA nào có thể             ║
║  LƯU VÀO interface?                                          ║
║                                                               ║
║  ┌───────────────────┬──────────────┬──────────────┐         ║
║  │ IMPLEMENT BẰNG    │ LƯU VALUE?  │ LƯU POINTER? │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ VALUE Receiver    │   ✅ CÓ     │  ⚠️ CÓ       │         ║
║  │ func (u user)     │   (an toàn) │  (cẩn thận!) │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ POINTER Receiver  │   ❌ KHÔNG! │   ✅ CÓ      │         ║
║  │ func (u *user)    │   (CẤM!)    │   (an toàn)  │         ║
║  └───────────────────┴──────────────┴──────────────┘         ║
║                                                               ║
║  GIẢI THÍCH TỪNG Ô:                                          ║
║                                                               ║
║  VALUE RECEIVER + Lưu VALUE ✅:                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ func (u user) print() { ... }                 │             ║
║  │ var p printer = u  // ✅ LƯU COPY!           │             ║
║  │                                              │             ║
║  │ → Implement bằng value = COPY an toàn!       │             ║
║  │ → Interface có bản sao riêng!                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VALUE RECEIVER + Lưu POINTER ⚠️:                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ func (u user) print() { ... }                 │             ║
║  │ var p printer = &u // ⚠️ LƯU ADDRESS!        │             ║
║  │                                              │             ║
║  │ → Cho phép nhưng CẨN THẬN!                  │             ║
║  │ → Đổi từ value → pointer = mixing semantics!│             ║
║  │ → Phải là conscious exception!               │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  POINTER RECEIVER + Lưu POINTER ✅:                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ func (u *user) print() { ... }                │             ║
║  │ var p printer = &u // ✅ LƯU ADDRESS!         │             ║
║  │                                              │             ║
║  │ → Implement bằng pointer = SHARE!             │             ║
║  │ → Lưu address = nhất quán!                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  POINTER RECEIVER + Lưu VALUE ❌:                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ func (u *user) print() { ... }                │             ║
║  │ var p printer = u  // ❌ COMPILER ERROR!      │             ║
║  │                                              │             ║
║  │ → KHÔNG ĐƯỢC! CẤM!                          │             ║
║  │ → Implement bằng pointer = CẦN share!        │             ║
║  │ → Lưu value (copy) = phá vỡ integrity!      │             ║
║  │ → Compiler BẢO VỆ bạn!                      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Tại Sao Pointer Semantics CẤM Lưu Value

### 4.1 Hai lý do về integrity

```
╔═══════════════════════════════════════════════════════════════╗
║   TẠI SAO POINTER RECEIVER + LƯU VALUE = CẤM?               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  LÝ DO #1: KHÔNG PHẢI MỌI VALUE ĐỀU CÓ ADDRESS!            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ type duration int                             │             ║
║  │                                              │             ║
║  │ func (d *duration) notify() {                 │             ║
║  │     fmt.Println("Notification in", *d)        │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ func main() {                                 │             ║
║  │     duration(42).notify() // ❌ COMPILER ERROR│             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ Error:                                        │             ║
║  │ "cannot call pointer method on duration(42)"  │             ║
║  │ "cannot take the address of duration(42)"     │             ║
║  │                                              │             ║
║  │ → duration(42) = LITERAL VALUE!               │             ║
║  │ → Literal values = CONSTANTS!                 │             ║
║  │ → Constants chỉ tồn tại lúc COMPILE TIME!   │             ║
║  │ → KHÔNG CÓ ADDRESS trong runtime!            │             ║
║  │ → Không thể share → không gọi *receiver!    │             ║
║  │                                              │             ║
║  │ CÁC GIÁ TRỊ KHÔNG CÓ ADDRESS:               │             ║
║  │ • Literal values: 42, "hello", true           │             ║
║  │ • Type conversions: duration(42)              │             ║
║  │ • Function return values (trực tiếp):        │             ║
║  │   getUser().notify() // có thể không address │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  LÝ DO #2: BẢO VỆ INTEGRITY CỦA SEMANTICS!                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ Nếu implement bằng POINTER receiver:          │             ║
║  │ → Ý nghĩa: "Data này PHẢI được SHARE!"     │             ║
║  │ → "KHÔNG AN TOÀN để tạo bản sao!"          │             ║
║  │                                              │             ║
║  │ Nếu cho phép lưu VALUE vào interface:        │             ║
║  │ → Interface có BẢN SAO riêng!                │             ║
║  │ → Pointer method ghi vào bản sao = VÔ ÍCH! │             ║
║  │ → Vi phạm ý đồ "share only"!               │             ║
║  │                                              │             ║
║  │ Kennedy:                                      │             ║
║  │ "You can NEVER assume that it is safe to     │             ║
║  │  make a COPY of any value that is pointed    │             ║
║  │  to by a pointer."                           │             ║
║  │                                              │             ║
║  │ → Pointer semantics = "ĐỪNG copy value này!"│             ║
║  │ → Method set rules BẢO VỆ nguyên tắc này!  │             ║
║  │ → Compiler là GUARDIAN của integrity!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TẠI SAO VALUE RECEIVER CHO PHÉP LƯU POINTER?               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Value semantics → "Copy an toàn!"            │             ║
║  │ → Nếu copy an toàn → share cũng an toàn!   │             ║
║  │ → Đổi từ value → pointer = CHẤP NHẬN được  │             ║
║  │   (nhưng phải là conscious exception!)        │             ║
║  │                                              │             ║
║  │ ⚠️ Mixing semantics = consistency issue!      │             ║
║  │ → Cho phép nhưng phải CẨN THẬN!             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Interfaces Are Valueless

### 5.1 Interface = "vô giá trị"

```
╔═══════════════════════════════════════════════════════════════╗
║   INTERFACES ARE VALUELESS — INTERFACE "VÔ GIÁ TRỊ"         ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  INTERFACE KHÔNG CÓ GIÁ TRỊ TỰ THÂN!                       ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ var n notifier       // n = nil interface     │             ║
║  │                      // n = VALUELESS!        │             ║
║  │                      // Không có gì cụ thể! │             ║
║  │                                              │             ║
║  │ n = duration(42)     // BÂY GIỜ n có data!  │             ║
║  │                      // n = CONCRETE!         │             ║
║  │                                              │             ║
║  │ → Interface chỉ CÓ Ý NGHĨA khi chứa data! │             ║
║  │ → Data bên trong = thứ DUY NHẤT quan trọng! │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CẤU TRÚC NỘI BỘ INTERFACE:                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ Interface value (2 words):                    │             ║
║  │ ┌─────────────────────────────┐              │             ║
║  │ │ Word 1: TYPE POINTER        │              │             ║
║  │ │ → trỏ tới type metadata   │              │             ║
║  │ │                             │              │             ║
║  │ │ Word 2: DATA POINTER        │              │             ║
║  │ │ → trỏ tới concrete value  │              │             ║
║  │ └─────────────────────────────┘              │             ║
║  │                                              │             ║
║  │ QUAN TRỌNG:                                   │             ║
║  │ → Word 2 LUÔN là pointer (từ Go 1.4+)!      │             ║
║  │ → Dù vậy, ĐỪNG dùng implementation detail   │             ║
║  │   để lý luận về method set rules!            │             ║
║  │ → Implementation có thể THAY ĐỔI!           │             ║
║  │ → Rules dựa trên SEMANTICS, không phải       │             ║
║  │   implementation!                             │             ║
║  │                                              │             ║
║  │ Lịch sử:                                     │             ║
║  │ Go 1.3: word 2 = pointer HOẶC scalar value  │             ║
║  │ Go 1.4: word 2 = LUÔN pointer               │             ║
║  │ → Thay đổi vì GC cần nhất quán!            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. So Sánh Interface Values

### 6.1 Data bên trong quyết định

```go
// ═══ SO SÁNH INTERFACE — DATA BÊN TRONG! ═══

type errorString struct {
    s string
}

func (e errorString) Error() string {
    return e.s
}

// Factory function dùng VALUE SEMANTICS!
func New(text string) error {
    return errorString{text}  // Return COPY of value!
}

var ErrBadRequest = New("Bad Request")

func main() {
    err := webCall()

    // So sánh 2 interface values!
    if err == ErrBadRequest {
        fmt.Println("Interface Values MATCH")  // ← IN RA!
    }
}

func webCall() error {
    return New("Bad Request")  // Interface MỚI, cùng data!
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   SO SÁNH INTERFACE — DATA QUYẾT ĐỊNH!                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  OUTPUT: "Interface Values MATCH" ← BẰNG NHAU!               ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ErrBadRequest (error interface):              │             ║
║  │ ┌───────────────────────┐                    │             ║
║  │ │ [type: errorString]   │                    │             ║
║  │ │ [data: {"Bad Request"}]│                   │             ║
║  │ └───────────────────────┘                    │             ║
║  │                                              │             ║
║  │ err (error interface):                        │             ║
║  │ ┌───────────────────────┐                    │             ║
║  │ │ [type: errorString]   │                    │             ║
║  │ │ [data: {"Bad Request"}]│                   │             ║
║  │ └───────────────────────┘                    │             ║
║  │                                              │             ║
║  │ → KHÁC interface value (2 biến riêng biệt!) │             ║
║  │ → NHƯNG: cùng TYPE + cùng DATA              │             ║
║  │ → err == ErrBadRequest = TRUE!               │             ║
║  │                                              │             ║
║  │ QUY TẮC SO SÁNH:                              │             ║
║  │ → So sánh interface = so sánh DATA BÊN TRONG│             ║
║  │ → KHÔNG phải so sánh interface bản thân!    │             ║
║  │ → Interface = "valueless"!                    │             ║
║  │                                              │             ║
║  │ VALUE semantics: so sánh VALUES              │             ║
║  │ POINTER semantics: so sánh ADDRESSES         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ⚠️ CHÚ Ý: Nếu dùng POINTER semantics:                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ func New(text string) error {                 │             ║
║  │     return &errorString{text}  // POINTER!   │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ → 2 lần gọi New("Bad Request")              │             ║
║  │ → 2 addresses KHÁC NHAU!                    │             ║
║  │ → err != ErrBadRequest! (FALSE!)             │             ║
║  │                                              │             ║
║  │ → Đây là cách errors package thật hoạt động!│             ║
║  │ → errors.New() return *errorString (pointer!)│             ║
║  │ → So sánh address = xác định identity!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Tổng Hợp Rules & Best Practices

### 7.1 Bảng quyết định

```
╔═══════════════════════════════════════════════════════════════╗
║   TỔNG HỢP RULES — BẢN ĐỒ QUYẾT ĐỊNH                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  BẢNG METHOD SET & INTERFACE:                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │           │ Values T    │ Pointers *T        │             ║
║  │ ──────────┼─────────────┼────────────────── │             ║
║  │ Method    │ (T) methods │ (T) AND (*T)       │             ║
║  │ Set of    │ ONLY        │ methods            │             ║
║  │           │             │                    │             ║
║  │ Can store │ YES: T vào  │ YES: *T vào       │             ║
║  │ in iface  │ iface(T)    │ iface(T)           │             ║
║  │ impl by T │             │                    │             ║
║  │           │             │                    │             ║
║  │ Can store │ NO!         │ YES: *T vào       │             ║
║  │ in iface  │ ❌ CẤM!    │ iface(*T)          │             ║
║  │ impl by *T│             │                    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  FLOW QUYẾT ĐỊNH:                                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Type có NÊN copy?                            │             ║
║  │       │                                      │             ║
║  │    ┌──┴──┐                                   │             ║
║  │   CÓ    KHÔNG                                │             ║
║  │    │       │                                 │             ║
║  │    ▼       ▼                                 │             ║
║  │  VALUE   POINTER                             │             ║
║  │  RECEIVER RECEIVER                           │             ║
║  │    │       │                                 │             ║
║  │    ▼       ▼                                 │             ║
║  │  Lưu vào  Lưu vào                           │             ║
║  │  iface:   iface:                             │             ║
║  │  T ✅     T ❌ CẤM!                         │             ║
║  │  *T ⚠️    *T ✅                              │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BEST PRACTICES:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. CHỌN 1 semantic cho mỗi type!            │             ║
║  │ 2. DÙNG NHẤT QUÁN trong toàn bộ code!      │             ║
║  │ 3. Mixing semantics = CHỈ khi conscious      │             ║
║  │    exception!                                 │             ║
║  │ 4. Code review: TÌM inconsistency!           │             ║
║  │ 5. Compiler BẢO VỆ bạn khỏi pointer→value!│             ║
║  │    (nhưng KHÔNG bảo vệ value→pointer!)       │             ║
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
║  Q1: "Method Set Rules là gì? Tại sao quan trọng?"          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = Quy tắc xác định DATA nào lưu được vào   │             ║
║  │ interface dựa trên cách implement!            │             ║
║  │                                              │             ║
║  │ Value receiver → lưu value ✅ và pointer ⚠️ │             ║
║  │ Pointer receiver → CHỈ lưu pointer ✅       │             ║
║  │                                              │             ║
║  │ Quan trọng vì BẢO VỆ INTEGRITY:              │             ║
║  │ → Pointer semantics = "đừng copy!"          │             ║
║  │ → Compiler ngăn bạn copy vào interface!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Tại sao pointer receiver KHÔNG CHO lưu value          ║
║       vào interface?"                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 2 lý do:                                     │             ║
║  │ 1. Không phải mọi value đều có ADDRESS!      │             ║
║  │    → Literal values (42, "hello") = no addr! │             ║
║  │    → Pointer receiver CẦN address!           │             ║
║  │                                              │             ║
║  │ 2. Bảo vệ semantic integrity!                 │             ║
║  │    → Pointer = "share, đừng copy!"          │             ║
║  │    → Lưu value = tạo copy = VI PHẠM!        │             ║
║  │    → "Never assume safe to copy pointed value"│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Interface value so sánh thế nào?"                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → So sánh DATA BÊN TRONG, không phải        │             ║
║  │   interface bản thân!                         │             ║
║  │ → Interface = "valueless"!                    │             ║
║  │                                              │             ║
║  │ Value semantics → so sánh VALUES             │             ║
║  │ → errorString{"Bad"} == errorString{"Bad"} ✅│             ║
║  │                                              │             ║
║  │ Pointer semantics → so sánh ADDRESSES        │             ║
║  │ → &e1 != &e2 (dù cùng data!) ❌             │             ║
║  │ → Đây là cách errors.New() hoạt động!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Value receiver implement interface — lưu pointer      ║
║       có được không?"                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ CÓ! Nhưng phải CẨN THẬN!                    │             ║
║  │                                              │             ║
║  │ → Value → pointer = mixing semantics!        │             ║
║  │ → Go CHO PHÉP (không error!)                 │             ║
║  │ → Nhưng phải là CONSCIOUS EXCEPTION!         │             ║
║  │ → Consistency là mục tiêu HÀNG ĐẦU!        │             ║
║  │ → Unexpected side effects nếu không cẩn thận│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: INTERFACE SEMANTICS                          │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Interface lưu VALUE (copy) hoặc        │            │
    │  │    POINTER (address) — tùy cách bạn lưu! │            │
    │  │                                          │            │
    │  │ 2. Method Set Rules BẢO VỆ integrity:     │            │
    │  │    → Pointer receiver → CHỈ pointer ✅   │            │
    │  │    → Value receiver → cả hai ✅⚠️        │            │
    │  │                                          │            │
    │  │ 3. "Interfaces Are Valueless"             │            │
    │  │    → Chỉ DATA bên trong là quan trọng!   │            │
    │  │    → So sánh = so sánh DATA!              │            │
    │  │                                          │            │
    │  │ 4. Mixing semantics = CONSCIOUS EXCEPTION │            │
    │  │    → Nhất quán là mục tiêu!               │            │
    │  │    → Code review: tìm inconsistency!      │            │
    │  │                                          │            │
    │  │ 5. Compiler = GUARDIAN of integrity!       │            │
    │  │    → Ngăn pointer→value (dangerous!)      │            │
    │  │    → Cho phép value→pointer (cẩn thận!)  │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "Look at the SEMANTICS you are using for any           │
    │   given type. During CODE REVIEWS, look for             │
    │   CONSISTENCY in semantic and QUESTION code              │
    │   that violates it."                                    │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
