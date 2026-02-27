# Go Variables — Deep Dive: Biến, Bộ Nhớ & Hệ Thống Kiểu Dữ Liệu

> **Nguồn gốc**: Ultimate Go — Variables (William Kennedy / Ardan Labs)
> **Mục đích**: Tài liệu ôn tập phỏng vấn Senior Golang — hoàn toàn bằng Tiếng Việt
> **Đối tượng**: Người mới bắt đầu, chưa biết gì về backend

---

## Mục Lục

1. [§1. Tại Sao Biến Quan Trọng?](#1-tại-sao-biến-quan-trọng)
2. [§2. Bộ Nhớ & Type — Nền Tảng Cốt Lõi](#2-bộ-nhớ--type--nền-tảng-cốt-lõi)
3. [§3. Khai Báo Biến — Zero Value với `var`](#3-khai-báo-biến--zero-value-với-var)
4. [§4. Short Variable Declaration — Toán Tử `:=`](#4-short-variable-declaration--toán-tử-)
5. [§5. Type Conversion — Chuyển Đổi Kiểu Dữ Liệu](#5-type-conversion--chuyển-đổi-kiểu-dữ-liệu)
6. [§6. Bộ Nhớ Bên Trong — Biến Được Lưu Thế Nào?](#6-bộ-nhớ-bên-trong--biến-được-lưu-thế-nào)
7. [§7. IEEE-754: Số Thực Dấu Phẩy Động](#7-ieee-754-số-thực-dấu-phẩy-động)
8. [§8. Quy Tắc Đặt Tên Biến](#8-quy-tắc-đặt-tên-biến)
9. [§9. Bài Tập Thực Hành](#9-bài-tập-thực-hành)
10. [§10. Tổng Kết & Câu Hỏi Phỏng Vấn](#10-tổng-kết--câu-hỏi-phỏng-vấn)

---

## §1. Tại Sao Biến Quan Trọng?

### Triết lý cốt lõi

```
╔═══════════════════════════════════════════════════════════════════════╗
║                                                                       ║
║  "The purpose of all programs and all parts of those programs        ║
║   is to TRANSFORM DATA from one form to the other."                  ║
║                                                                       ║
║                                          — William Kennedy            ║
║                                                                       ║
║  "If you don't understand the DATA, you don't understand             ║
║   the PROBLEM."                                                       ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  MỌI CHƯƠNG TRÌNH = BIẾN ĐỔI DATA                          │
    │                                                              │
    │    Input Data ──→ [ CHƯƠNG TRÌNH ] ──→ Output Data          │
    │                                                              │
    │  Chương trình làm 3 việc chính với bộ nhớ:                 │
    │                                                              │
    │    1. ALLOCATE (cấp phát)  → xin vùng nhớ mới             │
    │    2. READ (đọc)           → lấy giá trị từ bộ nhớ        │
    │    3. WRITE (ghi)          → thay đổi giá trị trong bộ nhớ│
    │                                                              │
    │  BIẾN = cách bạn đặt TÊN cho vùng nhớ!                    │
    │  BIẾN = cầu nối giữa BẠN và BỘ NHỚ!                     │
    │                                                              │
    │    var age int = 25                                          │
    │         │   │    │                                           │
    │         │   │    └─ Giá trị: 25                             │
    │         │   └────── Kiểu: int (bao nhiêu bytes, cách đọc) │
    │         └────────── Tên: age (để bạn gọi nó)              │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Tại sao phải hiểu Type?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  "Understanding TYPE is crucial to writing good code        │
    │   and understanding code."                                   │
    │                                                              │
    │  TYPE cho biết 2 điều:                                      │
    │                                                              │
    │  ┌───────────────────┬──────────────────────────────────┐   │
    │  │ 1. KÍCH THƯỚC     │ Biến chiếm bao nhiêu bytes      │   │
    │  │    (Size)          │ trong bộ nhớ?                    │   │
    │  ├───────────────────┼──────────────────────────────────┤   │
    │  │ 2. Ý NGHĨA       │ Các bytes đó ĐẠI DIỆN cho      │   │
    │  │    (Representation)│ cái gì? (số, chữ, đúng/sai?)   │   │
    │  └───────────────────┴──────────────────────────────────┘   │
    │                                                              │
    │  VÍ DỤ: Cùng 1 byte = 01000001                             │
    │  → Nếu kiểu int8:  giá trị = 65                           │
    │  → Nếu kiểu byte:  giá trị = 'A' (ký tự ASCII)           │
    │  → CÙNG DATA, KHÁC TYPE = KHÁC Ý NGHĨA!                  │
    │                                                              │
    │  → Go là ngôn ngữ TYPE SAFE!                               │
    │  → Compiler KHÔNG cho phép dùng biến sai kiểu!            │
    │  → Điều này NGĂN CHẶN bugs ngay từ lúc compile!           │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §2. Bộ Nhớ & Type — Nền Tảng Cốt Lõi

### Built-in Types của Go

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    CÁC KIỂU DỮ LIỆU CƠ BẢN CỦA GO                  ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────┐     ║
║  │ BOOLEAN (Luận lý)                                           │     ║
║  │ ┌──────┬────────┬────────────────────────────────────────┐  │     ║
║  │ │ Kiểu │ Size   │ Giải thích                             │  │     ║
║  │ ├──────┼────────┼────────────────────────────────────────┤  │     ║
║  │ │ bool │ 1 byte │ true hoặc false                       │  │     ║
║  │ └──────┴────────┴────────────────────────────────────────┘  │     ║
║  └─────────────────────────────────────────────────────────────┘     ║
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────┐     ║
║  │ INTEGER (Số nguyên)                                         │     ║
║  │ ┌────────┬────────┬──────────────────────────────────────┐  │     ║
║  │ │ Kiểu  │ Size   │ Phạm vi                              │  │     ║
║  │ ├────────┼────────┼──────────────────────────────────────┤  │     ║
║  │ │ int8   │ 1 byte │ -128 → 127                          │  │     ║
║  │ │ int16  │ 2 bytes│ -32,768 → 32,767                    │  │     ║
║  │ │ int32  │ 4 bytes│ -2.1 tỷ → 2.1 tỷ                  │  │     ║
║  │ │ int64  │ 8 bytes│ -9.2 × 10^18 → 9.2 × 10^18        │  │     ║
║  │ │ int    │ phụ    │ = int64 trên máy 64-bit             │  │     ║
║  │ │        │ thuộc  │ = int32 trên máy 32-bit             │  │     ║
║  │ ├────────┼────────┼──────────────────────────────────────┤  │     ║
║  │ │ uint8  │ 1 byte │ 0 → 255 (không âm)                 │  │     ║
║  │ │ uint16 │ 2 bytes│ 0 → 65,535                          │  │     ║
║  │ │ uint32 │ 4 bytes│ 0 → 4.2 tỷ                         │  │     ║
║  │ │ uint64 │ 8 bytes│ 0 → 18.4 × 10^18                   │  │     ║
║  │ │ byte   │ 1 byte │ = uint8 (alias)                      │  │     ║
║  │ └────────┴────────┴──────────────────────────────────────┘  │     ║
║  └─────────────────────────────────────────────────────────────┘     ║
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────┐     ║
║  │ FLOATING POINT (Số thực dấu phẩy động)                    │     ║
║  │ ┌─────────┬────────┬─────────────────────────────────────┐  │     ║
║  │ │ Kiểu   │ Size   │ Độ chính xác                       │  │     ║
║  │ ├─────────┼────────┼─────────────────────────────────────┤  │     ║
║  │ │ float32 │ 4 bytes│ ~7 chữ số thập phân               │  │     ║
║  │ │ float64 │ 8 bytes│ ~15 chữ số thập phân (mặc định!)  │  │     ║
║  │ └─────────┴────────┴─────────────────────────────────────┘  │     ║
║  └─────────────────────────────────────────────────────────────┘     ║
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────┐     ║
║  │ STRING & POINTER                                            │     ║
║  │ ┌─────────┬──────────┬───────────────────────────────────┐  │     ║
║  │ │ Kiểu   │ Size     │ Giải thích                        │  │     ║
║  │ ├─────────┼──────────┼───────────────────────────────────┤  │     ║
║  │ │ string  │ 16 bytes │ header: pointer (8) + len (8)    │  │     ║
║  │ │ pointer │ 8 bytes  │ địa chỉ bộ nhớ (trên 64-bit)   │  │     ║
║  │ └─────────┴──────────┴───────────────────────────────────┘  │     ║
║  └─────────────────────────────────────────────────────────────┘     ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Sơ đồ bộ nhớ — Kích thước từng type

```
    1 ô = 1 byte

    bool (1 byte):
    ┌────┐
    │ 00 │  ← false (zero value)
    └────┘

    int8 (1 byte):
    ┌────┐
    │ 00 │  ← 0 (zero value)
    └────┘

    int16 (2 bytes):
    ┌────┬────┐
    │ 00 │ 00 │  ← 0
    └────┴────┘

    int32 / float32 (4 bytes):
    ┌────┬────┬────┬────┐
    │ 00 │ 00 │ 00 │ 00 │  ← 0
    └────┴────┴────┴────┘

    int64 / float64 (8 bytes):
    ┌────┬────┬────┬────┬────┬────┬────┬────┐
    │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │  ← 0
    └────┴────┴────┴────┴────┴────┴────┴────┘

    string (16 bytes = pointer + length):
    ┌────┬────┬────┬────┬────┬────┬────┬────┐ ┌────┬────┬────┬────┬────┬────┬────┬────┐
    │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │
    └────┴────┴────┴────┴────┴────┴────┴────┘ └────┴────┴────┴────┴────┴────┴────┴────┘
     ▲ pointer = nil (empty)                    ▲ length = 0
```

---

## §3. Khai Báo Biến — Zero Value với `var`

### Zero Value là gì?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ZERO VALUE = giá trị MẶC ĐỊNH khi KHÔNG gán giá trị      │
    │                                                              │
    │  Go đảm bảo: MỌI biến LUÔN được khởi tạo!                │
    │  → KHÔNG BAO GIỜ có biến "rác" (uninitialized)!           │
    │  → Đây là tính năng AN TOÀN của Go!                        │
    │                                                              │
    │  ┌─────────────────┬────────────────────────────────────┐   │
    │  │ Kiểu            │ Zero Value                         │   │
    │  ├─────────────────┼────────────────────────────────────┤   │
    │  │ bool             │ false                              │   │
    │  │ int, int8, ...   │ 0                                  │   │
    │  │ float32, float64│ 0.0                                │   │
    │  │ complex64, ...   │ 0i                                 │   │
    │  │ string           │ "" (chuỗi rỗng)                  │   │
    │  │ pointer          │ nil                                │   │
    │  │ slice, map, chan │ nil                                │   │
    │  │ interface        │ nil                                │   │
    │  └─────────────────┴────────────────────────────────────┘   │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Code ví dụ — Khai báo Zero Value

```go
package main

import "fmt"

func main() {
    // Khai báo biến với Zero Value → dùng keyword "var"
    var a int
    var b string
    var c float64
    var d bool

    fmt.Printf("var a int     \t %T [%v]\n", a, a)
    fmt.Printf("var b string  \t %T [%v]\n", b, b)
    fmt.Printf("var c float64 \t %T [%v]\n", c, c)
    fmt.Printf("var d bool    \t %T [%v]\n", d, d)
}
```

**Output:**

```
var a int      int     [0]
var b string   string  []
var c float64  float64 [0]
var d bool     bool    [false]
```

### Sơ đồ bộ nhớ — Zero Value

```
    Sau khi khai báo: var a int; var b string; var c float64; var d bool

    STACK FRAME CỦA main():

    ┌────────────────────────────────────────────────────────────┐
    │                                                            │
    │  a (int = int64 trên 64-bit):                             │
    │  ┌────┬────┬────┬────┬────┬────┬────┬────┐               │
    │  │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ = 0          │
    │  └────┴────┴────┴────┴────┴────┴────┴────┘               │
    │                                                            │
    │  b (string = pointer + length):                           │
    │  ┌────┬────┬────┬────┬────┬────┬────┬────┐ ptr = nil     │
    │  │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │              │
    │  ├────┼────┼────┼────┼────┼────┼────┼────┤               │
    │  │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ len = 0      │
    │  └────┴────┴────┴────┴────┴────┴────┴────┘ = ""          │
    │                                                            │
    │  c (float64):                                              │
    │  ┌────┬────┬────┬────┬────┬────┬────┬────┐               │
    │  │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ 00 │ = 0.0        │
    │  └────┴────┴────┴────┴────┴────┴────┴────┘               │
    │                                                            │
    │  d (bool):                                                 │
    │  ┌────┐                                                    │
    │  │ 00 │ = false                                            │
    │  └────┘                                                    │
    │                                                            │
    └────────────────────────────────────────────────────────────┘

    → TẤT CẢ bytes đều = 00!
    → Go khởi tạo TOÀN BỘ bộ nhớ về 0!
    → AN TOÀN! Không có dữ liệu rác!
```

### QUY TẮC: Khi nào dùng `var`?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ★ QUY TẮC WILLIAM KENNEDY:                                │
    │                                                              │
    │  Dùng "var" khi muốn biến = ZERO VALUE!                    │
    │                                                              │
    │  var count int      ← count = 0     ✅ đúng!               │
    │  var name string    ← name = ""     ✅ đúng!               │
    │  var active bool    ← active = false ✅ đúng!               │
    │                                                              │
    │  KHÔNG dùng:                                                │
    │  var count int = 0  ← THỪA! ❌ (0 đã là zero value!)     │
    │  count := 0         ← OK nhưng var rõ ý đồ hơn!          │
    │                                                              │
    │  "var" nói cho người đọc code:                              │
    │  "Tôi MUỐN biến này bắt đầu từ zero value!"               │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §4. Short Variable Declaration — Toán Tử `:=`

### Khai báo và khởi tạo cùng lúc

```go
func main() {
    // Dùng := khi KHAI BÁO + GÁN GIÁ TRỊ cùng lúc!
    aa := 10          // Go suy luận kiểu: int
    bb := "hello"     // Go suy luận kiểu: string
    cc := 3.14159     // Go suy luận kiểu: float64
    dd := true        // Go suy luận kiểu: bool

    fmt.Printf("aa := 10      \t %T [%v]\n", aa, aa)
    fmt.Printf("bb := \"hello\" \t %T [%v]\n", bb, bb)
    fmt.Printf("cc := 3.14159 \t %T [%v]\n", cc, cc)
    fmt.Printf("dd := true    \t %T [%v]\n", dd, dd)
}
```

**Output:**

```
aa := 10       int     [10]
bb := "hello"  string  [hello]
cc := 3.14159  float64 [3.14159]
dd := true     bool    [true]
```

### Go suy luận kiểu (Type Inference) như thế nào?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ":=" = SHORT VARIABLE DECLARATION OPERATOR                  │
    │                                                              │
    │  Cho phép Go TỰ ĐỘNG suy luận kiểu từ giá trị!            │
    │                                                              │
    │  ┌──────────────────┬──────────────┬───────────────────┐    │
    │  │ Code              │ Go suy luận │ Tương đương       │    │
    │  ├──────────────────┼──────────────┼───────────────────┤    │
    │  │ x := 10           │ int          │ var x int = 10   │    │
    │  │ x := 3.14         │ float64      │ var x float64    │    │
    │  │ x := "hi"         │ string       │ var x string     │    │
    │  │ x := true         │ bool         │ var x bool       │    │
    │  │ x := 'A'          │ int32 (rune) │ var x rune       │    │
    │  └──────────────────┴──────────────┴───────────────────┘    │
    │                                                              │
    │  ★ CHÚ Ý:                                                   │
    │  → Số nguyên mặc định = int (KHÔNG phải int32!)            │
    │  → Số thực mặc định = float64 (KHÔNG phải float32!)       │
    │  → Ký tự rune mặc định = int32                             │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### QUY TẮC: Khi nào dùng `:=`?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ★ QUY TẮC WILLIAM KENNEDY:                                │
    │                                                              │
    │  Dùng ":=" khi KHAI BÁO + KHỞI TẠO với giá trị cụ thể!  │
    │                                                              │
    │  count := 10        ← KHAI BÁO + gán 10  ✅                │
    │  name := "Alice"    ← KHAI BÁO + gán     ✅                │
    │                                                              │
    │  KHÔNG dùng:                                                │
    │  count := 0         ← 0 là zero value → dùng var! ❌      │
    │  name := ""         ← "" là zero value → dùng var! ❌     │
    │                                                              │
    │  TÓM TẮT:                                                   │
    │  ┌────────────────┬───────────────────────────────────┐     │
    │  │ Tình huống     │ Dùng gì?                          │     │
    │  ├────────────────┼───────────────────────────────────┤     │
    │  │ Zero value     │ var x int                         │     │
    │  │ Có giá trị    │ x := 10                           │     │
    │  │ Cần chỉ định  │ x := int32(10)                   │     │
    │  │ kiểu cụ thể  │ (dùng := + type conversion)      │     │
    │  └────────────────┴───────────────────────────────────┘     │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §5. Type Conversion — Chuyển Đổi Kiểu Dữ Liệu

### Go KHÔNG có Type Casting — chỉ có Type Conversion!

```go
func main() {
    // Chỉ định kiểu cụ thể khi cần
    aaa := int32(10)

    fmt.Printf("aaa := int32(10) %T [%v]\n", aaa, aaa)
    // Output: aaa := int32(10) int32 [10]
}
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  TYPE CONVERSION vs TYPE CASTING                             │
    │                                                              │
    │  ┌────────────────────────────────────────────────────────┐  │
    │  │ TYPE CASTING (C/C++, Java):                            │  │
    │  │ → Nói compiler: "Giả vờ data này là kiểu khác!"     │  │
    │  │ → KHÔNG copy data! Chỉ đổi cách NHÌN!               │  │
    │  │ → NGUY HIỂM! Có thể đọc sai data!                  │  │
    │  │ → (int)3.14 = 3 (mất .14!)                           │  │
    │  └────────────────────────────────────────────────────────┘  │
    │                                                              │
    │  ┌────────────────────────────────────────────────────────┐  │
    │  │ TYPE CONVERSION (Go):                                  │  │
    │  │ → TẠO BẢN SAO MỚI với kiểu mới!                    │  │
    │  │ → CÓ copy data! + chuyển đổi giá trị!              │  │
    │  │ → AN TOÀN! Compiler kiểm tra tương thích!           │  │
    │  │ → int32(10) = tạo int32 mới, gán giá trị 10       │  │
    │  └────────────────────────────────────────────────────────┘  │
    │                                                              │
    │  VÍ DỤ:                                                     │
    │                                                              │
    │  a := 10            // int (8 bytes trên 64-bit)           │
    │  b := int32(a)      // int32 (4 bytes) ← CONVERSION!      │
    │                                                              │
    │  BỘ NHỚ:                                                    │
    │  a (int64):   [00][00][00][00][00][00][00][0A] = 10        │
    │  b (int32):   [00][00][00][0A]                   = 10      │
    │                ↑ BẢN SAO MỚI! Kích thước KHÁC!             │
    │                                                              │
    │  ★ Go KHÔNG TỰ ĐỘNG chuyển đổi kiểu!                     │
    │  var x int32 = 10                                           │
    │  var y int64 = x    ← COMPILE ERROR! ❌                    │
    │  var y int64 = int64(x)  ← OK! Phải convert rõ ràng ✅   │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Tại sao Go bắt buộc explicit conversion?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  TYPE SAFETY = NGĂN CHẶN BUGS TỪ LÚC COMPILE!             │
    │                                                              │
    │  C/C++ cho phép:                                             │
    │  int a = 100000;                                             │
    │  int8_t b = a;  // b = ? → OVERFLOW! b = -96 (sai!)      │
    │  → KHÔNG lỗi! Âm thầm mất data! → BUG!                   │
    │                                                              │
    │  Go bắt buộc:                                               │
    │  a := 100000                                                │
    │  b := int8(a)   // Go cho phép, NHƯNG bạn PHẢI viết rõ!  │
    │  → Bạn BIẾT bạn đang conversion!                          │
    │  → Bạn CHỊU TRÁCH NHIỆM cho overflow!                     │
    │  → Compiler không "âm thầm" làm gì cả!                   │
    │                                                              │
    │  ★ TRIẾT LÝ GO: "Explicit is better than implicit!"       │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §6. Bộ Nhớ Bên Trong — Biến Được Lưu Thế Nào?

### Stack Frame

```
    ┌──────────────────────────────────────────────────────────────┐
    │  Khi hàm main() chạy, Go tạo STACK FRAME:                 │
    │                                                              │
    │  STACK (từ cao → thấp):                                    │
    │  ┌─────────────────────────────────────────────────┐        │
    │  │ main() Stack Frame                               │        │
    │  │                                                   │        │
    │  │ ┌─────────┬──────────────────────────────────┐   │        │
    │  │ │ Biến   │ Bộ nhớ                           │   │        │
    │  │ ├─────────┼──────────────────────────────────┤   │        │
    │  │ │ a (int) │ [00 00 00 00 00 00 00 0A] = 10  │   │        │
    │  │ │ b (str) │ [ptr → "hello"] [len = 5]       │   │        │
    │  │ │ c (f64) │ [40 09 21 FB 54 44 2D 18] = π  │   │        │
    │  │ │ d (bool)│ [01] = true                      │   │        │
    │  │ └─────────┴──────────────────────────────────┘   │        │
    │  │                                                   │        │
    │  └─────────────────────────────────────────────────┘        │
    │                                                              │
    │  → Mỗi biến = 1 vùng nhớ liên tục trên Stack!            │
    │  → Kích thước phụ thuộc vào TYPE!                         │
    │  → Khi main() kết thúc → Stack Frame bị XÓA!             │
    │  → Tất cả biến local bị giải phóng tự động!             │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### String trong bộ nhớ

```
    bb := "hello"

    STACK:                          HEAP (hoặc read-only memory):
    ┌──────────────────────┐       ┌─────────────────────────┐
    │ bb (string header)   │       │                         │
    │ ┌──────────────────┐ │       │  "hello"                │
    │ │ pointer ──────────┼─┼──→   │  [68][65][6C][6C][6F]  │
    │ │ 0x1234...         │ │       │   h    e    l    l    o │
    │ ├──────────────────┤ │       │                         │
    │ │ length = 5        │ │       └─────────────────────────┘
    │ └──────────────────┘ │
    └──────────────────────┘

    → string header = 16 bytes (pointer 8 + length 8)
    → Data thực tế ("hello") nằm ở nơi khác!
    → string header chỉ TRỎ ĐẾN data!
    → string trong Go là IMMUTABLE (không thể sửa!)
```

---

## §7. IEEE-754: Số Thực Dấu Phẩy Động

### Tại sao 0.1 + 0.2 ≠ 0.3?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  IEEE-754 — CHUẨN BIỂU DIỄN SỐ THỰC TRONG MÁY TÍNH       │
    │                                                              │
    │  Máy tính KHÔNG THỂ biểu diễn chính xác MỌI số thực!     │
    │                                                              │
    │  float64 (8 bytes = 64 bits):                               │
    │  ┌───┬─────────────┬──────────────────────────────────────┐ │
    │  │ 1 │    11 bits   │           52 bits                    │ │
    │  │bit│  exponent    │          mantissa (fraction)         │ │
    │  │ S │ (mũ)        │         (phần thập phân)            │ │
    │  └───┴─────────────┴──────────────────────────────────────┘ │
    │   ↑                                                          │
    │   Sign: 0 = dương, 1 = âm                                  │
    │                                                              │
    │  VÍ DỤ KINH ĐIỂN:                                           │
    │  a := 0.1                                                    │
    │  b := 0.2                                                    │
    │  c := a + b                                                  │
    │  fmt.Println(c)        // 0.30000000000000004 ← KHÔNG = 0.3│
    │  fmt.Println(c == 0.3) // false! ← NGUY HIỂM!             │
    │                                                              │
    │  ★ BÀI HỌC:                                                │
    │  → KHÔNG BAO GIỜ so sánh float bằng "=="!                 │
    │  → Dùng epsilon: math.Abs(a - b) < 1e-9                   │
    │  → Dùng int cho tiền tệ! (đơn vị: cent, không dollar!)  │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §8. Quy Tắc Đặt Tên Biến

### "What's in a name?" — Andrew Gerrand

```
    ┌──────────────────────────────────────────────────────────────┐
    │  QUY TẮC ĐẶT TÊN BIẾN TRONG GO                            │
    │                                                              │
    │  ┌────────────────────────────────────────────────────────┐  │
    │  │ 1. TÊN NGẮN cho scope nhỏ:                            │  │
    │  │                                                        │  │
    │  │   for i := 0; i < 10; i++ { ... }  ← i OK, index OK │  │
    │  │   for _, v := range items { ... }   ← v OK            │  │
    │  │                                                        │  │
    │  │ 2. TÊN DÀI HƠN cho scope lớn:                       │  │
    │  │                                                        │  │
    │  │   var userCount int         ← rõ ràng cho package-level│  │
    │  │   var maxRetryAttempts int  ← rõ ràng cho config      │  │
    │  │                                                        │  │
    │  │ 3. CamelCase, KHÔNG dùng snake_case:                  │  │
    │  │                                                        │  │
    │  │   userName   ✅ (Go style)                             │  │
    │  │   user_name  ❌ (Python/Ruby style)                   │  │
    │  │   UserName   ✅ (exported — bắt đầu chữ HOA)        │  │
    │  │   userName   ✅ (unexported — bắt đầu chữ thường)   │  │
    │  │                                                        │  │
    │  │ 4. Acronyms VIẾT HOA HẾT:                            │  │
    │  │                                                        │  │
    │  │   userID     ✅ (không phải userId)                   │  │
    │  │   httpClient ✅ (không phải httpClient)               │  │
    │  │   xmlParser  ✅                                        │  │
    │  └────────────────────────────────────────────────────────┘  │
    │                                                              │
    │  ★ EXPORTED vs UNEXPORTED:                                  │
    │  → Bắt đầu chữ HOA = EXPORTED (public, ai cũng dùng)    │
    │  → Bắt đầu chữ thường = UNEXPORTED (private, chỉ pkg)  │
    │  → Đây là cách Go thay thế public/private keyword!       │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §9. Bài Tập Thực Hành

### Exercise 1 — Template

```go
package main

// Add imports

func main() {
    // Part A:
    // Khai báo 3 biến zero value: string, int, bool
    // Hiển thị giá trị

    // Khai báo 3 biến với giá trị cụ thể: string, int, bool
    // Hiển thị giá trị

    // Part B:
    // Khai báo biến float32, khởi tạo bằng cách convert Pi (3.14)
    // Hiển thị giá trị
}
```

### Exercise 1 — Đáp án

```go
package main

import "fmt"

func main() {
    // Part A — Zero values (dùng var!)
    var age int
    var name string
    var legal bool

    fmt.Println(age)        // 0
    fmt.Println(name)       // "" (empty)
    fmt.Println(legal)      // false

    // Part A — Literal values (dùng :=!)
    month := 10
    dayOfWeek := "Tuesday"
    happy := true

    fmt.Println(month)      // 10
    fmt.Println(dayOfWeek)  // Tuesday
    fmt.Println(happy)      // true

    // Part B — Type conversion
    pi := float32(3.14)

    fmt.Printf("%T [%v]\n", pi, pi)  // float32 [3.14]
}
```

### Giải thích từng dòng

```
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  var age int       → zero value = 0                         │
    │  var name string   → zero value = "" (chuỗi rỗng)         │
    │  var legal bool    → zero value = false                     │
    │                                                              │
    │  ★ Dùng "var" vì chúng ta MUỐN zero value!                │
    │                                                              │
    │  month := 10                → int, gán 10                   │
    │  dayOfWeek := "Tuesday"     → string, gán "Tuesday"        │
    │  happy := true              → bool, gán true                │
    │                                                              │
    │  ★ Dùng ":=" vì chúng ta CÓ GIÁ TRỊ CỤ THỂ!            │
    │                                                              │
    │  pi := float32(3.14)                                        │
    │        ↑                                                     │
    │        TYPE CONVERSION!                                      │
    │        3.14 mặc định = float64                              │
    │        float32(3.14) = chuyển thành float32 (4 bytes)      │
    │        → MẤT ĐỘ CHÍNH XÁC (float64=15 digits, float32=7) │
    │                                                              │
    │  %T = in ra TYPE của biến                                   │
    │  %v = in ra VALUE của biến                                  │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §10. Tổng Kết & Câu Hỏi Phỏng Vấn

### Sơ đồ tổng kết

```
╔═══════════════════════════════════════════════════════════════════════╗
║                     TỔNG KẾT: GO VARIABLES                           ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────┐     ║
║  │ 1. MỌI chương trình = BIẾN ĐỔI DATA                       │     ║
║  │ 2. Code = allocate + read + write bộ nhớ                   │     ║
║  │ 3. BIẾN = cách đặt TÊN cho vùng nhớ                       │     ║
║  │ 4. TYPE = kích thước + ý nghĩa của data                    │     ║
║  │ 5. Go = TYPE SAFE → compiler kiểm tra kiểu!              │     ║
║  └─────────────────────────────────────────────────────────────┘     ║
║                                                                       ║
║  CÁCH KHAI BÁO:                                                      ║
║  ┌───────────────────────────────────────────────────────────┐       ║
║  │                                                           │       ║
║  │  var x int        ← Zero value (x = 0)                  │       ║
║  │  x := 10          ← Khai báo + gán giá trị             │       ║
║  │  x := int32(10)   ← Khai báo + chỉ định kiểu         │       ║
║  │                                                           │       ║
║  │  RULE:                                                    │       ║
║  │  → Zero value? → dùng "var"!                            │       ║
║  │  → Có giá trị? → dùng ":="!                            │       ║
║  │                                                           │       ║
║  └───────────────────────────────────────────────────────────┘       ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Câu hỏi phỏng vấn Senior Golang

```
╔═══════════════════════════════════════════════════════════════════════╗
║                   CÂU HỎI PHỎNG VẤN SENIOR GOLANG                    ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  Q1: Zero value trong Go là gì? Tại sao quan trọng?                 ║
║  A1: Zero value = giá trị mặc định khi không gán giá trị.          ║
║      Go khởi tạo MỌI biến về zero value → AN TOÀN!                ║
║      Không có biến "rác" (uninitialized) như C/C++.                 ║
║      bool=false, int=0, float=0.0, string="", pointer=nil.         ║
║                                                                       ║
║  Q2: Khi nào dùng "var" vs ":="?                                    ║
║  A2: var → khi muốn biến = zero value (ý đồ rõ ràng!)            ║
║      := → khi khai báo + gán giá trị cụ thể.                     ║
║      var count int (= 0 by design) vs count := 10.                  ║
║                                                                       ║
║  Q3: Go có type casting không?                                       ║
║  A3: KHÔNG! Go chỉ có TYPE CONVERSION.                              ║
║      Casting = reinterpret bits (nguy hiểm).                        ║
║      Conversion = tạo bản sao mới với kiểu mới (an toàn).        ║
║      Go KHÔNG tự động convert kiểu → phải explicit!               ║
║                                                                       ║
║  Q4: int và int32 có giống nhau không?                               ║
║  A4: KHÔNG! Chúng là 2 kiểu KHÁC NHAU trong Go!                   ║
║      int = phụ thuộc platform (32 hoặc 64 bit).                    ║
║      int32 = luôn 4 bytes. Không thể gán trực tiếp!               ║
║      var a int32 = 10; var b int = a → COMPILE ERROR!              ║
║      Phải: var b int = int(a)                                        ║
║                                                                       ║
║  Q5: Tại sao không nên so sánh float bằng "=="?                     ║
║  A5: IEEE-754 không thể biểu diễn chính xác mọi số thực.          ║
║      0.1 + 0.2 = 0.30000000000000004 (≠ 0.3!)                      ║
║      Dùng epsilon comparison: math.Abs(a-b) < 1e-9.                ║
║      Dùng int cho tiền tệ (cents thay vì dollars).                ║
║                                                                       ║
║  Q6: string trong Go được lưu thế nào trong bộ nhớ?                ║
║  A6: string = struct header 16 bytes (pointer 8 + length 8).       ║
║      Data thực tế nằm ở nơi khác (read-only memory).              ║
║      string là IMMUTABLE! Không thể sửa từng byte.                ║
║      Khi "cộng" string → tạo string MỚI hoàn toàn!              ║
║                                                                       ║
║  Q7: Exported vs Unexported trong Go?                                ║
║  A7: Chữ HOA đầu = Exported (public, ai cũng access).              ║
║      Chữ thường đầu = Unexported (private, chỉ pkg hiện tại).    ║
║      Go KHÔNG CÓ keyword public/private/protected!                 ║
║      Cách đặt tên = cơ chế kiểm soát access!                     ║
║                                                                       ║
║  Q8: Go dùng "var x = 10" hay "x := 10"? Khác gì?                  ║
║  A8: Cả 2 tương đương về kết quả! Nhưng:                          ║
║      "var x = 10" → dùng NGOÀI function (package level)           ║
║      "x := 10" → CHỈ dùng TRONG function!                         ║
║      ":=" KHÔNG dùng được ở package level!                         ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

## §S1. Kiến Thức Bổ Sung — Two's Complement & Các Kiểu Đặc Biệt

> Nguồn: [Go Language Specification — Numeric Types](http://golang.org/ref/spec#Numeric_types)

### Two's Complement — Cách máy tính biểu diễn số âm

```
    ┌──────────────────────────────────────────────────────────────┐
    │  TWO'S COMPLEMENT — BIỂU DIỄN SỐ ÂM TRONG MÁY TÍNH       │
    │                                                              │
    │  Go spec: "The value of an n-bit integer is n bits wide     │
    │  and represented using two's complement arithmetic."         │
    │                                                              │
    │  Ví dụ int8 (8 bits, phạm vi -128 → 127):                 │
    │                                                              │
    │  ┌──────────────┬──────────────────────────────────────────┐ │
    │  │ Giá trị     │ Binary (8 bits)                         │ │
    │  ├──────────────┼──────────────────────────────────────────┤ │
    │  │     0         │ 00000000                                │ │
    │  │     1         │ 00000001                                │ │
    │  │     2         │ 00000010                                │ │
    │  │   127         │ 01111111  ← GIÁ TRỊ DƯƠNG LỚN NHẤT  │ │
    │  │  -128         │ 10000000  ← GIÁ TRỊ ÂM NHỎ NHẤT     │ │
    │  │    -1         │ 11111111                                │ │
    │  │    -2         │ 11111110                                │ │
    │  └──────────────┴──────────────────────────────────────────┘ │
    │                                                              │
    │  CÁCH TÍNH SỐ ÂM (Two's Complement):                       │
    │  Bước 1: Lấy giá trị dương, viết binary                   │
    │  Bước 2: Đảo tất cả bits (0→1, 1→0) = One's Complement   │
    │  Bước 3: Cộng thêm 1                                       │
    │                                                              │
    │  Ví dụ: -5 từ 5                                             │
    │    5       = 00000101                                        │
    │    Đảo    = 11111010  (one's complement)                   │
    │    +1      = 11111011  = -5 ✅                              │
    │                                                              │
    │  ★ TẠI SAO DÙNG TWO'S COMPLEMENT?                          │
    │  → Phép cộng số âm + dương CÙNG mạch phần cứng!          │
    │  → Chỉ có 1 số 0 (không có +0 và -0!)                    │
    │  → Overflow tự nhiên "wrap around"                         │
    │                                                              │
    │  ★ OVERFLOW TRONG GO:                                       │
    │  var x int8 = 127                                           │
    │  x++  // x = -128! ← WRAP AROUND! Không lỗi!             │
    │  → Go KHÔNG panic khi integer overflow!                    │
    │  → Đây là hành vi DEFINED (không phải undefined!)         │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Các kiểu đặc biệt: `uintptr`, `rune`, `byte`

```
    ┌──────────────────────────────────────────────────────────────┐
    │  CÁC KIỂU ĐẶC BIỆT TỪ GO SPEC                            │
    │                                                              │
    │  ┌──────────────────────────────────────────────────────────┐│
    │  │ uintptr — Lưu trữ con trỏ dưới dạng số nguyên        ││
    │  │                                                          ││
    │  │ → "an unsigned integer large enough to store the        ││
    │  │    uninterpreted bits of a pointer value"                ││
    │  │ → Dùng trong unsafe operations và CGo                   ││
    │  │ → KHÔNG BAO GIỜ dùng trực tiếp trừ khi bạn biết      ││
    │  │   chính xác mình đang làm gì!                          ││
    │  │                                                          ││
    │  │ ptr := uintptr(unsafe.Pointer(&x))                      ││
    │  │ // Chuyển pointer → integer (để tính toán pointer)     ││
    │  └──────────────────────────────────────────────────────────┘│
    │                                                              │
    │  ┌──────────────────────────────────────────────────────────┐│
    │  │ byte = uint8 (ALIAS, không phải kiểu mới!)             ││
    │  │                                                          ││
    │  │ → byte và uint8 là CÙNG KIỂU!                          ││
    │  │ → Dùng byte khi xử lý raw data, binary protocols       ││
    │  │ → Dùng uint8 khi xử lý số nguyên nhỏ                  ││
    │  │                                                          ││
    │  │ var b byte = 'A'    // = 65 (ASCII code)                ││
    │  │ var u uint8 = b     // OK! Cùng kiểu!                  ││
    │  └──────────────────────────────────────────────────────────┘│
    │                                                              │
    │  ┌──────────────────────────────────────────────────────────┐│
    │  │ rune = int32 (ALIAS, không phải kiểu mới!)             ││
    │  │                                                          ││
    │  │ → rune và int32 là CÙNG KIỂU!                          ││
    │  │ → rune = 1 Unicode code point                           ││
    │  │ → Dùng rune khi xử lý ký tự Unicode                   ││
    │  │ → String Go = chuỗi bytes, KHÔNG phải chuỗi runes!   ││
    │  │                                                          ││
    │  │ var r rune = '世'   // = 19990 (Unicode code point)    ││
    │  │ // '世' chiếm 3 bytes trong UTF-8, nhưng 1 rune!      ││
    │  │                                                          ││
    │  │ s := "Hello 世界"                                       ││
    │  │ len(s)           = 12   (bytes!)                         ││
    │  │ len([]rune(s))   = 8    (characters!)                   ││
    │  └──────────────────────────────────────────────────────────┘│
    │                                                              │
    │  ★ TẤT CẢ kiểu số trong Go là DEFINED TYPES và DISTINCT! │
    │  → int32 và int KHÁC NHAU dù cùng kích thước!            │
    │  → CHỈ byte/uint8 và rune/int32 là ALIAS (cùng kiểu!)    │
    │  → Mọi trường hợp khác → phải explicit conversion!       │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §S2. Kiến Thức Bổ Sung — Khai Báo Nâng Cao

> Nguồn: [Effective Go — Initialization](https://golang.org/doc/effective_go.html#variables), [Go Spec — Short Variable Declarations](http://golang.org/ref/spec#Short_variable_declarations)

### Redeclaration trong `:=` — Bẫy phổ biến!

```
    ┌──────────────────────────────────────────────────────────────┐
    │  REDECLARATION — ĐẶC TÍNH ĐỘC ĐÁO CỦA ":="!             │
    │                                                              │
    │  Go Spec: ":= may REDECLARE variables provided they were   │
    │  originally declared earlier in the same block, and at      │
    │  least ONE of the non-blank variables is NEW."               │
    │                                                              │
    │  ┌─────────────────────────────────────────────────────────┐ │
    │  │ // Ví dụ redeclaration HỢP LỆ:                        │ │
    │  │ field1, offset := nextField(str, 0)                    │ │
    │  │ field2, offset := nextField(str, offset)               │ │
    │  │ //              ↑ offset REDECLARED! (gán lại)        │ │
    │  │ //       ↑ field2 là NEW! → OK!                       │ │
    │  │                                                         │ │
    │  │ // offset không tạo biến MỚI, chỉ GÁN LẠI GIÁ TRỊ!│ │
    │  └─────────────────────────────────────────────────────────┘ │
    │                                                              │
    │  ┌─────────────────────────────────────────────────────────┐ │
    │  │ // Ví dụ redeclaration KHÔNG HỢP LỆ:                  │ │
    │  │ x, y, x := 1, 2, 3   // ❌ ILLEGAL! x lặp lại!      │ │
    │  └─────────────────────────────────────────────────────────┘ │
    │                                                              │
    │  ⚠️ BẪY VARIABLE SHADOWING:                                │
    │                                                              │
    │  var err error                                               │
    │  if true {                                                   │
    │      val, err := someFunc()  // ← TẠO err MỚI trong if! │
    │      // err ở đây KHÁC với err bên ngoài!               │
    │      // err bên ngoài VẪN = nil! → BUG NGUY HIỂM!      │
    │  }                                                           │
    │  // err vẫn = nil ở đây! ← KHÔNG THAY ĐỔI!             │
    │                                                              │
    │  ★ CÁCH FIX:                                                │
    │  var err error                                               │
    │  var val int                                                 │
    │  if true {                                                   │
    │      val, err = someFunc()  // ← dùng = thay vì :=       │
    │  }                                                           │
    │  // err thay đổi đúng!                                     │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Package-level `var` blocks — Effective Go

```go
// Effective Go: Variables can be initialized just like constants
// but the initializer can be a GENERAL EXPRESSION computed at RUN TIME.

var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  VAR BLOCKS — NHÓM BIẾN CÓ LIÊN QUAN                       │
    │                                                              │
    │  ★ Dùng var ( ... ) để nhóm biến có cùng ngữ cảnh!       │
    │  ★ Khác constants: biến có thể dùng BIỂU THỨC run-time!  │
    │  ★ os.Getenv() chạy lúc runtime, KHÔNG phải compile-time! │
    │                                                              │
    │  ★ ":=" KHÔNG dùng được ở package level!                   │
    │  home := os.Getenv("HOME")  ← ❌ COMPILE ERROR!          │
    │  var home = os.Getenv("HOME") ← ✅ OK!                    │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Constants & `iota` — Enumerated Constants

```go
// Effective Go: enumerated constants using iota enumerator

type ByteSize float64

const (
    _  = iota              // ignore first value (0)
    KB ByteSize = 1 << (10 * iota)  // 1 << 10 = 1024
    MB                     // 1 << 20 = 1,048,576
    GB                     // 1 << 30
    TB                     // 1 << 40
    PB                     // 1 << 50
    EB                     // 1 << 60
)

func (b ByteSize) String() string {
    switch {
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    // ... tiếp tục cho TB, GB, MB, KB
    }
    return fmt.Sprintf("%.2fB", b)
}
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  IOTA — BỘ ĐẾM TỰ ĐỘNG TRONG CONST BLOCK                 │
    │                                                              │
    │  ★ iota bắt đầu từ 0, tự tăng 1 mỗi dòng               │
    │  ★ Biểu thức tự REPEATED cho các dòng sau!               │
    │  ★ Constants tạo lúc COMPILE TIME (không phải runtime!)   │
    │  ★ Chỉ có thể là: numbers, characters, strings, booleans  │
    │                                                              │
    │  const block:                                                │
    │  ┌────────┬────────┬──────────────────────────┐             │
    │  │ Dòng  │ iota   │ Giá trị                  │             │
    │  ├────────┼────────┼──────────────────────────┤             │
    │  │ _ =    │ 0      │ bỏ qua (blank identifier)│             │
    │  │ KB =   │ 1      │ 1 << 10 = 1,024          │             │
    │  │ MB =   │ 2      │ 1 << 20 = 1,048,576      │             │
    │  │ GB =   │ 3      │ 1 << 30 = 1,073,741,824  │             │
    │  │ TB =   │ 4      │ 1 << 40 = 1.09 × 10^12   │             │
    │  │ PB =   │ 5      │ 1 << 50 = 1.12 × 10^15   │             │
    │  │ EB =   │ 6      │ 1 << 60 = 1.15 × 10^18   │             │
    │  └────────┴────────┴──────────────────────────┘             │
    │                                                              │
    │  ★ ByteSize(1e13) in ra: 9.09TB                            │
    │  → Custom type + String() method = auto formatting!        │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Hàm `init()` — Khởi tạo package

```go
// Effective Go: each source file can define its own init function

func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  init() — THỨ TỰ KHỞI TẠO TRONG GO                        │
    │                                                              │
    │  1. Import packages → KHỞI TẠO các packages imported      │
    │  2. Package-level var → evaluate initializers               │
    │  3. init() functions → chạy SAU TẤT CẢ var declarations   │
    │                                                              │
    │  ┌───────────────────────────────────────────────────────┐   │
    │  │ Imported Packages       │ Đệ quy khởi tạo trước!   │   │
    │  │          ↓               │                            │   │
    │  │ Package-level vars      │ var home = os.Getenv(...)  │   │
    │  │          ↓               │                            │   │
    │  │ init() functions         │ Kiểm tra, sửa, validate  │   │
    │  │          ↓               │                            │   │
    │  │ main() (nếu là main)   │ Bắt đầu chạy!            │   │
    │  └───────────────────────────────────────────────────────┘   │
    │                                                              │
    │  ★ Mỗi file có thể có NHIỀU init() functions!              │
    │  ★ init() KHÔNG có parameters và return values!             │
    │  ★ Dùng để: validate config, setup connections, repair state│
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §S3. Kiến Thức Bổ Sung — Recursive Zero Value

> Nguồn: [Go Spec — The Zero Value](http://golang.org/ref/spec#The_zero_value)

### Zero Value áp dụng ĐỆ QUY cho structs!

```go
type T struct {
    i    int
    f    float64
    next *T
}

t := new(T)   // hoặc: var t T
// t.i    == 0      ✅
// t.f    == 0.0    ✅
// t.next == nil    ✅
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  RECURSIVE ZERO VALUE — THEO GO SPEC                        │
    │                                                              │
    │  "This initialization is done recursively, so for instance  │
    │  each element of an array of structs will have its fields   │
    │  zeroed if no value is specified."                            │
    │                                                              │
    │  Sơ đồ bộ nhớ khi: t := new(T)                             │
    │                                                              │
    │  ┌──────────────────────────────────────┐                   │
    │  │ t (*T) ─────→ T struct:              │                   │
    │  │              ┌────────────────────────┤                   │
    │  │              │ i (int)    = 0         │                   │
    │  │              ├────────────────────────┤                   │
    │  │              │ f (float64)= 0.0       │                   │
    │  │              ├────────────────────────┤                   │
    │  │              │ next (*T)  = nil       │                   │
    │  │              └────────────────────────┘                   │
    │  └──────────────────────────────────────┘                   │
    │                                                              │
    │  Ví dụ mảng structs:                                        │
    │  var arr [3]T                                                │
    │  // arr[0].i == 0, arr[0].f == 0.0, arr[0].next == nil     │
    │  // arr[1].i == 0, arr[1].f == 0.0, arr[1].next == nil     │
    │  // arr[2].i == 0, arr[2].f == 0.0, arr[2].next == nil     │
    │  → TẤT CẢ fields, TẤT CẢ elements đều = zero value!    │
    │                                                              │
    │  ★ BÀI HỌC: Bạn KHÔNG CẦN constructor chỉ để set        │
    │    giá trị về 0/nil/false. Go đã làm sẵn!                 │
    │  ★ Đây là lý do nhiều types trong Go "usable" ngay khi    │
    │    zero value (ví dụ: sync.Mutex, bytes.Buffer...)         │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §S4. Kiến Thức Bổ Sung — Gustavo's IEEE-754 Brain Teaser

> Nguồn: [Gustavo's IEEE-754 Brain Teaser](https://www.ardanlabs.com/blog/2013/08/gustavos-ieee-754-brain-teaser.html) — William Kennedy

### Bài toán: Tại sao số thực "nhìn giống" nhưng "khác nhau"?

```go
package main

import "fmt"

func main() {
    a := 0.1
    b := 0.2
    c := a + b

    fmt.Printf("a = %.20f\n", a)   // 0.10000000000000000555
    fmt.Printf("b = %.20f\n", b)   // 0.20000000000000001110
    fmt.Printf("c = %.20f\n", c)   // 0.30000000000000004441
    fmt.Printf("0.3 = %.20f\n", 0.3) // 0.29999999999999998890

    fmt.Println(c == 0.3)  // false! ← CHÚ Ý!
}
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  GUSTAVO'S IEEE-754 BRAIN TEASER                             │
    │                                                              │
    │  TẠI SAO 0.1 + 0.2 ≠ 0.3?                                  │
    │                                                              │
    │  float64 dùng binary (cơ số 2) để biểu diễn số!           │
    │  → 0.1 trong binary = 0.0001100110011... (LẶP VÔ TẬN!)   │
    │  → Machine PHẢI cắt bớt → mất precision!                  │
    │                                                              │
    │  Phân tích bit-level (IEEE-754 double):                     │
    │                                                              │
    │  0.1 thực tế lưu trong máy:                                │
    │  = 0.1000000000000000055511151231257827021181583404541015625 │
    │                                                              │
    │  0.2 thực tế lưu trong máy:                                │
    │  = 0.2000000000000000111022302462515654042363166809082031250 │
    │                                                              │
    │  0.1 + 0.2 =                                                 │
    │  = 0.3000000000000000444089209850062616169452667236328125000 │
    │                                                              │
    │  0.3 literal thực tế:                                       │
    │  = 0.2999999999999999888977697537484345957636833190917968750 │
    │                                                              │
    │  → 0.1 + 0.2 và 0.3 có CÁC BITS KHÁC NHAU!               │
    │  → Dù in ra giống "0.3" nhưng thực tế KHÁC!               │
    │                                                              │
    │  ┌──────────────────────────────────────────────────────┐    │
    │  │ ★ QUY TẮC VÀNG KHI LÀM VIỆC VỚI FLOAT:            │    │
    │  │                                                      │    │
    │  │ 1. KHÔNG BAO GIỜ dùng == để so sánh float!         │    │
    │  │                                                      │    │
    │  │ 2. Dùng epsilon comparison:                         │    │
    │  │    const epsilon = 1e-9                              │    │
    │  │    if math.Abs(a - b) < epsilon { /* equal */ }     │    │
    │  │                                                      │    │
    │  │ 3. Dùng math/big.Float cho precision cao!           │    │
    │  │                                                      │    │
    │  │ 4. TIỀN TỆ → dùng integer (cents, not dollars!)   │    │
    │  │    price := 299  // $2.99 = 299 cents               │    │
    │  │                                                      │    │
    │  │ 5. float32 chỉ ~7 digits precision!                │    │
    │  │    float64 chỉ ~15 digits precision!               │    │
    │  │    → Sau ngưỡng đó → kết quả SAI!               │    │
    │  └──────────────────────────────────────────────────────┘    │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Sơ đồ IEEE-754: float32 vs float64

```
    float32 (4 bytes = 32 bits):
    ┌───┬──────────┬───────────────────────┐
    │ 1 │  8 bits  │       23 bits         │
    │ S │ exponent │      mantissa         │
    └───┴──────────┴───────────────────────┘
     ↑                ↑
     Sign bit         ~7 chữ số thập phân chính xác

    float64 (8 bytes = 64 bits):
    ┌───┬─────────────┬──────────────────────────────────────┐
    │ 1 │   11 bits   │              52 bits                  │
    │ S │  exponent   │             mantissa                  │
    └───┴─────────────┴──────────────────────────────────────┘
     ↑                     ↑
     Sign bit              ~15 chữ số thập phân chính xác

    Giá trị đặc biệt trong IEEE-754:
    ┌───────────────────────────────────────────────────────────┐
    │  +Inf   = exponent tất cả 1, mantissa tất cả 0         │
    │  -Inf   = sign=1, exponent tất cả 1, mantissa tất cả 0│
    │  NaN    = exponent tất cả 1, mantissa ≠ 0              │
    │  +0/-0  = exponent tất cả 0, mantissa tất cả 0        │
    └───────────────────────────────────────────────────────────┘

    Ví dụ trong Go:
    fmt.Println(math.Inf(1))     // +Inf
    fmt.Println(math.Inf(-1))    // -Inf
    fmt.Println(math.NaN())      // NaN
    fmt.Println(math.IsNaN(x))   // kiểm tra NaN
    fmt.Println(0.0 == -0.0)     // true! (Go coi bằng nhau)
```

---

## §S5. Kiến Thức Bổ Sung — Conversion Rules & String Internals

> Nguồn: [Go Spec — Conversions](http://golang.org/ref/spec#Conversions)

### Quy tắc Conversion đầy đủ từ Go Spec

```
    ┌──────────────────────────────────────────────────────────────┐
    │  KHI NÀO CÓ THỂ CONVERT T(x)?                              │
    │                                                              │
    │  Go Spec cho phép conversion khi:                           │
    │                                                              │
    │  1. x assignable cho T                                      │
    │  2. x và T có CÙNG underlying type                         │
    │  3. x và T đều là integer hoặc floating point              │
    │  4. x và T đều là complex                                  │
    │  5. x là integer/[]byte/[]rune, T là string                │
    │  6. x là string, T là []byte hoặc []rune                  │
    │  7. x là slice, T là array (Go 1.20+)                      │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

```go
// Ví dụ conversion HỢPỆ:
uint(iota)               // iota → uint
float32(2.718281828)      // float64 → float32
complex128(1)             // int → complex128 (= 1+0i)
string('x')              // rune → string (= "x")
string(0x266c)           // int → string (= "♬")
string([]byte{'a'})      // []byte → string

// Ví dụ conversion KHÔNG hợp lệ:
int(1.2)                  // ❌ 1.2 không thể biểu diễn bằng int
string(65.0)             // ❌ 65.0 không phải integer constant
```

### String ↔ []byte ↔ []rune — Quan hệ ba bên

```
    ┌──────────────────────────────────────────────────────────────┐
    │  STRING ↔ BYTE ↔ RUNE                                      │
    │                                                              │
    │  s := "Hello 世界"                                          │
    │                                                              │
    │  string (UTF-8 encoded bytes):                              │
    │  [48][65][6C][6C][6F][20][E4][B8][96][E7][95][8C]          │
    │   H   e   l   l   o   █   ←──世──→  ←──界──→              │
    │  len(s) = 12 bytes                                          │
    │                                                              │
    │  []byte(s) — chuyển thành slice of bytes:                  │
    │  [72, 101, 108, 108, 111, 32, 228, 184, 150, 231, 149, 140]│
    │  len = 12                                                    │
    │                                                              │
    │  []rune(s) — chuyển thành slice of runes (Unicode):        │
    │  [72, 101, 108, 108, 111, 32, 19990, 30028]                │
    │   H    e    l    l    o    █   世     界                     │
    │  len = 8 (8 ký tự Unicode!)                                │
    │                                                              │
    │  ★ QUAN TRỌNG:                                              │
    │  → len(string) = số BYTES, KHÔNG phải số ký tự!          │
    │  → 1 rune có thể = 1, 2, 3, hoặc 4 bytes UTF-8!         │
    │  → '世' = 3 bytes UTF-8 nhưng 1 rune!                     │
    │  → Dùng for range trên string → iterate theo RUNE!        │
    │  → Dùng for i := 0; i < len(s); i++ → iterate theo BYTE! │
    │                                                              │
    │  for i, r := range "Hello 世界" {                           │
    │      fmt.Printf("index=%d rune=%c\n", i, r)               │
    │  }                                                           │
    │  // index=0 rune=H                                          │
    │  // index=1 rune=e                                          │
    │  // ...                                                      │
    │  // index=6 rune=世  ← index nhảy từ 5 → 6 (3 bytes!)   │
    │  // index=9 rune=界  ← index nhảy từ 6 → 9 (3 bytes!)   │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Naming Conventions từ Effective Go — Chi tiết

```
    ┌──────────────────────────────────────────────────────────────┐
    │  NAMING CONVENTIONS — EFFECTIVE GO CHI TIẾT                 │
    │                                                              │
    │  ┌──────────────────────────────────────────────────────┐   │
    │  │ PACKAGE NAMES:                                       │   │
    │  │                                                      │   │
    │  │ → Tên ngắn, lowercase, 1 từ!                       │   │
    │  │ → KHÔNG dùng under_scores hay mixedCaps!           │   │
    │  │ → bufio, fmt, os, sync, http                       │   │
    │  │ → Package name = phần cuối của import path         │   │
    │  │   import "encoding/base64" → dùng base64.xxx()    │   │
    │  └──────────────────────────────────────────────────────┘   │
    │                                                              │
    │  ┌──────────────────────────────────────────────────────┐   │
    │  │ GETTERS — Go KHÔNG có "Get" prefix!                 │   │
    │  │                                                      │   │
    │  │ Java:    obj.GetName()    ← có "Get" prefix         │   │
    │  │ Go:      obj.Name()      ← KHÔNG có "Get"! ✅     │   │
    │  │ Setter:  obj.SetName(v)  ← có "Set" OK! ✅        │   │
    │  └──────────────────────────────────────────────────────┘   │
    │                                                              │
    │  ┌──────────────────────────────────────────────────────┐   │
    │  │ INTERFACE NAMES:                                     │   │
    │  │                                                      │   │
    │  │ → Interface 1 method → tên = method + "er"         │   │
    │  │ → Reader (method Read), Writer (method Write)      │   │
    │  │ → Stringer (method String)                          │   │
    │  │ → Formatter (method Format)                         │   │
    │  └──────────────────────────────────────────────────────┘   │
    │                                                              │
    │  ┌──────────────────────────────────────────────────────┐   │
    │  │ GOFMT — Formatting tự động:                         │   │
    │  │                                                      │   │
    │  │ → KHÔNG TRANH CÃI style! Dùng gofmt!              │   │
    │  │ → gofmt tự căn chỉnh: indent, alignment, comments │   │
    │  │ → Tất cả Go standard library dùng gofmt           │   │
    │  │ → "gofmt's style is nobody's favorite, but gofmt  │   │
    │  │    is everybody's favorite." — Rob Pike             │   │
    │  └──────────────────────────────────────────────────────┘   │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## §S6. Kiến Thức Bổ Sung — "What's in a Name?" (Andrew Gerrand)

> Nguồn: [What's in a name](https://www.youtube.com/watch?v=sFUSP8Au_PE) — Andrew Gerrand (Go Team, Google Sydney)
> Andrew Gerrand là thành viên Go Team, phụ trách release infrastructure, public-facing communications, và code review.

### Tại sao naming quan trọng?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  "There are two hard problems in computer science:          │
    │   cache invalidation, naming things, and off-by-one errors" │
    │                                                              │
    │  → Naming = GIAO DIỆN ĐẦU TIÊN người khác gặp code bạn!  │
    │  → Readability = PHẨM CHẤT CỐT LÕI của code tốt!         │
    │  → Good names = thành phần THIẾT YẾU của readability!      │
    │                                                              │
    │  3 TIÊU CHÍ CỦA TÊN TỐT:                                  │
    │  ┌──────────────┬────────────────────────────────────────┐   │
    │  │ 1. Consistent │ Giống style các tên khác, dễ đoán    │   │
    │  │ 2. Short      │ Gõ nhanh, đọc nhanh                  │   │
    │  │ 3. Accurate   │ Mô tả đúng thứ nó đại diện         │   │
    │  └──────────────┴────────────────────────────────────────┘   │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### QUY TẮC VÀNG: Khoảng cách & Độ dài tên

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ★ RULE OF THUMB — ANDREW GERRAND:                          │
    │                                                              │
    │  "The greater the DISTANCE between a name's DECLARATION     │
    │   and its USE, the LONGER the name should be."               │
    │                                                              │
    │  Khoảng cách KHAI BÁO ↔ SỬ DỤNG → Độ dài tên            │
    │                                                              │
    │  ┌──────────────────────────────────────────────────────┐    │
    │  │ NGẮN (1-2 ký tự):                                   │    │
    │  │ → Dùng ngay dòng kế tiếp (for loop, range)         │    │
    │  │ → for i := 0; i < n; i++                            │    │
    │  │ → for _, v := range items                           │    │
    │  ├──────────────────────────────────────────────────────┤    │
    │  │ VỪA (1 từ):                                         │    │
    │  │ → Dùng trong cùng function (~10-20 dòng)           │    │
    │  │ → buf := new(bytes.Buffer)                          │    │
    │  │ → req, err := http.NewRequest(...)                  │    │
    │  ├──────────────────────────────────────────────────────┤    │
    │  │ DÀI (mô tả rõ):                                    │    │
    │  │ → Exported, dùng bởi packages/projects KHÁC        │    │
    │  │ → var maxRetryAttempts int                          │    │
    │  │ → func NewTLSConfig(...) *tls.Config                │    │
    │  └──────────────────────────────────────────────────────┘    │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Local Variables — Tên NGẮN là tốt!

```
    ┌──────────────────────────────────────────────────────────────┐
    │  "Long local variable names HIDE what the code is actually  │
    │   doing. You spend time reading NAMES instead of reading    │
    │   CONTROL FLOW and reading the CODE."                        │
    │                                                              │
    │  → Prefer c to command                                      │
    │  → Prefer i to index                                        │
    │  → Prefer r to reader                                       │
    │  → Prefer b to buffer                                       │
    │                                                              │
    │  ★ "If you feel you NEED long names, maybe the solution    │
    │     is NOT long names but to REFACTOR your function         │
    │     to be SHORTER with FEWER variables."                     │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

```go
// ❌ BAD — Tên dài, khó đọc flow:
func (repository *Repository) RunCommand(command *Command) error {
    for _, command := range repository.Commands {
        command.Execute()
    }
    return nil
}

// ✅ GOOD — Tên ngắn, dễ đọc flow:
func (r *Repository) RunCommand(cmd *Command) error {
    for _, c := range r.Commands {
        c.Execute()
    }
    return nil
}
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  PHÂN TÍCH SỰ KHÁC BIỆT:                                  │
    │                                                              │
    │  ❌ repository *Repository → ✅ r *Repository              │
    │  → r xuất hiện KHẮP NƠI trong function                    │
    │  → "The whole subject of this function is the repo,        │
    │     so r is a fine name for it."                              │
    │                                                              │
    │  ❌ for _, command := range → ✅ for _, c := range         │
    │  → "I'm looping over commands. Why do I need the full     │
    │     name command? Surely c is obvious enough — it's on    │
    │     nearly every line of the following few lines."           │
    │                                                              │
    │  → Đặc biệt TỆ khi tên dài lặp lại nhiều lần:           │
    │  command.Name, command.Args, command.Execute()              │
    │  vs c.Name, c.Args, c.Execute() ← DỄ ĐỌC HƠN!          │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Function Parameters — Khi nào cần tên dài?

```
    ┌──────────────────────────────────────────────────────────────┐
    │  PARAMETERS = giống LOCAL VARIABLES, dùng tên ngắn!         │
    │                                                              │
    │  ★ Type MÔ TẢ rõ → tên parameter ngắn OK:                │
    │                                                              │
    │  func After(d time.Duration) <-chan Time     ← d OK!       │
    │  func TCP(addr string, f func() error) error ← f OK!      │
    │                                                              │
    │  → d cho Duration, f cho func → type nói lên tất cả!     │
    │                                                              │
    │  ★ Type KHÔNG mô tả (int, string) → tên parameter dài:   │
    │                                                              │
    │  func Timestamp(seconds, nanoseconds int64) Time            │
    │  → "seconds" và "nanoseconds" CẦN THIẾT để hiểu!        │
    │  → Nếu viết Timestamp(a, b int64) → HOÀN TOÀN vô nghĩa!│
    │                                                              │
    │  ★ Dùng PHÁN ĐOÁN:                                         │
    │  func HasPrefix(s, prefix string) bool                      │
    │  → s = string cần check, prefix = tiền tố              │
    │  → Đủ rõ ràng! Không cần inputString, prefixString       │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Named Return Values — Chỉ dùng cho Documentation!

```go
// ✅ GOOD — Named returns dùng để tài liệu hóa:
func Copy(dst Writer, src Reader) (written int64, err error)
// → "written" cho biết: number of bytes WRITTEN

// ✅ GOOD — Từ scanner package:
func (s *Scanner) Scan() (advance int, token []byte, err error)
// → "advance" = số bytes cần tiến, "token" = token trả về

// ❌ BAD — Named return THỪA:
func (f *File) Read(b []byte) (n int, err error)
// → "n int, err error" đã quá hiển nhiên, không cần named!
```

```
    ┌──────────────────────────────────────────────────────────────┐
    │  NAMED RETURN VALUES — ANDREW GERRAND's OPINION:            │
    │                                                              │
    │  "They should ONLY be used for DOCUMENTATION purposes."     │
    │                                                              │
    │  ⚠️ CẢNH BÁO: KHÔNG dùng "naked return"!                  │
    │                                                              │
    │  // ❌ NGUY HIỂM — Naked return:                           │
    │  func foo() (err error) {                                   │
    │      err = doSomething()                                    │
    │      if something {                                          │
    │          err := otherFunc()  // ← SHADOW! err MỚI!        │
    │          // ...                                              │
    │      }                                                       │
    │      return  // ← naked return: return err NGOÀI (nil!)   │
    │  }           //    KHÔNG phải err bị shadow bên trong!     │
    │                                                              │
    │  ★ NGOẠI LỆ DUY NHẤT — Dùng named return trong defer:    │
    │                                                              │
    │  func foo() (err error) {                                   │
    │      defer func() {                                          │
    │          if err != nil {                                     │
    │              err = fmt.Errorf("foo failed: %w", err)       │
    │          }                                                   │
    │      }()                                                     │
    │      // ... defer CÓ THỂ sửa err sau khi function return! │
    │      return doSomething()                                    │
    │  }                                                           │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Function Receivers — KHÔNG BAO GIỜ dùng `this` hay `self`!

```
    ┌──────────────────────────────────────────────────────────────┐
    │  RECEIVERS = giống local variables, dùng 1-2 ký tự!         │
    │                                                              │
    │  Convention: 1-2 chữ cái gợi nhớ đến tên TYPE             │
    │                                                              │
    │  ┌─────────────────────┬──────────────────────────────┐     │
    │  │ Type                │ Receiver                     │     │
    │  ├─────────────────────┼──────────────────────────────┤     │
    │  │ Buffer               │ b                            │     │
    │  │ ServerHandler        │ sh (hoặc h)                 │     │
    │  │ Rectangle            │ r                            │     │
    │  │ Client               │ c                            │     │
    │  │ Repository           │ r                            │     │
    │  └─────────────────────┴──────────────────────────────┘     │
    │                                                              │
    │  ❌ TUYỆT ĐỐI KHÔNG DÙNG:                                │
    │  func (this *Server) Start() { ... }  ← ❌ Java style!   │
    │  func (self *Server) Start() { ... }  ← ❌ Python style! │
    │                                                              │
    │  ✅ ĐÚNG:                                                   │
    │  func (s *Server) Start() { ... }     ← ✅ Go style!     │
    │                                                              │
    │  "If you're using longer names or worse, this or self,     │
    │   just cut it out."                                          │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Package Name = Phần Của API!

```
    ┌──────────────────────────────────────────────────────────────┐
    │  PACKAGE NAME LÀ PHẦN CỦA TÊN EXPORTED!                   │
    │                                                              │
    │  Khi export, người dùng LUÔN gõ: package.Name              │
    │  → Nên tên KHÔNG ĐƯỢC LẶP LẠI package name!              │
    │                                                              │
    │  ┌─────────────────────────────────────────────────────┐    │
    │  │ ❌ STUTTERING (lắp):                                │    │
    │  │                                                      │    │
    │  │ bytes.ByteBuffer → ❌ "bytes" lặp!                  │    │
    │  │ strings.StringReader → ❌ "string" lặp!             │    │
    │  │ http.HTTPClient → ❌ "http" lặp!                    │    │
    │  ├─────────────────────────────────────────────────────┤    │
    │  │ ✅ ĐÚNG:                                             │    │
    │  │                                                      │    │
    │  │ bytes.Buffer → ✅ gọn, rõ!                          │    │
    │  │ strings.Reader → ✅                                  │    │
    │  │ http.Client → ✅                                     │    │
    │  └─────────────────────────────────────────────────────┘    │
    │                                                              │
    │  ★ Tên trong package có thể "trông ngắn" lạ:              │
    │  → Bên TRONG package: type Buffer struct { ... }           │
    │  → Bên NGOÀI dùng: bytes.Buffer ← HOÀN HẢO!             │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Interface Naming — Method + "er"

```
    ┌──────────────────────────────────────────────────────────────┐
    │  INTERFACE 1 METHOD → Tên = Method + "er"                  │
    │                                                              │
    │  ┌─────────────────────────────────────────────────────┐    │
    │  │ Method    → Interface Name                          │    │
    │  ├─────────────────────────────────────────────────────┤    │
    │  │ Read()    → Reader        ← tiếng Anh đúng!       │    │
    │  │ Write()   → Writer        ← tiếng Anh đúng!       │    │
    │  │ String()  → Stringer      ← OK!                    │    │
    │  │ Format()  → Formatter     ← OK!                    │    │
    │  │ Exec()    → Execer        ← KHÔNG phải từ thật!   │    │
    │  └─────────────────────────────────────────────────────┘    │
    │                                                              │
    │  "Execer is not a word... maybe it is in French...         │
    │   but we do it anyway because it makes it PREDICTABLE      │
    │   which is one of the important aspects of readability."    │
    │                                                              │
    │  ★ Tính PREDICTABLE quan trọng hơn tiếng Anh đúng!       │
    │  → Khi thấy interface XxxEr → biết có method Xxx()!       │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Error Naming Convention

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ERROR NAMING — PHÂN BIỆT VARIABLE vs TYPE                 │
    │                                                              │
    │  ┌─────────────────────────────────────────────────────┐    │
    │  │ Error VALUE (biến):      Err + Tên                  │    │
    │  │                                                      │    │
    │  │ var ErrNotFound = errors.New("not found")           │    │
    │  │ var ErrFormat = errors.New("bad format")            │    │
    │  │ var ErrTimeout = errors.New("timeout")              │    │
    │  ├─────────────────────────────────────────────────────┤    │
    │  │ Error TYPE (kiểu):      Tên + Error                 │    │
    │  │                                                      │    │
    │  │ type ExitError struct { ... }                        │    │
    │  │ type SyntaxError struct { ... }                      │    │
    │  │ type PathError struct { ... }                        │    │
    │  └─────────────────────────────────────────────────────┘    │
    │                                                              │
    │  → Khi thấy ErrXxx → biết là ERROR VALUE (sentinel)       │
    │  → Khi thấy XxxError → biết là ERROR TYPE!                │
    │  → Convention từ standard library, dùng trong mọi project! │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Package Naming — KHÔNG dùng "util", "common", "helpers"!

```
    ┌──────────────────────────────────────────────────────────────┐
    │  PACKAGE NAMING — ANDREW GERRAND:                           │
    │                                                              │
    │  → lowercase, KHÔNG dùng camelCase hay under_scores        │
    │  → TÊN có nghĩa! Package name mang ý nghĩa cho exports  │
    │                                                              │
    │  ❌ BAD PACKAGES:                                           │
    │  package util     ← ❌ Quá chung chung!                    │
    │  package common   ← ❌ Mọi thứ đều "common"!             │
    │  package helpers  ← ❌ Helper cho cái gì?                 │
    │  package misc     ← ❌ Vô nghĩa!                          │
    │                                                              │
    │  ✅ GOOD PACKAGES:                                          │
    │  package auth     ← ✅ Rõ ràng: authentication!           │
    │  package cache    ← ✅ Rõ ràng: caching!                  │
    │  package config   ← ✅ Rõ ràng: configuration!            │
    │                                                              │
    │  "It's better to have MANY SMALLER meaningful packages     │
    │   with meaningful names than one catch-all util."            │
    │                                                              │
    │  ★ Import path: directory name = package name               │
    │  compress/gzip → package gzip  ✅                           │
    │  compress/gzip → package whatever  ❌ Rất confusing!      │
    │                                                              │
    │  ★ Tránh stuttering trong import paths:                    │
    │  ❌ go.tools/go/types     → import lặp "go" 2 lần!       │
    │  ✅ golang.org/x/tools    → gọn, không lặp!              │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Kết luận từ Andrew Gerrand

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ★ TÓM GỌN TRONG 3 CÂU:                                   │
    │                                                              │
    │  1. USE SHORT NAMES! Đây là điều quan trọng nhất!          │
    │                                                              │
    │  2. Think about CONTEXT — nghĩ về NGƯỜI DÙNG tên bạn,    │
    │     không chỉ nghĩ cho bản thân lúc viết code!             │
    │                                                              │
    │  3. Use your JUDGMENT — không có hard rules tuyệt đối,    │
    │     luôn có exceptions, hãy áp dụng linh hoạt!             │
    │                                                              │
    │  ★ Standard Library = nơi TỐT để học Go idiom!            │
    │  → NHƯNG không 100% hoàn hảo (viết khi team còn học Go!) │
    │  → "Don't come to me and say 'but I found this in the     │
    │     standard library!' Yes, because we learned since."      │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

> **Tài liệu tham khảo:**
>
> - [Built-In Types](http://golang.org/ref/spec#Boolean_types)
> - [Variables](https://golang.org/doc/effective_go.html#variables)
> - [Gustavo's IEEE-754 Brain Teaser](https://www.ardanlabs.com/blog/2013/08/gustavos-ieee-754-brain-teaser.html)
> - [What's in a name](https://www.youtube.com/watch?v=sFUSP8Au_PE) — Andrew Gerrand
