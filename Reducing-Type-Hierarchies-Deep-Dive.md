# Reducing Type Hierarchies — Giảm Phân Cấp Kiểu Dữ Liệu

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Reducing Type Hierarchies" của William Kennedy (2016)
> **Góc nhìn:** OOP Type Hierarchy vs Go Composition + Interface — nhóm theo BEHAVIOR
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                    | Mô tả                                          |
| --- | ----------------------------------------- | ---------------------------------------------- |
| §1  | Vấn đề: Type Hierarchy trong Go           | OOP pattern không phù hợp Go                   |
| §2  | Part I — Code sai: Animal base type       | Embedding ≠ Inheritance, Go không có subtyping |
| §3  | Tại sao Type Hierarchy thất bại           | Không thể dùng Animal làm base type            |
| §4  | Part II — Code đúng: Interface + Behavior | Nhóm theo BEHAVIOR, không theo STATE           |
| §5  | Guidelines khai báo Types                 | 5 quy tắc khai báo type                        |
| §6  | So sánh trước vs sau                      | Bảng so sánh hai cách tiếp cận                 |
| §7  | Tổng kết & Câu hỏi phỏng vấn Senior       | Ôn tập & thực hành                             |

---

## §1. Vấn Đề: Type Hierarchy Trong Go

```
╔═══════════════════════════════════════════════════════════════╗
║   TYPE HIERARCHY — VẤN ĐỀ TRONG GO                           ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Developers từ Java/C# thường mang theo thói quen:          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Java/C# (OOP truyền thống):                │             ║
║  │                                              │             ║
║  │       ┌──────────┐                           │             ║
║  │       │  Animal   │  ← Base class            │             ║
║  │       │ (parent)  │                           │             ║
║  │       └────┬──────┘                           │             ║
║  │       ┌────┴────┐                             │             ║
║  │   ┌───┴───┐ ┌───┴───┐                       │             ║
║  │   │  Dog  │ │  Cat  │  ← Subclasses         │             ║
║  │   │(child)│ │(child)│                       │             ║
║  │   └───────┘ └───────┘                       │             ║
║  │                                              │             ║
║  │  → Dog IS-A Animal!                          │             ║
║  │  → Cat IS-A Animal!                          │             ║
║  │  → Có thể: Animal a = new Dog()             │             ║
║  │  → Grouping bằng BASE TYPE!                  │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NHƯNG GO KHÁC:                                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Go KHÔNG CÓ:                                │             ║
║  │  ❌ Base types / base classes                 │             ║
║  │  ❌ Subtyping (Dog IS-A Animal)              │             ║
║  │  ❌ Inheritance (kế thừa)                    │             ║
║  │  ❌ Class hierarchy                           │             ║
║  │                                              │             ║
║  │  Go CÓ:                                      │             ║
║  │  ✅ Composition (embedding)                   │             ║
║  │  ✅ Interfaces (behavior grouping)            │             ║
║  │  ✅ Concrete types (struct)                   │             ║
║  │                                              │             ║
║  │  → Embedding ≠ Inheritance!                  │             ║
║  │  → Dog KHÔNG PHẢI IS-A Animal!               │             ║
║  │  → Dog HAS-A Animal (embedded)!              │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Part I — Code Sai: Animal Base Type

### 2.1 Anti-pattern: Type Hierarchy

```go
// ═══ ❌ ANTI-PATTERN: TYPE HIERARCHY ═══

// "Base type" — Animal chứa state chung
type Animal struct {
    Name     string
    IsMammal bool
}

// Generic Speak — KHÔNG đại diện tốt cho animal nào!
func (a Animal) Speak() {
    fmt.Println("UGH!",
        "My name is", a.Name,
        ", it is", a.IsMammal, "I am a mammal")
}

// Dog "kế thừa" Animal qua embedding
type Dog struct {
    Animal              // ← EMBED Animal!
    PackFactor int
}

// Dog override Speak
func (d Dog) Speak() {
    fmt.Println("Woof!",
        "My name is", d.Name,
        ", it is", d.IsMammal,
        "I am a mammal with a pack factor of", d.PackFactor)
}

// Cat "kế thừa" Animal qua embedding
type Cat struct {
    Animal              // ← EMBED Animal!
    ClimbFactor int
}

// Cat override Speak
func (c Cat) Speak() {
    fmt.Println("Meow!",
        "My name is", c.Name,
        ", it is", c.IsMammal,
        "I am a mammal with a climb factor of", c.ClimbFactor)
}

func main() {
    d := Dog{
        Animal:     Animal{Name: "Fido", IsMammal: true},
        PackFactor: 5,
    }
    c := Cat{
        Animal:      Animal{Name: "Milo", IsMammal: true},
        ClimbFactor: 4,
    }

    d.Speak()  // Woof!
    c.Speak()  // Meow!
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   VẤN ĐỀ VỚI CÁCH LÀM NÀY                                  ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  EMBEDDING ≠ INHERITANCE!                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Dog struct:                                  │             ║
║  │  ┌────────────────────────────┐              │             ║
║  │  │ Animal ─┐                  │              │             ║
║  │  │  Name   │ EMBEDDED value! │              │             ║
║  │  │  IsMammal                  │              │             ║
║  │  │ ────────┘                  │              │             ║
║  │  │ PackFactor                 │              │             ║
║  │  └────────────────────────────┘              │             ║
║  │                                              │             ║
║  │  → Dog CHỨA 1 Animal value BÊN TRONG!      │             ║
║  │  → Dog KHÔNG PHẢI 1 loại Animal!            │             ║
║  │  → HAS-A, không phải IS-A!                  │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  THỬ DÙNG ANIMAL NHƯ BASE TYPE:                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  // Muốn group Dog + Cat vào []Animal?      │             ║
║  │  animals := []Animal{                         │             ║
║  │      Dog{},  // ❌ COMPILER ERROR!            │             ║
║  │      Cat{},  // ❌ COMPILER ERROR!            │             ║
║  │  }                                            │             ║
║  │                                              │             ║
║  │  Error:                                       │             ║
║  │  "cannot use Dog literal (type Dog)           │             ║
║  │   as type Animal in array or slice literal"   │             ║
║  │                                              │             ║
║  │  → Dog KHÔNG PHẢI Animal!                    │             ║
║  │  → KHÔNG THỂ group bằng base type!          │             ║
║  │  → Đây là lỗi QUAN TRỌNG NHẤT!             │             ║
║  │  → Type hierarchy THẤT BẠI trong Go!        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Tại Sao Type Hierarchy Thất Bại

```
╔═══════════════════════════════════════════════════════════════╗
║   TẠI SAO TYPE HIERARCHY THẤT BẠI TRONG GO                   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  4 LÝ DO:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  1. Animal cung cấp abstraction THỪA:         │             ║
║  │     → Chỉ reuse state (Name, IsMammal)       │             ║
║  │     → Có thể COPY trực tiếp vào Dog/Cat!   │             ║
║  │                                              │             ║
║  │  2. Không ai tạo value Animal RIÊNG:          │             ║
║  │     → var a Animal ← vô nghĩa!              │             ║
║  │     → Animal không HÀNH ĐỘNG được!           │             ║
║  │     → Luôn cần Dog hoặc Cat cụ thể!        │             ║
║  │                                              │             ║
║  │  3. Animal.Speak() = generalization VÔ ÍCH:  │             ║
║  │     → "UGH!" ← không ai muốn gọi!          │             ║
║  │     → LUÔN bị override bởi Dog/Cat!          │             ║
║  │     → Method vô dụng!                        │             ║
║  │                                              │             ║
║  │  4. KHÔNG THỂ group Dog + Cat:               │             ║
║  │     → []Animal{Dog{}, Cat{}} = ERROR!         │             ║
║  │     → Go không có subtyping!                  │             ║
║  │     → Mất khả năng polymorphism!             │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "The Animal type and the use of type         │             ║
║  │  hierarchies in this case is NOT providing   │             ║
║  │  us any REAL VALUE. I would argue it is      │             ║
║  │  leading us down a path of code that is      │             ║
║  │  not READABLE, SIMPLE or ADAPTABLE."         │             ║
║  │                                              │             ║
║  │ → Type hierarchy = code KHÔNG readable!      │             ║
║  │ → KHÔNG simple! KHÔNG adaptable!             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Part II — Code Đúng: Interface + Behavior

### 4.1 Nhóm theo BEHAVIOR

```go
// ═══ ✅ ĐÚNG: INTERFACE + BEHAVIOR ═══

// Speaker — nhóm theo HÀNH VI, không theo STATE!
type Speaker interface {
    Speak()
}

// Dog — ĐỘC LẬP, CÓ ĐỦ fields riêng!
type Dog struct {
    Name       string   // ← Copy trực tiếp, không embed!
    IsMammal   bool
    PackFactor int
}

func (d Dog) Speak() {
    fmt.Println("Woof!",
        "My name is", d.Name,
        ", it is", d.IsMammal,
        "I am a mammal with a pack factor of", d.PackFactor)
}

// Cat — ĐỘC LẬP, CÓ ĐỦ fields riêng!
type Cat struct {
    Name        string  // ← Copy trực tiếp, không embed!
    IsMammal    bool
    ClimbFactor int
}

func (c Cat) Speak() {
    fmt.Println("Meow!",
        "My name is", c.Name,
        ", it is", c.IsMammal,
        "I am a mammal with a climb factor of", c.ClimbFactor)
}

func main() {
    // GROUP bằng INTERFACE — theo BEHAVIOR!
    speakers := []Speaker{
        Dog{Name: "Fido", IsMammal: true, PackFactor: 5},
        Cat{Name: "Milo", IsMammal: true, ClimbFactor: 4},
    }

    for _, spkr := range speakers {
        spkr.Speak()  // POLYMORPHISM!
    }
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   INTERFACE + BEHAVIOR — GIẢI PHÁP ĐÚNG                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  SƠ ĐỒ SO SÁNH:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ❌ TRƯỚC (Type Hierarchy):                   │             ║
║  │                                              │             ║
║  │       ┌──────────┐                           │             ║
║  │       │  Animal   │ ← Base type              │             ║
║  │       │ (struct)  │   VÔ ÍCH!               │             ║
║  │       └────┬──────┘                           │             ║
║  │       ┌────┴────┐                             │             ║
║  │   ┌───┴───┐ ┌───┴───┐                       │             ║
║  │   │  Dog  │ │  Cat  │                       │             ║
║  │   │embed  │ │embed  │                       │             ║
║  │   │Animal │ │Animal │                       │             ║
║  │   └───────┘ └───────┘                       │             ║
║  │   → KHÔNG THỂ group!                        │             ║
║  │   → []Animal{Dog{}} = ERROR!                 │             ║
║  │                                              │             ║
║  │  ════════════════════════════════             │             ║
║  │                                              │             ║
║  │  ✅ SAU (Behavior Grouping):                  │             ║
║  │                                              │             ║
║  │       ┌──────────┐                           │             ║
║  │       │ Speaker  │ ← Interface               │             ║
║  │       │(behavior)│   BEHAVIOR!               │             ║
║  │       └────┬──────┘                           │             ║
║  │       ┌────┴────┐                             ║
║  │   ┌───┴───┐ ┌───┴───┐                       │             ║
║  │   │  Dog  │ │  Cat  │                       │             ║
║  │   │ĐỘC LẬP│ │ĐỘC LẬP│                     │             ║
║  │   │Speak()│ │Speak()│                       │             ║
║  │   └───────┘ └───────┘                       │             ║
║  │   → CÓ THỂ group!                           │             ║
║  │   → []Speaker{Dog{}, Cat{}} = ✅             │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  THAY ĐỔI CHÍNH:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. BỎ Animal struct!                          │             ║
║  │    → Copy fields trực tiếp vào Dog/Cat!     │             ║
║  │                                              │             ║
║  │ 2. THÊM Speaker interface!                    │             ║
║  │    → Nhóm theo BEHAVIOR (Speak)!             │             ║
║  │                                              │             ║
║  │ 3. Dog và Cat là ĐỘC LẬP!                   │             ║
║  │    → Không embed gì!                         │             ║
║  │    → Có đủ fields riêng!                    │             ║
║  │                                              │             ║
║  │ 4. []Speaker cho phép POLYMORPHISM!           │             ║
║  │    → Group Dog + Cat bằng behavior!          │             ║
║  │    → Iterate và gọi Speak()!                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Guidelines Khai Báo Types

```
╔═══════════════════════════════════════════════════════════════╗
║   5 QUY TẮC KHAI BÁO TYPE                                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ✅ #1: Declare type đại diện MỚI hoặc ĐỘC ĐÁO!           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ✅ type Dog struct { ... }   // Dog là MỚI! │             ║
║  │ ✅ type Cat struct { ... }   // Cat là MỚI! │             ║
║  │ ❌ type Animal struct { ... } // Chỉ reuse  │             ║
║  │                                state!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ✅ #2: Validate value CÓ ĐƯỢC tạo/dùng RIÊNG?             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ✅ var d Dog ← CÓ ý nghĩa riêng!           │             ║
║  │ ❌ var a Animal ← VÔ NGHĨA! Không ai dùng! │             ║
║  │                                              │             ║
║  │ Nếu type KHÔNG BAO GIỜ được tạo riêng      │             ║
║  │ → Đặt câu hỏi: "Cần type này không?"       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ✅ #3: Embed type để REUSE BEHAVIOR cần thiết!             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ // Embed khi CẦN behavior (methods)!         │             ║
║  │ type Admin struct {                           │             ║
║  │     User            // ← CẦN User methods!  │             ║
║  │     AdminLevel int                            │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ → Embed = reuse BEHAVIOR!                    │             ║
║  │ → KHÔNG phải embed chỉ để reuse STATE!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ⚠️ #4: Question type là ALIAS hoặc ABSTRACTION!            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ❌ type ID int       // Alias, thêm gì?     │             ║
║  │ ❌ type Animal struct // Abstraction thừa!   │             ║
║  │                                              │             ║
║  │ → Type thêm GIÁ TRỊ gì?                    │             ║
║  │ → Nếu không → ĐỪNG TẠO!                     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ⚠️ #5: Question type CHỈ ĐỂ share STATE!                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ❌ type Animal struct { // CHỈ share state!  │             ║
║  │        Name     string                        │             ║
║  │        IsMammal bool                          │             ║
║  │    }                                          │             ║
║  │                                              │             ║
║  │ → Mục đích duy nhất = share state?          │             ║
║  │ → Copy fields trực tiếp TỐTVÀ ĐƠN GIẢN HƠN!│             ║
║  │ → Đừng tạo type chỉ vì sợ duplicate fields!│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. So Sánh Trước vs Sau

```
╔═══════════════════════════════════════════════════════════════╗
║   BẢNG SO SÁNH CHI TIẾT                                      ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌───────────────────┬──────────────┬──────────────┐         ║
║  │ Tiêu chí         │ ❌ Hierarchy │ ✅ Behavior  │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ Grouping         │ Base type    │ Interface    │         ║
║  │                   │ (❌ error!)  │ (✅ works!) │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ Polymorphism     │ KHÔNG có    │ CÓ!          │         ║
║  │                   │              │ []Speaker    │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ Nhóm theo        │ STATE chung  │ BEHAVIOR!    │         ║
║  │                   │ (fields)     │ (methods)    │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ Animal type      │ CÓ (thừa!) │ KHÔNG CÓ    │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ Type pollution   │ CÓ Animal   │ KHÔNG!       │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ Dog/Cat fields   │ Embedded     │ ĐỘC LẬP!   │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ Readable         │ Phức tạp    │ ĐƠN GIẢN!   │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ Adaptable        │ Cứng nhắc   │ LINH HOẠT!  │         ║
║  ├───────────────────┼──────────────┼──────────────┤         ║
║  │ Go Idiomatic     │ ❌           │ ✅          │         ║
║  └───────────────────┴──────────────┴──────────────┘         ║
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "When coding in Go try to AVOID type         │             ║
║  │  hierarchies that promote COMMON STATE       │             ║
║  │  and think about COMMON BEHAVIOR."           │             ║
║  │                                              │             ║
║  │ → TRÁNH nhóm theo state chung!               │             ║
║  │ → NGHĨ về behavior chung!                   │             ║
║  │ → Dùng INTERFACE để group!                   │             ║
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
║  Q1: "Go có inheritance không? Embedding khác gì?"          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Go KHÔNG có inheritance!                      │             ║
║  │                                              │             ║
║  │ Embedding = HAS-A, không phải IS-A!          │             ║
║  │ → Dog HAS-A Animal (chứa bên trong)         │             ║
║  │ → Dog KHÔNG IS-A Animal!                     │             ║
║  │ → KHÔNG THỂ: []Animal{Dog{}} ← ERROR!       │             ║
║  │ → Go không có subtyping!                      │             ║
║  │                                              │             ║
║  │ Embedding dùng để:                            │             ║
║  │ → Reuse BEHAVIOR (methods) cần thiết!        │             ║
║  │ → KHÔNG nên chỉ dùng để reuse STATE!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Làm sao group concrete types trong Go?"               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Bằng INTERFACE — theo BEHAVIOR!               │             ║
║  │                                              │             ║
║  │ type Speaker interface { Speak() }            │             ║
║  │                                              │             ║
║  │ speakers := []Speaker{                        │             ║
║  │     Dog{...},  // ✅                          │             ║
║  │     Cat{...},  // ✅                          │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ → Nhóm theo BEHAVIOR, không STATE!           │             ║
║  │ → Interface = contract of behavior!           │             ║
║  │ → Polymorphism qua interface!                 │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Khi nào NÊN và KHÔNG NÊN dùng embedding?"            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ NÊN: Khi cần REUSE behavior (methods)!       │             ║
║  │ → VD: Admin embed User vì cần User methods  │             ║
║  │                                              │             ║
║  │ KHÔNG NÊN: Khi chỉ reuse STATE (fields)!     │             ║
║  │ → VD: Dog embed Animal chỉ vì Name, IsMammal│             ║
║  │ → Copy fields trực tiếp đơn giản hơn!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Type pollution là gì?"                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = Khai báo type KHÔNG CẦN THIẾT!             │             ║
║  │                                              │             ║
║  │ Animal struct:                                │             ║
║  │ → Không ai tạo var a Animal riêng!           │             ║
║  │ → Animal.Speak() generalization vô dụng!     │             ║
║  │ → Chỉ tồn tại để share state!               │             ║
║  │ → TYPE POLLUTION!                             │             ║
║  │                                              │             ║
║  │ Tránh bằng: 5 guidelines khai báo type!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 7.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: REDUCING TYPE HIERARCHIES                    │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Go KHÔNG CÓ inheritance/subtyping!     │            │
    │  │    → Embedding = HAS-A, không IS-A!       │            │
    │  │                                          │            │
    │  │ 2. Nhóm theo BEHAVIOR, không STATE!       │            │
    │  │    → Interface = behavior contract!        │            │
    │  │    → []Speaker{Dog{}, Cat{}} = ✅         │            │
    │  │                                          │            │
    │  │ 3. 5 Guidelines khai báo type:            │            │
    │  │    → Đại diện cái MỚI?                   │            │
    │  │    → Có được tạo RIÊNG?                  │            │
    │  │    → Embed để reuse BEHAVIOR?             │            │
    │  │    → Là alias/abstraction THỪA?           │            │
    │  │    → Mục đích CHỈ share state?           │            │
    │  │                                          │            │
    │  │ 4. Tránh TYPE POLLUTION!                  │            │
    │  │    → Bỏ type không ai dùng riêng!        │            │
    │  │    → Copy fields > embed chỉ vì state!   │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "Try to AVOID type hierarchies that promote            │
    │   COMMON STATE and think about                          │
    │   COMMON BEHAVIOR."                                     │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
