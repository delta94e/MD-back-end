# Interface Values Are Valueless — Interface "Vô Giá Trị"

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Interface Values Are Valueless" của William Kennedy (2018)
> **Góc nhìn:** Data Oriented Design, Polymorphism, Interface = Behavior Contract
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                              | Mô tả                                      |
| --- | ----------------------------------- | ------------------------------------------ |
| §1  | Data Oriented Design                | DOD vs OOD — dữ liệu là trung tâm          |
| §2  | Concrete Data — Struct              | Dữ liệu cụ thể tạo bằng struct             |
| §3  | Interface = Valueless!              | Interface KHÔNG CÓ giá trị thật            |
| §4  | Polymorphism — Đa hình              | Hàm hoạt động khác dựa trên data           |
| §5  | Giving Data Behavior — Methods      | Cho data hành vi để thỏa interface         |
| §6  | Ví dụ đầy đủ: file & pipe           | Polymorphic function hoàn chỉnh            |
| §7  | Interface Value Assignments         | Gán interface cho nhau = gán concrete data |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior | Ôn tập & thực hành                         |

---

## §1. Data Oriented Design — Thiết Kế Hướng Dữ Liệu

### 1.1 DOD vs OOD

```
╔═══════════════════════════════════════════════════════════════╗
║   DATA ORIENTED DESIGN (DOD)                                  ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "I believe in Data Oriented Design over      │             ║
║  │  Object Oriented Design when it comes to     │             ║
║  │  writing code in Go."                        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  LUẬT SỐ 1 CỦA DOD:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "If you don't understand the DATA you are    │             ║
║  │  working with, you don't understand the      │             ║
║  │  PROBLEM you are trying to solve."           │             ║
║  │                                              │             ║
║  │ → Không hiểu DATA = không hiểu BÀI TOÁN!  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  MỌI BÀI TOÁN = BIẾN ĐỔI DỮ LIỆU:                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌─────────┐    ┌───────────┐   ┌────────┐  │             ║
║  │  │  INPUT  │───→│ ALGORITHM │──→│ OUTPUT │  │             ║
║  │  │  (data) │    │ (biến đổi)│   │ (data) │  │             ║
║  │  └─────────┘    └───────────┘   └────────┘  │             ║
║  │                                              │             ║
║  │  → Mọi chương trình: nhận INPUT → tạo OUTPUT│             ║
║  │  → Mỗi function = 1 phép biến đổi NHỎ     │             ║
║  │  → Tập hợp function = giải bài toán LỚN   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  LUẬT SỐ 2 CỦA DOD:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "When data is CHANGING, your PROBLEM is      │             ║
║  │  changing. When the problem is changing,     │             ║
║  │  the ALGORITHMS need to change."             │             ║
║  │                                              │             ║
║  │ → Data thay đổi → bài toán thay đổi!      │             ║
║  │ → Bài toán thay đổi → algorithm phải đổi!  │             ║
║  │                                              │             ║
║  │ CẦN:                                         │             ║
║  │ → Algorithm NHỎ, CHÍNH XÁC cho mỗi phép   │             ║
║  │   biến đổi!                                  │             ║
║  │ → Thay đổi algorithm KHÔNG gây cascade!     │             ║
║  │ → ĐÂY LÀ LÚC INTERFACE CẦN THIẾT!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Concrete Data — Struct

### 2.1 Dữ liệu cụ thể

```
╔═══════════════════════════════════════════════════════════════╗
║   CONCRETE DATA — DỮ LIỆU CỤ THỂ                           ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CONCRETE = CỤ THỂ = CÓ THẬT trong memory!                 ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ // Keyword STRUCT → tạo CONCRETE type!       │             ║
║  │                                              │             ║
║  │ type file struct {                            │             ║
║  │     name string                               │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ type pipe struct {                            │             ║
║  │     name string                               │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ func main() {                                 │             ║
║  │     var f file   // f = CONCRETE value!       │             ║
║  │     var p pipe   // p = CONCRETE value!       │             ║
║  │ }                                             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CONCRETE DATA:                                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Memory:                                      │             ║
║  │  ┌──────────────────────┐                    │             ║
║  │  │ f = file{...}        │ ← CÓ THẬT!       │             ║
║  │  │ → chiếm memory!     │                    │             ║
║  │  │ → có address!        │                    │             ║
║  │  │ → có thể thao tác! │                    │             ║
║  │  ├──────────────────────┤                    │             ║
║  │  │ p = pipe{...}        │ ← CÓ THẬT!       │             ║
║  │  │ → chiếm memory!     │                    │             ║
║  │  │ → có address!        │                    │             ║
║  │  │ → có thể thao tác! │                    │             ║
║  │  └──────────────────────┘                    │             ║
║  │                                              │             ║
║  │  → struct = CONCRETE (cụ thể, có thật!)     │             ║
║  │  → Có thể lưu trữ, gửi qua network,       │             ║
║  │    ghi vào file, thao tác trực tiếp!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Interface = Valueless!

### 3.1 Interface KHÔNG CÓ giá trị thật

```
╔═══════════════════════════════════════════════════════════════╗
║   INTERFACE = VALUELESS! — "VÔ GIÁ TRỊ"                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  HAI KEYWORDS TẠO TYPE:                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  struct    → CONCRETE type (có thật!)        │             ║
║  │  interface → VALUELESS type (vô giá trị!)   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SO SÁNH TRỰC QUAN:                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  STRUCT:                                      │             ║
║  │  ┌────────────────────────────┐              │             ║
║  │  │ type file struct {         │              │             ║
║  │  │     name string            │ ← CÓ DATA!  │             ║
║  │  │ }                          │   CỤ THỂ!   │             ║
║  │  └────────────────────────────┘              │             ║
║  │  var f file  → f CHỨA data thật!            │             ║
║  │                                              │             ║
║  │  ═══════════════════════════════              │             ║
║  │                                              │             ║
║  │  INTERFACE:                                   │             ║
║  │  ┌────────────────────────────┐              │             ║
║  │  │ type reader interface {    │              │             ║
║  │  │     read(b []byte)         │ ← CHỈ CÓ   │             ║
║  │  │         (int, error)       │   BEHAVIOR!  │             ║
║  │  │ }                          │   KHÔNG DATA!│             ║
║  │  └────────────────────────────┘              │             ║
║  │  var r reader → r VALUELESS! KHÔNG CÓ GÌ!  │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "An interface type defines and creates       │             ║
║  │  values that are VALUELESS!"                 │             ║
║  │                                              │             ║
║  │ "Boom! Head Explodes."                       │             ║
║  │                                              │             ║
║  │ BAO GIỜ CŨNG NHỚ:                            │             ║
║  │ • var r reader → r KHÔNG CÓ GÌ thật!       │             ║
║  │ • r KHÔNG concrete!                           │             ║
║  │ • r KHÔNG có data riêng!                     │             ║
║  │ • r là VALUELESS!                             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ SO SÁNH:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ var f file               var r reader        │             ║
║  │ ┌──────────────┐       ┌──────────────┐     │             ║
║  │ │ name: ""     │       │   (nil)      │     │             ║
║  │ │              │       │   NOTHING!   │     │             ║
║  │ │ CÓ THẬT!    │       │   VALUELESS! │     │             ║
║  │ │ CÓ MEMORY!  │       │   NO DATA!   │     │             ║
║  │ └──────────────┘       └──────────────┘     │             ║
║  │       ▲                       ▲              │             ║
║  │    CONCRETE              VALUELESS           │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Polymorphism — Đa Hình

### 4.1 Polymorphism theo Tom Kurtz

```
╔═══════════════════════════════════════════════════════════════╗
║   POLYMORPHISM — ĐA HÌNH                                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Tom Kurtz (inventor of BASIC):                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Polymorphism means that you write a certain │             ║
║  │  PROGRAM and it behaves DIFFERENTLY depending│             ║
║  │  on the DATA that it operates on."           │             ║
║  │                                              │             ║
║  │ → Viết 1 chương trình                        │             ║
║  │ → Hoạt động KHÁC NHAU                       │             ║
║  │ → Tùy thuộc vào DATA                        │             ║
║  │ → DATA là DRIVER của polymorphism!           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  POLYMORPHIC FUNCTION:                                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ func retrieve(r reader) error {               │             ║
║  │     data := make([]byte, 100)                 │             ║
║  │     len, err := r.read(data)                  │             ║
║  │     if err != nil {                           │             ║
║  │         return err                            │             ║
║  │     }                                         │             ║
║  │     fmt.Println(string(data[:len]))           │             ║
║  │     return nil                                │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ → r reader = VALUELESS parameter!            │             ║
║  │ → Không tồn tại "value of type reader"!     │             ║
║  │ → Vậy hàm này THẬT SỰ nói gì?              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  HÀM NÀY THẬT SỰ NÓI:                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ "Tôi chấp nhận BẤT KỲ concrete data nào    │             ║
║  │  (value hoặc pointer) mà IMPLEMENT contract │             ║
║  │  reader. Tức là implement ĐẦY ĐỦ method    │             ║
║  │  set behavior: read(b []byte) (int, error)"  │             ║
║  │                                              │             ║
║  │ → KHÔNG bound vào 1 concrete type!           │             ║
║  │ → Bound vào BEHAVIOR (hành vi)!              │             ║
║  │ → Đây là POLYMORPHISM trong Go!              │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ:                                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │          reader interface                     │             ║
║  │          ┌────────────┐                      │             ║
║  │          │ read(...)  │ ← BEHAVIOR!          │             ║
║  │          └─────┬──────┘                      │             ║
║  │                │                             │             ║
║  │       ┌────────┼────────┐                    │             ║
║  │       │                 │                    │             ║
║  │  ┌────┴────┐      ┌────┴────┐               │             ║
║  │  │  file   │      │  pipe   │               │             ║
║  │  │ (struct)│      │ (struct)│               │             ║
║  │  │CONCRETE!│      │CONCRETE!│               │             ║
║  │  │read(...)|      │read(...)|               │             ║
║  │  └─────────┘      └─────────┘               │             ║
║  │                                              │             ║
║  │  retrieve(f)  → read từ file!               │             ║
║  │  retrieve(p)  → read từ pipe!               │             ║
║  │  CÙNG function, KHÁC behavior!               │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Giving Data Behavior — Methods

### 5.1 Methods cho data hành vi

```
╔═══════════════════════════════════════════════════════════════╗
║   GIVING DATA BEHAVIOR — CHO DATA HÀNH VI                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CÂU HỎI: Làm sao data "có" behavior?                       ║
║  TRẢ LỜI: Bằng METHODS!                                     ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ // Interface chỉ ĐỊNH NGHĨA behavior!       │             ║
║  │ type reader interface {                       │             ║
║  │     read(b []byte) (int, error)               │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ // Concrete type file                         │             ║
║  │ type file struct {                            │             ║
║  │     name string                               │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ // CHO file behavior "read"!                  │             ║
║  │ func (file) read(b []byte) (int, error) {     │             ║
║  │     s := "<rss>...</rss>"                     │             ║
║  │     copy(b, s)                                │             ║
║  │     return len(s), nil                        │             ║
║  │ }                                             │             ║
║  │ // → file bây giờ CÓ behavior "read"!       │             ║
║  │ // → file THỎA MÃN interface reader!        │             ║
║  │                                              │             ║
║  │ // Concrete type pipe                         │             ║
║  │ type pipe struct {                            │             ║
║  │     name string                               │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ // CHO pipe behavior "read"!                  │             ║
║  │ func (pipe) read(b []byte) (int, error) {     │             ║
║  │     s := `{name: "bill"}`                     │             ║
║  │     copy(b, s)                                │             ║
║  │     return len(s), nil                        │             ║
║  │ }                                             │             ║
║  │ // → pipe cũng THỎA MÃN interface reader!   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CHÚ Ý RECEIVER KHÔNG CÓ TÊN:                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ func (file) read(...)                         │             ║
║  │       ↑ không có tên biến!                   │             ║
║  │                                              │             ║
║  │ → Khi method KHÔNG CẦN truy cập receiver    │             ║
║  │ → Bỏ tên = common practice!                 │             ║
║  │ → Vẫn implement interface bình thường!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  FUNCTION vs METHOD:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Function: func process(data []byte)           │             ║
║  │ → Không gắn với type nào                     │             ║
║  │                                              │             ║
║  │ Method: func (f file) read(b []byte)          │             ║
║  │ → GẮN với type file                          │             ║
║  │ → CHO file behavior!                         │             ║
║  │ → Giúp file thỏa mãn interface!             │             ║
║  │                                              │             ║
║  │ CHỌN method KHI:                              │             ║
║  │ → Data cần behavior để thỏa interface!      │             ║
║  │ → Đây là LÝ DO CHÍNH dùng method!           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Ví Dụ Đầy Đủ: file & pipe

### 6.1 Polymorphic example hoàn chỉnh

```go
// ═══ POLYMORPHIC EXAMPLE HOÀN CHỈNH ═══

package main

import "fmt"

type reader interface {
    read(b []byte) (int, error)
}

type file struct{ name string }

func (file) read(b []byte) (int, error) {
    s := "<rss><channel><title>Going Go</title></channel></rss>"
    copy(b, s)
    return len(s), nil
}

type pipe struct{ name string }

func (pipe) read(b []byte) (int, error) {
    s := `{name: "bill", title: "developer"}`
    copy(b, s)
    return len(s), nil
}

func main() {
    f := file{"data.json"}
    p := pipe{"cfg_service"}

    retrieve(f)   // Concrete data: file
    retrieve(p)   // Concrete data: pipe
}

// POLYMORPHIC FUNCTION!
func retrieve(r reader) error {
    data := make([]byte, 100)
    len, err := r.read(data)
    if err != nil {
        return err
    }
    fmt.Println(string(data[:len]))
    return nil
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   POLYMORPHISM IN ACTION!                                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  OUTPUT:                                                       ║
║  <rss><channel><title>Going Go</title></channel></rss>        ║
║  {name: "bill", title: "developer"}                           ║
║                                                               ║
║  SƠ ĐỒ THỰC THI:                                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  retrieve(f)  — f là file                    │             ║
║  │  ┌─────────────────────┐                     │             ║
║  │  │ r = reader          │                     │             ║
║  │  │ ┌─────────────────┐ │                     │             ║
║  │  │ │ type: file      │ │                     │             ║
║  │  │ │ data: file{...} │ │ ← CONCRETE data!   │             ║
║  │  │ └─────────────────┘ │                     │             ║
║  │  │ r.read() → file.read() → XML output!    │             ║
║  │  └─────────────────────┘                     │             ║
║  │                                              │             ║
║  │  retrieve(p)  — p là pipe                    │             ║
║  │  ┌─────────────────────┐                     │             ║
║  │  │ r = reader          │                     │             ║
║  │  │ ┌─────────────────┐ │                     │             ║
║  │  │ │ type: pipe      │ │                     │             ║
║  │  │ │ data: pipe{...} │ │ ← CONCRETE data!   │             ║
║  │  │ └─────────────────┘ │                     │             ║
║  │  │ r.read() → pipe.read() → JSON output!   │             ║
║  │  └─────────────────────┘                     │             ║
║  │                                              │             ║
║  │  CÙNG hàm retrieve → KHÁC behavior!         │             ║
║  │  → Vì CONCRETE DATA khác nhau!              │             ║
║  │  → ĐÂY LÀ POLYMORPHISM!                     │             ║
║  │                                              │             ║
║  │  Kennedy:                                     │             ║
║  │  "The retrieve function is NOT bound to a    │             ║
║  │   single piece of concrete data, but bound   │             ║
║  │   to any concrete data that exhibits the     │             ║
║  │   read BEHAVIOR."                            │             ║
║  │                                              │             ║
║  │  DECOUPLING CỰC ĐẠI:                        │             ║
║  │  → Biết CHÍNH XÁC behavior cần thiết!       │             ║
║  │  → Không generalized, không hidden!           │             ║
║  │  → Interface = PRECISE decoupling!           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Interface Value Assignments

### 7.1 Gán interface = gán concrete data!

```go
// ═══ INTERFACE VALUE ASSIGNMENTS ═══

type Reader interface {
    Read()
}

type Writer interface {
    Write()
}

type ReadWriter interface {
    Reader
    Writer
}

type system struct {
    Host string
}

func (*system) Read()  { /* ... */ }
func (*system) Write() { /* ... */ }

func main() {
    var rw ReadWriter = &system{"127.0.0.1"}
    var r  Reader     = rw    // ← GÁN interface cho nhau!
    fmt.Println(rw, r)
}

// OUTPUT: &{127.0.0.1} &{127.0.0.1}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   INTERFACE ASSIGNMENTS — GÁN CONCRETE DATA!                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CÂU HỎI: rw (ReadWriter) gán cho r (Reader)?               ║
║  → Hai KHÁC named type → bình thường KHÔNG ĐƯỢC!            ║
║  → Go không implicit type conversion!                         ║
║  → VẬY TẠI SAO ĐƯỢC?                                         ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ TRẢ LỜI: Vì interface = VALUELESS!           │             ║
║  │                                              │             ║
║  │ rw và r KHÔNG PHẢI biến thật!               │             ║
║  │ → KHÔNG GÁN interface cho interface!         │             ║
║  │ → GÁN CONCRETE DATA bên trong!              │             ║
║  │                                              │             ║
║  │ Code không gán rw → r                        │             ║
║  │ Code gán: concrete data TRONG rw → r!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ:                                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ var rw ReadWriter = &system{"127.0.0.1"}     │             ║
║  │ ┌───────────────────┐                        │             ║
║  │ │ rw (ReadWriter)   │                        │             ║
║  │ │ ┌───────────────┐ │                        │             ║
║  │ │ │ type: *system  │ │     ┌──────────────┐  │             ║
║  │ │ │ data: ─────────┼─┼───→ │system{       │  │             ║
║  │ │ └───────────────┘ │     │ "127.0.0.1"  │  │             ║
║  │ └───────────────────┘     │}              │  │             ║
║  │                           └──────────────┘  │             ║
║  │ var r Reader = rw                            │             ║
║  │ ┌───────────────────┐           ▲            │             ║
║  │ │ r (Reader)        │           │            │             ║
║  │ │ ┌───────────────┐ │           │            │             ║
║  │ │ │ type: *system  │ │           │            │             ║
║  │ │ │ data: ─────────┼─┼───────────┘            │             ║
║  │ │ └───────────────┘ │                        │             ║
║  │ └───────────────────┘                        │             ║
║  │                                              │             ║
║  │ → CẢ HAI trỏ tới CÙNG concrete data!       │             ║
║  │ → Output: cả hai in &{127.0.0.1}            │             ║
║  │ → Compiler kiểm tra: concrete data TRONG rw │             ║
║  │   có thỏa Reader interface không?            │             ║
║  │ → *system có Read()? CÓ! → CHO PHÉP!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  QUY TẮC:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Gán interface = gán CONCRETE DATA!         │             ║
║  │ → Compiler validate: concrete data có thỏa  │             ║
║  │   interface đích không?                       │             ║
║  │ → fmt.Println hiển thị CONCRETE DATA!        │             ║
║  │ → Interface bản thân = NOTHING TO DISPLAY!   │             ║
║  │ → "We can only work with concrete data!"     │             ║
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
║  Q1: "'Interface values are valueless' nghĩa là gì?"        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Interface type = KHÔNG có data riêng!      │             ║
║  │ → var r reader → r KHÔNG chứa gì cụ thể!   │             ║
║  │ → Chỉ khi lưu concrete data vào → mới có   │             ║
║  │   ý nghĩa!                                   │             ║
║  │ → struct = CONCRETE (có thật)                 │             ║
║  │   interface = VALUELESS (vô giá trị)         │             ║
║  │ → Thiết kế dựa trên BEHAVIOR, không DATA!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Polymorphism trong Go hoạt động thế nào?"             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Tom Kurtz: "Viết 1 chương trình, hoạt động  │             ║
║  │ KHÁC NHAU tùy DATA!"                        │             ║
║  │                                              │             ║
║  │ → Định nghĩa interface (behavior contract)  │             ║
║  │ → Nhiều concrete types implement interface   │             ║
║  │ → Polymorphic function nhận interface param  │             ║
║  │ → Truyền concrete data vào → khác behavior! │             ║
║  │ → Decoupling CHÍNH XÁC và RẠCH RÒI!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Data Oriented Design trong Go là gì?"                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Luật 1: Không hiểu DATA = không hiểu PROBLEM│             ║
║  │ Luật 2: DATA thay đổi → algorithm thay đổi! │             ║
║  │                                              │             ║
║  │ → Mọi bài toán = DATA TRANSFORMATION!        │             ║
║  │ → Algorithm NHỎ, CHÍNH XÁC cho mỗi phép    │             ║
║  │   biến đổi!                                  │             ║
║  │ → Interface cho phép thay đổi algorithm      │             ║
║  │   KHÔNG cascade!                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Tại sao gán ReadWriter sang Reader được?              ║
║       Go không có implicit conversion mà?"                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Vì interface = VALUELESS!                   │             ║
║  │ → KHÔNG gán interface cho interface!          │             ║
║  │ → GÁN concrete data bên trong!              │             ║
║  │ → Compiler validate: concrete data có thỏa  │             ║
║  │   interface đích?                             │             ║
║  │ → *system có Read()? CÓ → cho phép!          │             ║
║  │ → Quy tắc này KHÁC với type conversion!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: INTERFACE VALUES ARE VALUELESS               │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. struct = CONCRETE (có thật, có data!) │            │
    │  │    interface = VALUELESS (vô giá trị!)   │            │
    │  │                                          │            │
    │  │ 2. Data Oriented Design:                  │            │
    │  │    → Hiểu DATA = hiểu PROBLEM!           │            │
    │  │    → Mọi bài toán = data transformation! │            │
    │  │                                          │            │
    │  │ 3. Polymorphism = CONCRETE DATA drive!    │            │
    │  │    → 1 function, khác behavior tùy data! │            │
    │  │    → Interface = behavior contract!       │            │
    │  │                                          │            │
    │  │ 4. Methods = cho data BEHAVIOR!           │            │
    │  │    → Data + method = thỏa interface!     │            │
    │  │    → CHỌN method khi cần thỏa interface! │            │
    │  │                                          │            │
    │  │ 5. Interface assignments = gán CONCRETE!  │            │
    │  │    → rw → r = gán concrete data bên trong│            │
    │  │    → Compiler validate behavior match!    │            │
    │  │                                          │            │
    │  │ 6. Decoupling = FOCUS ON BEHAVIOR!        │            │
    │  │    → Precise, not generalized!            │            │
    │  │    → Readable, not hidden!                │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "It all makes sense when you accept that               │
    │   interface values are VALUELESS. The function           │
    │   is not asking for a reader value because              │
    │   they don't exist. The function is asking              │
    │   for CONCRETE DATA that exhibits the                   │
    │   correct BEHAVIORS."                                   │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
