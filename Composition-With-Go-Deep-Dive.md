# Composition With Go — Composition Trong Go

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Composition with Go" của William Kennedy (2015)
> **Góc nhìn:** Interface Composition, Type Embedding, Inner/Outer Type, Polymorphism
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                | Mô tả                                    |
| --- | ------------------------------------- | ---------------------------------------- |
| §1  | Composition là gì?                    | Paradigm thiết kế từ phần nhỏ → lớn      |
| §2  | Thiết kế Interfaces — Hành vi rời rạc | NailDriver, NailPuller, NailDrivePuller  |
| §3  | Concrete Types — Tools                | Mallet (đóng đinh) & Crowbar (nhổ đinh)  |
| §4  | Contractor — Workflow Methods         | Fasten, Unfasten, ProcessBoards          |
| §5  | Inner/Outer Type Embedding            | user embed trong admin — promotion rules |
| §6  | Toolbox — Embed Interface Types       | Interface embedding trong struct         |
| §7  | Main — Tất cả kết hợp cùng nhau       | Full example, interface conversion       |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior   | Ôn tập & thực hành                       |

---

## §1. Composition Là Gì?

```
╔═══════════════════════════════════════════════════════════════╗
║   COMPOSITION — XÂY DỰNG TỪ PHẦN NHỎ                        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Composition goes beyond the mechanics of    │             ║
║  │  type embedding. It's a PARADIGM we can      │             ║
║  │  leverage to design better APIs and to       │             ║
║  │  build larger programs from SMALLER PARTS."  │             ║
║  │                                              │             ║
║  │ → Không chỉ là embedding fields!            │             ║
║  │ → Là PARADIGM thiết kế!                     │             ║
║  │ → Xây chương trình LỚN từ phần NHỎ!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NGUYÊN TẮC:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌────────┐                                  │             ║
║  │  │ Part A │──┐                               │             ║
║  │  └────────┘  │   ┌──────────────────┐        │             ║
║  │  ┌────────┐  ├──→│  BIGGER PROGRAM  │        │             ║
║  │  │ Part B │──┤   │  (composed!)     │        │             ║
║  │  └────────┘  │   └──────────────────┘        │             ║
║  │  ┌────────┐  │                               │             ║
║  │  │ Part C │──┘                               │             ║
║  │  └────────┘                                  │             ║
║  │                                              │             ║
║  │  → Mỗi phần: MỘT mục đích duy nhất!       │             ║
║  │  → Kết hợp → chương trình hoàn chỉnh!      │             ║
║  │  → DỄ đọc, DỄ hiểu, DỄ thay đổi!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VÍ DỤ THỰC TẾ — SỬA NHÀ:                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Tình huống: Thuê thợ sửa nhà               │             ║
║  │  → Có tấm ván MỤC cần nhổ đinh ra         │             ║
║  │  → Có tấm ván MỚI cần đóng đinh vào       │             ║
║  │  → Thợ được cung cấp: đinh + dụng cụ       │             ║
║  │                                              │             ║
║  │  ┌─────────┐  ┌──────────┐  ┌──────────┐   │             ║
║  │  │ Boards  │  │  Tools   │  │Contractor│   │             ║
║  │  │(ván gỗ)│  │(dụng cụ)│  │ (thợ)    │   │             ║
║  │  └─────────┘  └──────────┘  └──────────┘   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Thiết Kế Interfaces — Hành Vi Rời Rạc

```go
// ═══ BOARD — CONCRETE DATA ═══

// Board represents a surface we can work on.
type Board struct {
    NailsNeeded int  // Số đinh CẦN
    NailsDriven int  // Số đinh ĐÃ ĐÓNG
}
```

```go
// ═══ INTERFACES — HÀNH VI RỜI RẠC ═══

// NailDriver — hành vi ĐÓNG ĐINH
type NailDriver interface {
    DriveNail(nailSupply *int, b *Board)
}

// NailPuller — hành vi NHỔ ĐINH
type NailPuller interface {
    PullNail(nailSupply *int, b *Board)
}

// NailDrivePuller — COMPOSED từ 2 interfaces trên!
type NailDrivePuller interface {
    NailDriver
    NailPuller
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   INTERFACE COMPOSITION — TỔ HỢP INTERFACE                   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CHIA NHỎ HÀNH VI:                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  NailDriver        NailPuller                │             ║
║  │  ┌────────────┐    ┌────────────┐            │             ║
║  │  │ DriveNail()│    │ PullNail() │            │             ║
║  │  │ (đóng 1   │    │ (nhổ 1    │            │             ║
║  │  │  đinh)     │    │  đinh)     │            │             ║
║  │  └─────┬──────┘    └─────┬──────┘            │             ║
║  │        │                 │                   │             ║
║  │        └────────┬────────┘                   │             ║
║  │                 ▼                            │             ║
║  │        ┌────────────────┐                    │             ║
║  │        │NailDrivePuller │                    │             ║
║  │        │ = NailDriver   │ ← COMPOSED!       │             ║
║  │        │ + NailPuller   │                    │             ║
║  │        └────────────────┘                    │             ║
║  │                                              │             ║
║  │  → Mỗi interface = 1 HÀNH VI đơn lẻ!       │             ║
║  │  → Đơn giản, rõ ràng, dễ hiểu!            │             ║
║  │  → Compose interface lớn từ interface nhỏ!  │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TẠI SAO CHIA NHỎ?                                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ❌ 1 interface lớn:                           │             ║
║  │ type Tool interface {                         │             ║
║  │     DriveNail(...)                            │             ║
║  │     PullNail(...)                             │             ║
║  │ }                                             │             ║
║  │ → Buộc MỌI tool phải có CẢ HAI method!     │             ║
║  │ → Mallet chỉ đóng, Crowbar chỉ nhổ → SAI! │             ║
║  │                                              │             ║
║  │ ✅ Interface nhỏ, rời rạc:                   │             ║
║  │ → Mallet chỉ cần implement NailDriver!      │             ║
║  │ → Crowbar chỉ cần implement NailPuller!      │             ║
║  │ → Linh hoạt! Composable! Dễ mở rộng!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Concrete Types — Tools

```go
// ═══ MALLET — CÔNG CỤ ĐÓNG ĐINH ═══

type Mallet struct{}  // Empty struct — chỉ cần behavior!

func (Mallet) DriveNail(nailSupply *int, b *Board) {
    *nailSupply--       // Lấy 1 đinh từ nguồn cung
    b.NailsDriven++     // Đóng 1 đinh vào board
    fmt.Println("Mallet: pounded nail into the board.")
}
// → Mallet implements NailDriver!

// ═══ CROWBAR — CÔNG CỤ NHỔ ĐINH ═══

type Crowbar struct{}  // Empty struct — chỉ cần behavior!

func (Crowbar) PullNail(nailSupply *int, b *Board) {
    b.NailsDriven--     // Nhổ 1 đinh từ board
    *nailSupply++       // Trả đinh lại nguồn cung
    fmt.Println("Crowbar: yanked nail out of the board.")
}
// → Crowbar implements NailPuller!
```

```
╔═══════════════════════════════════════════════════════════════╗
║   TOOLS — MỖI TOOL = 1 BEHAVIOR!                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Mallet (búa gỗ)       Crowbar (xà beng)   │             ║
║  │  ┌──────────────┐      ┌──────────────┐     │             ║
║  │  │ struct{}     │      │ struct{}     │     │             ║
║  │  │              │      │              │     │             ║
║  │  │ DriveNail()  │      │ PullNail()   │     │             ║
║  │  │ implements   │      │ implements   │     │             ║
║  │  │ NailDriver   │      │ NailPuller   │     │             ║
║  │  └──────────────┘      └──────────────┘     │             ║
║  │                                              │             ║
║  │  → Empty struct = KHÔNG CẦN state!          │             ║
║  │  → CHỈ CẦN behavior!                        │             ║
║  │  → Mỗi tool = 1 việc duy nhất!             │             ║
║  │  → Single Responsibility!                    │             ║
║  │                                              │             ║
║  │  CHÚ Ý: Value Receiver!                      │             ║
║  │  → func (Mallet) DriveNail(...)              │             ║
║  │  → Không tên receiver!                       │             ║
║  │  → Value receiver = cả value & pointer       │             ║
║  │    đều thỏa mãn interface!                   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Contractor — Workflow Methods

```go
// ═══ CONTRACTOR — THỢ SỬA NHÀ ═══

type Contractor struct{}

// Fasten — đóng đinh vào board cho ĐỦ
func (Contractor) Fasten(d NailDriver, nailSupply *int, b *Board) {
    for b.NailsDriven < b.NailsNeeded {
        d.DriveNail(nailSupply, b)    // Dùng tool!
    }
}

// Unfasten — nhổ đinh thừa khỏi board
func (Contractor) Unfasten(p NailPuller, nailSupply *int, b *Board) {
    for b.NailsDriven > b.NailsNeeded {
        p.PullNail(nailSupply, b)     // Dùng tool!
    }
}

// ProcessBoards — xử lý NHIỀU boards cùng lúc
func (c Contractor) ProcessBoards(dp NailDrivePuller, nailSupply *int, boards []Board) {
    for i := range boards {
        b := &boards[i]
        fmt.Printf("contractor: examining board #%d: %+v\n", i+1, b)

        switch {
        case b.NailsDriven < b.NailsNeeded:
            c.Fasten(dp, nailSupply, b)     // Cần đóng thêm!
        case b.NailsDriven > b.NailsNeeded:
            c.Unfasten(dp, nailSupply, b)   // Cần nhổ bớt!
        }
    }
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   CONTRACTOR — POLYMORPHIC METHODS                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  NGUYÊN TẮC QUAN TRỌNG:                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Mỗi function/method chỉ ACCEPT interface   │             ║
║  │  cho BEHAVIOR mà nó SỬ DỤNG!"               │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Fasten(d NailDriver, ...)                    │             ║
║  │  → CHỈ CẦN đóng đinh!                      │             ║
║  │  → Accept NailDriver (1 behavior)!           │             ║
║  │  → ❌ KHÔNG accept NailDrivePuller!          │             ║
║  │  → Minimal coupling!                         │             ║
║  │                                              │             ║
║  │  Unfasten(p NailPuller, ...)                  │             ║
║  │  → CHỈ CẦN nhổ đinh!                       │             ║
║  │  → Accept NailPuller (1 behavior)!           │             ║
║  │  → ❌ KHÔNG accept NailDrivePuller!          │             ║
║  │  → Minimal coupling!                         │             ║
║  │                                              │             ║
║  │  ProcessBoards(dp NailDrivePuller, ...)       │             ║
║  │  → CẦN CẢ đóng VÀ nhổ!                    │             ║
║  │  → Accept NailDrivePuller (2 behaviors)!     │             ║
║  │  → Gọi cả Fasten() VÀ Unfasten()!          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  INTERFACE CONVERSION:                                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ProcessBoards nhận dp NailDrivePuller       │             ║
║  │                                              │             ║
║  │  c.Fasten(dp, ...)                            │             ║
║  │           ↑ truyền NailDrivePuller            │             ║
║  │           ↓ param nhận NailDriver             │             ║
║  │                                              │             ║
║  │  → Compiler BIẾT: bất kỳ concrete type nào  │             ║
║  │    trong NailDrivePuller CHẮC CHẮN cũng     │             ║
║  │    implement NailDriver!                      │             ║
║  │  → INTERFACE CONVERSION tự động!              │             ║
║  │  → Compile time, KHÔNG runtime assertion!    │             ║
║  │  → An toàn! Hiệu quả!                       │             ║
║  │                                              │             ║
║  │  SƠ ĐỒ:                                      │             ║
║  │  NailDrivePuller ──→ NailDriver  ✅          │             ║
║  │  NailDrivePuller ──→ NailPuller  ✅          │             ║
║  │  NailDriver ──→ NailDrivePuller  ❌          │             ║
║  │  (thiếu PullNail!)                            │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Inner/Outer Type Embedding

### 5.1 Ví dụ user/admin

```go
// ═══ INNER/OUTER TYPE — user & admin ═══

type user struct {
    name  string
    email string
}

func (u *user) notify() {
    fmt.Printf("Sending user email To %s<%s>\n", u.name, u.email)
}

// admin EMBED user — inner type!
type admin struct {
    user          // ← INNER type!
    level string
}

func main() {
    ad := admin{
        user:  user{name: "john smith", email: "john@yahoo.com"},
        level: "super",
    }

    ad.user.notify()  // Gọi QUA inner type value
    ad.notify()       // Inner type PROMOTED → gọi trực tiếp!
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   INNER/OUTER TYPE — PROMOTION!                              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  MEMORY:                                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  admin (OUTER type):                          │             ║
║  │  ┌──────────────────────────────────┐        │             ║
║  │  │                                  │        │             ║
║  │  │ user (INNER type):               │        │             ║
║  │  │ ┌──────────────────────────┐     │        │             ║
║  │  │ │ name:  "john smith"      │     │        │             ║
║  │  │ │ email: "john@yahoo.com"  │     │        │             ║
║  │  │ │ notify() method          │     │        │             ║
║  │  │ └──────────────────────────┘     │        │             ║
║  │  │                                  │        │             ║
║  │  │ level: "super"                   │        │             ║
║  │  └──────────────────────────────────┘        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  PROMOTION RULES:                                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ad.user.notify()  → Truy cập TRỰC TIẾP!   │             ║
║  │  ad.notify()       → PROMOTED! Tự gọi được!│             ║
║  │  ad.name           → PROMOTED! Tự truy cập!│             ║
║  │  ad.email          → PROMOTED! Tự truy cập!│             ║
║  │                                              │             ║
║  │  → Fields + methods của inner type          │             ║
║  │    được PROMOTED lên outer type!             │             ║
║  │  → Gọi như thể là CỦA outer type!          │             ║
║  │                                              │             ║
║  │  ⚠️ LƯU Ý QUAN TRỌNG:                       │             ║
║  │  → Đây KHÔNG PHẢI subtyping!                │             ║
║  │  → admin KHÔNG PHẢI user!                    │             ║
║  │  → Không thể: var u user = ad ← ERROR!     │             ║
║  │  → admin CHỈ LÀ admin!                      │             ║
║  │  → Inner type value TỒN TẠI RIÊNG bên trong!│             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Toolbox — Embed Interface Types

```go
// ═══ TOOLBOX — EMBED INTERFACE! ═══

type Toolbox struct {
    NailDriver    // ← Embed INTERFACE type!
    NailPuller    // ← Embed INTERFACE type!
    nails int     // Số đinh còn lại
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   TOOLBOX — EMBED INTERFACE TYPES TRONG STRUCT!               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ĐIỂM ĐẶC BIỆT: EMBED INTERFACE, KHÔNG PHẢI STRUCT!        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Toolbox (OUTER type):                        │             ║
║  │  ┌──────────────────────────────────┐        │             ║
║  │  │                                  │        │             ║
║  │  │ NailDriver (INNER interface):    │        │             ║
║  │  │ ┌──────────────────────┐         │        │             ║
║  │  │ │ = Mallet{}           │ ← Gán!  │        │             ║
║  │  │ │ DriveNail() ✅       │ PROMOTED│        │             ║
║  │  │ └──────────────────────┘         │        │             ║
║  │  │                                  │        │             ║
║  │  │ NailPuller (INNER interface):    │        │             ║
║  │  │ ┌──────────────────────┐         │        │             ║
║  │  │ │ = Crowbar{}          │ ← Gán!  │        │             ║
║  │  │ │ PullNail() ✅        │ PROMOTED│        │             ║
║  │  │ └──────────────────────┘         │        │             ║
║  │  │                                  │        │             ║
║  │  │ nails: 10                        │        │             ║
║  │  └──────────────────────────────────┘        │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  KẾT QUẢ CỦA EMBEDDING:                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  → tb.DriveNail(...) = Mallet.DriveNail()    │             ║
║  │  → tb.PullNail(...)  = Crowbar.PullNail()    │             ║
║  │                                              │             ║
║  │  Toolbox có CẢ DriveNail() VÀ PullNail()!   │             ║
║  │  → Toolbox implements NailDriver!  ✅        │             ║
║  │  → Toolbox implements NailPuller!  ✅        │             ║
║  │  → Toolbox implements NailDrivePuller! ✅    │             ║
║  │                                              │             ║
║  │  SỨC MẠNH:                                    │             ║
║  │  → Embed 2 interface nhỏ                    │             ║
║  │  → Tự động thỏa mãn interface COMPOSED!    │             ║
║  │  → Có thể truyền Toolbox vào ProcessBoards! │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Main — Tất Cả Kết Hợp Cùng Nhau

```go
// ═══ FULL EXAMPLE ═══

func main() {
    // Boards: ván mục (cần nhổ đinh) + ván mới (cần đóng đinh)
    boards := []Board{
        // Rotted boards — cần nhổ đinh!
        {NailsDriven: 3},
        {NailsDriven: 1},
        {NailsDriven: 6},
        // Fresh boards — cần đóng đinh!
        {NailsNeeded: 6},
        {NailsNeeded: 9},
        {NailsNeeded: 4},
    }

    // Toolbox: gán Mallet + Crowbar + 10 đinh
    tb := Toolbox{
        NailDriver: Mallet{},    // Đóng bằng búa!
        NailPuller: Crowbar{},   // Nhổ bằng xà beng!
        nails:      10,
    }

    // Thuê thợ + làm việc!
    var c Contractor
    c.ProcessBoards(&tb, &tb.nails, boards)
    //               ↑ Toolbox thỏa NailDrivePuller!
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   COMPOSITION IN ACTION — FULL FLOW                           ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  SƠ ĐỒ LUỒNG THỰC THI:                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  1. Tạo Toolbox:                              │             ║
║  │     ┌────────────────────────┐               │             ║
║  │     │ Toolbox                │               │             ║
║  │     │ ├─ NailDriver: Mallet  │               │             ║
║  │     │ ├─ NailPuller: Crowbar │               │             ║
║  │     │ └─ nails: 10           │               │             ║
║  │     └────────────────────────┘               │             ║
║  │                │                             │             ║
║  │  2. ProcessBoards(&tb, ...)                   │             ║
║  │     → &tb implements NailDrivePuller? ✅     │             ║
║  │     → Có DriveNail (qua Mallet)!             │             ║
║  │     → Có PullNail (qua Crowbar)!             │             ║
║  │                │                             │             ║
║  │  3. Duyệt boards:                            │             ║
║  │     ┌─────────────────────────────────┐      │             ║
║  │     │ Board{NailsDriven:3, Needed:0}  │      │             ║
║  │     │ → Driven > Needed → UNFASTEN!   │      │             ║
║  │     │ → Gọi Crowbar.PullNail() x3    │      │             ║
║  │     ├─────────────────────────────────┤      │             ║
║  │     │ Board{NailsDriven:0, Needed:6}  │      │             ║
║  │     │ → Driven < Needed → FASTEN!     │      │             ║
║  │     │ → Gọi Mallet.DriveNail() x6    │      │             ║
║  │     └─────────────────────────────────┘      │             ║
║  │                                              │             ║
║  │  4. Interface Conversion (compile time!):     │             ║
║  │     ProcessBoards(dp NailDrivePuller)         │             ║
║  │        ↓                                     │             ║
║  │     c.Fasten(dp, ...)    ← dp truyền vào    │             ║
║  │     Fasten(d NailDriver) ← param là NailDriver│            ║
║  │        ↓                                     │             ║
║  │     Compiler: "NailDrivePuller bao gồm      │             ║
║  │     NailDriver → cho phép!"                   │             ║
║  │     → INTERFACE CONVERSION, KHÔNG assertion! │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  LINH HOẠT — THAY ĐỔI TOOLS DỄ DÀNG:                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ // Dùng búa khác? Chỉ cần implement!        │             ║
║  │ type PowerDrill struct{}                      │             ║
║  │ func (PowerDrill) DriveNail(...) { ... }      │             ║
║  │                                              │             ║
║  │ tb := Toolbox{                                │             ║
║  │     NailDriver: PowerDrill{}, // Đổi tool!  │             ║
║  │     NailPuller: Crowbar{},                    │             ║
║  │     nails:      10,                           │             ║
║  │ }                                             │             ║
║  │ // → Contractor code KHÔNG ĐỔI!             │             ║
║  │ // → ProcessBoards vẫn hoạt động!           │             ║
║  │ // → Decoupling CỰC ĐẠI!                   │             ║
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
║  Q1: "Composition trong Go là gì? Khác inheritance?"        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Composition = xây từ PHẦN NHỎ!              │             ║
║  │ → Embed types = HAS-A (có), không IS-A!     │             ║
║  │ → Inner type promotion = access trực tiếp!  │             ║
║  │ → KHÔNG phải subtyping!                      │             ║
║  │ → admin KHÔNG PHẢI user!                     │             ║
║  │                                              │             ║
║  │ Inheritance (Java):                           │             ║
║  │ → Dog IS-A Animal (kế thừa)                  │             ║
║  │ → Hierarchy cứng nhắc                        │             ║
║  │                                              │             ║
║  │ Composition (Go):                             │             ║
║  │ → Dog HAS-A Speaker behavior                 │             ║
║  │ → Linh hoạt, decoupled                       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Tại sao accept interface NHỎ NHẤT có thể?"           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Fasten chỉ CẦN NailDriver → accept nó!   │             ║
║  │ → KHÔNG accept NailDrivePuller!              │             ║
║  │                                              │             ║
║  │ Lý do:                                        │             ║
║  │ → Minimal coupling!                           │             ║
║  │ → Dễ hiểu: đọc param biết hàm làm gì!     │             ║
║  │ → Dễ dùng: ít requirement = nhiều           │             ║
║  │   concrete types có thể truyền vào!          │             ║
║  │ → Interface Segregation Principle! (SOLID)    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Embed interface trong struct là gì?"                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ type Toolbox struct {                         │             ║
║  │     NailDriver  // ← Interface embedded!     │             ║
║  │     NailPuller                                │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ → Gán concrete type VÀO interface field!     │             ║
║  │ → Inner type promotion = methods promoted!   │             ║
║  │ → Toolbox tự động implement:                │             ║
║  │   NailDriver + NailPuller + NailDrivePuller! │             ║
║  │ → Thay tool = chỉ thay concrete type!       │             ║
║  │ → Decoupling + flexibility!                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Interface Conversion vs Type Assertion?"               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Interface Conversion (compile time):          │             ║
║  │ → NailDrivePuller → NailDriver                │             ║
║  │ → Compiler BIẾT mối quan hệ!                │             ║
║  │ → An toàn! KHÔNG runtime cost!               │             ║
║  │                                              │             ║
║  │ Type Assertion (runtime):                     │             ║
║  │ → v, ok := iface.(ConcreteType)               │             ║
║  │ → Runtime check! Có thể panic!               │             ║
║  │                                              │             ║
║  │ → Interface Conversion TỐT HƠN khi có thể! │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: COMPOSITION WITH GO                         │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Chia behavior thành interface NHỎ:     │            │
    │  │    → NailDriver (1 method)                │            │
    │  │    → NailPuller (1 method)                │            │
    │  │    → Compose → NailDrivePuller!           │            │
    │  │                                          │            │
    │  │ 2. Accept interface NHỎ NHẤT cần thiết:   │            │
    │  │    → Fasten: chỉ NailDriver!             │            │
    │  │    → Unfasten: chỉ NailPuller!           │            │
    │  │    → ProcessBoards: NailDrivePuller!       │            │
    │  │                                          │            │
    │  │ 3. Embedding = Inner/Outer type:          │            │
    │  │    → Fields + methods = PROMOTED!          │            │
    │  │    → KHÔNG phải subtyping!                │            │
    │  │    → Inner type vẫn tồn tại riêng!       │            │
    │  │                                          │            │
    │  │ 4. Embed INTERFACE trong struct:           │            │
    │  │    → Toolbox embed NailDriver+NailPuller! │            │
    │  │    → Tự động implement NailDrivePuller!   │            │
    │  │    → Thay tool = thay concrete type!      │            │
    │  │                                          │            │
    │  │ 5. Interface Conversion compile-time:     │            │
    │  │    → Compiler biết static relationship!    │            │
    │  │    → An toàn, không runtime cost!         │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "Programs that are architected with composition        │
    │   in mind have a better chance to GROW and ADAPT        │
    │   to changing needs. They are also much easier          │
    │   to READ and REASON about."                            │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
