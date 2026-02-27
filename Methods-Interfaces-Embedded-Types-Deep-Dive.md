# Methods, Interfaces and Embedded Types — Phương Thức, Interface và Kiểu Nhúng

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Methods, Interfaces and Embedded Types in Go" của William Kennedy (2014)
> **Góc nhìn:** Method Sets, Interface Compliance, Embedded Type Promotion, Inner vs Outer
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                    | Mô tả                                         |
| --- | ----------------------------------------- | --------------------------------------------- |
| §1  | Methods trong Go                          | Function vs Method, Value vs Pointer receiver |
| §2  | Interfaces trong Go                       | Implicit implementation, naming convention    |
| §3  | Method Set Rules — Interface Compliance   | Quy tắc \*T vs T, tại sao value lỗi?          |
| §4  | Embedded Types — Kiểu nhúng               | Composition, inner/outer type, promotion      |
| §5  | Method Promotion Rules                    | 4 quy tắc promote method từ inner type        |
| §6  | Câu hỏi lớn: Cả Outer VÀ Inner implement? | Outer override inner! Không conflict!         |
| §7  | Tổng kết & Câu hỏi phỏng vấn Senior       | Ôn tập & thực hành                            |

---

## §1. Methods Trong Go

### 1.1 Function vs Method

```go
// ═══ FUNCTION vs METHOD ═══

// FUNCTION — không gắn với type nào
func PrintName(name string) {
    fmt.Println(name)
}

// METHOD — gắn với type qua RECEIVER!
type User struct {
    Name  string
    Email string
}

func (u User) Notify() error {   // ← RECEIVER = User
    // u là một COPY của User
    return nil
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   METHOD = FUNCTION + RECEIVER                                ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ func (u User) Notify() error                  │             ║
║  │  ↑     ↑  ↑     ↑       ↑                   │             ║
║  │  │     │  │     │       └── return type      │             ║
║  │  │     │  │     └── method name              │             ║
║  │  │     │  └── receiver TYPE                  │             ║
║  │  │     └── receiver variable                 │             ║
║  │  └── keyword func                            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VALUE RECEIVER vs POINTER RECEIVER:                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  func (u User) Notify()   ← VALUE receiver  │             ║
║  │  → Method nhận COPY!                        │             ║
║  │  → Thay đổi u KHÔNG ảnh hưởng gốc!        │             ║
║  │                                              │             ║
║  │  func (u *User) Notify()  ← POINTER receiver│             ║
║  │  → Method nhận ADDRESS!                     │             ║
║  │  → Thay đổi u ẢNH HƯỞNG gốc!              │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GỌI METHOD:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ // Value receiver → cả value & pointer gọi được│           ║
║  │ bill := User{"Bill", "bill@email.com"}        │             ║
║  │ bill.Notify()    // ✅ value gọi value recv  │             ║
║  │                                              │             ║
║  │ jill := &User{"Jill", "jill@email.com"}      │             ║
║  │ jill.Notify()    // ✅ pointer gọi value recv│             ║
║  │ // → Go tự dereference: (*jill).Notify()    │             ║
║  │                                              │             ║
║  │ // Pointer receiver → cả value & pointer gọi được│        ║
║  │ bill.Notify()    // ✅ Go tự: (&bill).Notify()│            ║
║  │ jill.Notify()    // ✅ pointer gọi ptr recv  │             ║
║  │                                              │             ║
║  │ ⚠️ NHƯNG method set rules KHÁC KHI dùng     │             ║
║  │    interface! (xem §3)                        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Interfaces Trong Go

```go
// ═══ INTERFACE — HÀNH VI (BEHAVIOR) ═══

// Naming convention: -er suffix khi 1 method!
type Notifier interface {
    Notify() error
}

// Polymorphic function — nhận bất kỳ Notifier nào!
func SendNotification(notify Notifier) error {
    return notify.Notify()
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   GO INTERFACE = IMPLICIT IMPLEMENTATION!                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Java: class User IMPLEMENTS Notifier { }     │             ║
║  │  → PHẢI khai báo "implements"!               │             ║
║  │                                              │             ║
║  │  Go: type User struct { ... }                 │             ║
║  │      func (u *User) Notify() error { ... }    │             ║
║  │  → KHÔNG CẦN khai báo gì thêm!             │             ║
║  │  → Implement ĐỦ methods = THỎAmãn MÃN!     │             ║
║  │  → IMPLICIT! Compiler tự kiểm tra!          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NAMING CONVENTION:                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1 method → đặt tên interface = verb + "er"  │             ║
║  │                                              │             ║
║  │ Notify()    → Notifier                        │             ║
║  │ Read()      → Reader                          │             ║
║  │ Write()     → Writer                          │             ║
║  │ String()    → Stringer                        │             ║
║  │ Close()     → Closer                          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Method Set Rules — Interface Compliance

### 3.1 Ví dụ lỗi kinh điển

```go
// User implement Notifier bằng POINTER receiver
func (u *User) Notify() error {
    log.Printf("Sending Email To %s<%s>\n", u.Name, u.Email)
    return nil
}

func main() {
    user := User{Name: "janet", Email: "janet@email.com"}

    SendNotification(user)   // ❌ COMPILER ERROR!
}

// Error: cannot use user (type User) as type Notifier
//        User does not implement Notifier
//        (Notify method has pointer receiver)
```

```
╔═══════════════════════════════════════════════════════════════╗
║   METHOD SET RULES — QUY TẮC QUAN TRỌNG NHẤT!                ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  2 QUY TẮC TỪ GO SPEC:                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  QUY TẮC 1: Method set của *T                │             ║
║  │  = TẤT CẢ methods với receiver *T HOẶC T    │             ║
║  │                                              │             ║
║  │  → Pointer chứa CẢ HAI loại methods!       │             ║
║  │  → *User → có cả (u User) VÀ (u *User)     │             ║
║  │                                              │             ║
║  │  QUY TẮC 2: Method set của T                 │             ║
║  │  = CHỈ methods với receiver T                │             ║
║  │                                              │             ║
║  │  → Value CHỈ có value receiver methods!      │             ║
║  │  → User → CHỈ có (u User), KHÔNG có (u *User)│            ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BẢNG METHOD SET:                                              ║
║  ┌────────────────┬──────────────────────────────┐             ║
║  │ Giá trị       │ Method Set                    │             ║
║  ├────────────────┼──────────────────────────────┤             ║
║  │ T (value)     │ (t T) methods CHỈ!           │             ║
║  │ *T (pointer)  │ (t T) + (t *T) TẤT CẢ!     │             ║
║  └────────────────┴──────────────────────────────┘             ║
║                                                               ║
║  ÁP DỤNG VÀO VÍ DỤ:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ func (u *User) Notify() error                │             ║
║  │                 ↑ POINTER receiver!           │             ║
║  │                                              │             ║
║  │ SendNotification(user)                        │             ║
║  │                  ↑ VALUE! (User, không *User)│             ║
║  │                                              │             ║
║  │ → Method set của VALUE User:                 │             ║
║  │   CHỈ có methods với receiver (u User)       │             ║
║  │ → Notify() có receiver (u *User)             │             ║
║  │ → KHÔNG NẰM TRONG method set!               │             ║
║  │ → User KHÔNG implement Notifier!             │             ║
║  │ → ❌ COMPILER ERROR!                          │             ║
║  │                                              │             ║
║  │ FIX: Dùng POINTER!                           │             ║
║  │ SendNotification(&user) // ← address!       │             ║
║  │ → Method set của *User có cả (u *User)      │             ║
║  │ → ✅ WORKS!                                   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TẠI SAO CÓ QUY TẮC NÀY?                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Pointer receiver = muốn SHARE + MUTATE!   │             ║
║  │ → Truyền value vào interface = COPY value!   │             ║
║  │ → Copy → method thay đổi COPY, không gốc!  │             ║
║  │ → Vi phạm semantic integrity!                │             ║
║  │ → Go BẢO VỆ bạn bằng compiler error!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Embedded Types — Kiểu Nhúng

```go
// ═══ EMBEDDED TYPE — COMPOSITION! ═══

type Admin struct {
    User           // ← EMBEDDED (inner type)!
    Level string
}

func main() {
    admin := &Admin{
        User: User{
            Name:  "john smith",
            Email: "john@email.com",
        },
        Level: "super",
    }

    // User.Notify() PROMOTED lên Admin!
    SendNotification(admin) // ✅ WORKS!
}
// Output: User: Sending User Email To john smith<john@email.com>
```

```
╔═══════════════════════════════════════════════════════════════╗
║   EMBEDDED TYPES = COMPOSITION, NOT INHERITANCE!              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  INNER vs OUTER TYPE:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Admin (OUTER type):                          │             ║
║  │  ┌──────────────────────────────────┐        │             ║
║  │  │                                  │        │             ║
║  │  │  User (INNER type):              │        │             ║
║  │  │  ┌──────────────────────────┐    │        │             ║
║  │  │  │ Name:  "john smith"      │    │        │             ║
║  │  │  │ Email: "john@email.com"  │    │        │             ║
║  │  │  │ Notify() ← *User method │    │        │             ║
║  │  │  └──────────────────────────┘    │        │             ║
║  │  │                                  │        │             ║
║  │  │  Level: "super"                  │        │             ║
║  │  └──────────────────────────────────┘        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  PROMOTION:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  admin.User.Notify()  → Gọi qua INNER type  │             ║
║  │  admin.Notify()       → PROMOTED! Gọi trực tiếp│          ║
║  │  admin.Name           → PROMOTED! Truy cập trực tiếp│     ║
║  │                                              │             ║
║  │  → Nhờ promotion, Admin IMPLEMENT Notifier!  │             ║
║  │  → SendNotification(admin) ← ✅              │             ║
║  │                                              │             ║
║  │  Effective Go:                                │             ║
║  │  "When embedded methods are invoked, the     │             ║
║  │   RECEIVER is the INNER type, not outer."    │             ║
║  │  → admin.Notify() → receiver = User bên trong│             ║
║  │  → KHÔNG phải Admin!                         │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Method Promotion Rules

```
╔═══════════════════════════════════════════════════════════════╗
║   4 QUY TẮC PROMOTE METHOD TỪ INNER TYPE                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Cho struct S chứa anonymous field:                           ║
║                                                               ║
║  ✅ QUY TẮC 1: "S contains T" (embed value)                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ type S struct { T }                           │             ║
║  │                                              │             ║
║  │ → Method set của S và *S                     │             ║
║  │   BAO GỒM methods với receiver T             │             ║
║  │                                              │             ║
║  │ VD: func (t T) Method()                       │             ║
║  │ → s.Method()   ✅ (value S gọi được!)       │             ║
║  │ → (&s).Method() ✅ (pointer *S cũng được!)  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ✅ QUY TẮC 2: Thêm cho *S                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ type S struct { T }                           │             ║
║  │                                              │             ║
║  │ → Method set của *S THÊM:                    │             ║
║  │   methods với receiver *T                    │             ║
║  │                                              │             ║
║  │ VD: func (t *T) Method()                      │             ║
║  │ → (&s).Method() ✅ (pointer *S gọi được!)   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ✅ QUY TẮC 3: "S contains *T" (embed pointer)              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ type S struct { *T }                          │             ║
║  │                                              │             ║
║  │ → Method set của S và *S                     │             ║
║  │   BAO GỒM methods với receiver T HOẶC *T    │             ║
║  │                                              │             ║
║  │ → Embed pointer = promote TẤT CẢ methods!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ❌ QUY TẮC 4 (suy ra): "S contains T, value S"             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ type S struct { T }                           │             ║
║  │                                              │             ║
║  │ → Method set của S (value, KHÔNG phải *S):   │             ║
║  │   KHÔNG BAO GỒM methods với receiver *T     │             ║
║  │                                              │             ║
║  │ VD: func (t *T) Method()                      │             ║
║  │ → s.Method()   ❌ VALUE S KHÔNG GỌI ĐƯỢC!   │             ║
║  │ → (&s).Method() ✅ Pointer *S gọi được!     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BẢNG TỔNG HỢP:                                               ║
║  ┌──────────────┬──────────┬────────────┬──────────────┐      ║
║  │ Embed        │ Caller   │ (t T) recv │ (t *T) recv  │      ║
║  ├──────────────┼──────────┼────────────┼──────────────┤      ║
║  │ S { T }      │ s  (val) │ ✅         │ ❌           │      ║
║  │ S { T }      │ &s (ptr) │ ✅         │ ✅           │      ║
║  │ S { *T }     │ s  (val) │ ✅         │ ✅           │      ║
║  │ S { *T }     │ &s (ptr) │ ✅         │ ✅           │      ║
║  └──────────────┴──────────┴────────────┴──────────────┘      ║
║                                                               ║
║  → CHỈ 1 ô ❌: embed VALUE T, gọi bằng VALUE S,             ║
║    với pointer receiver *T method!                            ║
║  → NHẤT QUÁN với method set rules ở §3!                     ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Câu Hỏi Lớn: Cả Outer VÀ Inner Implement?

### 6.1 Ví dụ: Admin CŨNG implement Notifier

```go
// Inner type User implement Notifier
func (u *User) Notify() error {
    log.Printf("User: Sending User Email To %s<%s>\n", u.Name, u.Email)
    return nil
}

// Outer type Admin CŨNG implement Notifier!
func (a *Admin) Notify() error {
    log.Printf("Admin: Sending Admin Email To %s<%s>\n", a.Name, a.Email)
    return nil
}

func main() {
    admin := &Admin{
        User:  User{Name: "john smith", Email: "john@email.com"},
        Level: "super",
    }

    SendNotification(admin)  // ← GỌI CÁI NÀO?
    admin.Notify()           // ← GỌI CÁI NÀO?
}
// Output: Admin: Sending Admin Email To john smith<john@email.com>
// → OUTER type's implementation WINS!
```

```
╔═══════════════════════════════════════════════════════════════╗
║   CẢ OUTER VÀ INNER IMPLEMENT — AI THẮNG?                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CÂU HỎI 1: Compiler có báo lỗi không?                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → KHÔNG! Không có conflict!                   │             ║
║  │                                              │             ║
║  │ Vì sao? Embedded type = FIELD!               │             ║
║  │ → User.Notify() = inner type method          │             ║
║  │ → Admin.Notify() = outer type method         │             ║
║  │ → Hai method KHÁC NHAU, unique!              │             ║
║  │ → user là FIELD NAME bên trong Admin!       │             ║
║  │ → admin.User.Notify() vẫn gọi được!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CÂU HỎI 2: Gọi cái nào cho interface call?                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → OUTER type's implementation!                │             ║
║  │                                              │             ║
║  │ SendNotification(admin)                       │             ║
║  │ → Gọi Admin.Notify()! ← OUTER wins!         │             ║
║  │                                              │             ║
║  │ admin.Notify()                                │             ║
║  │ → Gọi Admin.Notify()! ← OUTER wins!         │             ║
║  │                                              │             ║
║  │ admin.User.Notify()                           │             ║
║  │ → Gọi User.Notify()! ← vẫn accessible!     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  QUY TẮC:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ 1. Outer type CÓ implement → dùng OUTER!    │             ║
║  │ 2. Outer type KHÔNG implement → PROMOTE      │             ║
║  │    từ inner type!                             │             ║
║  │ 3. Inner type method LUÔN accessible qua     │             ║
║  │    tên field: admin.User.Notify()            │             ║
║  │                                              │             ║
║  │ → Outer "override" inner khi cần!            │             ║
║  │ → Inner vẫn tồn tại và accessible!          │             ║
║  │ → Không conflict, không ambiguity!           │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ:                                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  admin.Notify() ?                             │             ║
║  │       │                                      │             ║
║  │       ▼                                      │             ║
║  │  Admin có Notify()? ─── CÓ ──→ Dùng ADMIN! │             ║
║  │       │                                      │             ║
║  │      KHÔNG                                    │             ║
║  │       │                                      │             ║
║  │       ▼                                      │             ║
║  │  Inner User có Notify()? ── CÓ ──→ PROMOTE! │             ║
║  │       │                                      │             ║
║  │      KHÔNG                                    │             ║
║  │       │                                      │             ║
║  │       ▼                                      │             ║
║  │  ❌ Compile Error!                            │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Tổng Kết & Câu Hỏi Phỏng Vấn Senior Golang

### 7.1 Câu hỏi phỏng vấn

```
╔═══════════════════════════════════════════════════════════════╗
║   CÂU HỎI PHỎNG VẤN SENIOR GOLANG                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Q1: "Method set của T và *T khác gì nhau?                  ║
║       Ảnh hưởng interface thế nào?"                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ T (value):  CHỈ methods với receiver T       │             ║
║  │ *T (pointer): methods (T) + methods (*T)      │             ║
║  │                                              │             ║
║  │ Ảnh hưởng:                                   │             ║
║  │ → func (u *User) Notify()                    │             ║
║  │ → User value KHÔNG implement Notifier!       │             ║
║  │ → *User pointer CÓ implement!               │             ║
║  │ → SendNotification(user) ← ❌ ERROR!         │             ║
║  │ → SendNotification(&user) ← ✅ OK!           │             ║
║  │                                              │             ║
║  │ Lý do: Pointer receiver = SHARE + MUTATE     │             ║
║  │ → Copy vào interface = mất reference!        │             ║
║  │ → Go bảo vệ semantic integrity!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Embedded type có phải inheritance không?"              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ KHÔNG! Là COMPOSITION!                        │             ║
║  │                                              │             ║
║  │ → Embedding = inner type TỒN TẠI bên trong  │             ║
║  │ → Admin KHÔNG PHẢI User!                     │             ║
║  │ → Không thể: var u User = admin ← ERROR!    │             ║
║  │ → Fields + methods PROMOTED nhưng            │             ║
║  │   receiver = INNER type!                      │             ║
║  │ → admin.Notify() → receiver = inner User!    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Outer và inner cùng implement interface?"             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → KHÔNG conflict! Compiler chấp nhận!        │             ║
║  │ → Outer type implementation THẮNG!           │             ║
║  │ → Inner type method vẫn accessible:          │             ║
║  │   admin.User.Notify()                        │             ║
║  │ → Nếu outer KHÔNG implement → inner PROMOTE!│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "4 quy tắc method promotion?"                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. S{T}  → S, *S promote (t T) methods ✅   │             ║
║  │ 2. S{T}  → *S thêm (t *T) methods ✅        │             ║
║  │ 3. S{*T} → S, *S promote (t T) VÀ (t *T) ✅│             ║
║  │ 4. S{T}  → S KHÔNG promote (t *T) ❌        │             ║
║  │                                              │             ║
║  │ Nhớ: CHỈ 1 trường hợp KHÔNG promote:        │             ║
║  │ Embed VALUE T, dùng VALUE S, ptr recv method │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 7.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: METHODS, INTERFACES & EMBEDDED TYPES        │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Method = function + RECEIVER:          │            │
    │  │    → Value recv: operate on COPY!         │            │
    │  │    → Pointer recv: operate on ORIGINAL!   │            │
    │  │                                          │            │
    │  │ 2. Interface = IMPLICIT implementation:   │            │
    │  │    → Không cần keyword "implements"!       │            │
    │  │    → Implement đủ methods = thỏa mãn!    │            │
    │  │                                          │            │
    │  │ 3. Method Set Rules:                      │            │
    │  │    → T: chỉ value receiver methods!      │            │
    │  │    → *T: value + pointer receiver methods!│            │
    │  │    → Ảnh hưởng interface compliance!       │            │
    │  │                                          │            │
    │  │ 4. Embedding = COMPOSITION:               │            │
    │  │    → Inner type promotion!                 │            │
    │  │    → KHÔNG phải subtyping!                │            │
    │  │    → Receiver = INNER type khi promote!   │            │
    │  │                                          │            │
    │  │ 5. Outer vs Inner implement:              │            │
    │  │    → Outer wins! No conflict!             │            │
    │  │    → Inner vẫn accessible qua field name! │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "The way methods, interfaces, and embedded types       │
    │   work together is something that makes Go very         │
    │   UNIQUE. These features help us create powerful        │
    │   constructs to achieve the same ends as OOP without    │
    │   all the complexity."                                  │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
