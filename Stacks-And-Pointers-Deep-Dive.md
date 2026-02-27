# Language Mechanics On Stacks And Pointers — Cơ Chế Stack & Con Trỏ

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Language Mechanics On Stacks And Pointers" của William Kennedy (2017)
> **Góc nhìn:** Stack, Frame Boundaries, Pass by Value, Pointers — nền tảng cốt lõi của Go
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                              | Mô tả                                  |
| --- | ----------------------------------- | -------------------------------------- |
| §1  | Goroutine & Stack                   | Stack là gì, goroutine khởi tạo ra sao |
| §2  | Frame Boundaries                    | Mỗi function có không gian riêng       |
| §3  | Pass By Value                       | Go truyền mọi thứ bằng VALUE           |
| §4  | Addresses & Biến                    | Mọi biến đều có địa chỉ                |
| §5  | Function Returns                    | Stack memory sau khi return            |
| §6  | Pointers — Sharing Values           | Con trỏ = chia sẻ value                |
| §7  | Indirect Memory Access              | Đọc/ghi memory NGOÀI frame             |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior | Ôn tập & thực hành                     |

---

## §1. Goroutine & Stack

### 1.1 Stack là gì?

```
╔═══════════════════════════════════════════════════════════════╗
║   GOROUTINE & STACK — NỀN TẢNG                               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  KHI CHƯƠNG TRÌNH GO KHỞI ĐỘNG:                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Runtime tạo MAIN GOROUTINE                 │             ║
║  │ 2. Goroutine = "đường dẫn thực thi"         │             ║
║  │    → Đặt trên OS thread                      │             ║
║  │    → Thực thi trên CPU core                  │             ║
║  │ 3. Mỗi goroutine nhận 1 STACK riêng          │             ║
║  │    → Khởi tạo: 2,048 bytes (2KB)            │             ║
║  │    → Block memory LIÊN TỤC (contiguous)      │             ║
║  │    → Có thể GROW khi cần thêm               │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  STACK CỦA GOROUTINE:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Địa chỉ CAO (top of memory)                │             ║
║  │  ┌──────────────────────────┐                │             ║
║  │  │                          │ ← Valid!       │             ║
║  │  │    (free stack space)     │                │             ║
║  │  │                          │                │             ║
║  │  ├══════════════════════════╡                │             ║
║  │  │  main() FRAME            │ ← Active Frame│             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ count = 10         │  │                │             ║
║  │  │  │ addr: 0x10429fa4   │  │                │             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  ├══════════════════════════╡                │             ║
║  │  │  (invalid memory below)  │ ← INVALID!    │             ║
║  │  └──────────────────────────┘                │             ║
║  │  Địa chỉ THẤP (bottom)                      │             ║
║  │                                              │             ║
║  │  QUAN TRỌNG:                                  │             ║
║  │  → Memory TỪ active frame TRỞ LÊN = VALID  │             ║
║  │  → Memory DƯỚI active frame = INVALID!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Frame Boundaries — Ranh Giới Frame

### 2.1 Mỗi function = 1 frame

```
╔═══════════════════════════════════════════════════════════════╗
║   FRAME BOUNDARIES — RANH GIỚI FRAME                          ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  FRAME BOUNDARY = không gian memory RIÊNG                    ║
║  cho mỗi function!                                            ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Mỗi function được GỌI → tạo FRAME mới!     │             ║
║  │                                              │             ║
║  │ Frame cung cấp:                               │             ║
║  │ 1. MEMORY RIÊNG cho function                 │             ║
║  │    → Biến local sống ở đây!                 │             ║
║  │    → Function ĐỌC/GHI TRỰC TIẾP!           │             ║
║  │                                              │             ║
║  │ 2. CONTEXT riêng                              │             ║
║  │    → Mỗi function hoạt động TÁCH BIỆT!     │             ║
║  │                                              │             ║
║  │ 3. FLOW CONTROL                               │             ║
║  │    → Call → tạo frame mới DƯỚI              │             ║
║  │    → Return → xóa frame, quay LÊN          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TRUY CẬP MEMORY:                                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ TRONG frame → TRỰC TIẾP (qua frame pointer) │             ║
║  │ NGOÀI frame → GIÁN TIẾP (cần được "share") │             ║
║  │                                              │             ║
║  │ → Function KHÔNG THỂ tự ý truy cập memory  │             ║
║  │   của function khác!                          │             ║
║  │ → PHẢI được SHARE (chia sẻ)!                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Pass By Value — Truyền Bằng Giá Trị

### 3.1 Go truyền MỌI THỨ by value

```go
// ═══ PASS BY VALUE — CHƯƠNG TRÌNH VÍ DỤ ═══

package main

func main() {
    // Khai báo count = 10
    count := 10

    // Hiện value và address
    println("count:\tValue Of[", count, "]\tAddr Of[", &count, "]")

    // Truyền VALUE của count vào increment
    increment(count)  // ← COPY value 10!

    // count KHÔNG ĐỔI!
    println("count:\tValue Of[", count, "]\tAddr Of[", &count, "]")
}

//go:noinline
func increment(inc int) {  // inc nhận BẢN SAO!
    inc++  // Chỉ tăng BẢN SAO!
    println("inc:\tValue Of[", inc, "]\tAddr Of[", &inc, "]")
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   PASS BY VALUE — TỪNG BƯỚC TRÊN STACK                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  BƯỚC 1: Trước khi gọi increment()                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Stack:                                        │             ║
║  │                                              │             ║
║  │  ┌──────────────────────────┐                │             ║
║  │  │  main() FRAME            │ ← Active      │             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ count = 10         │  │                │             ║
║  │  │  │ addr: 0x10429fa4   │  │                │             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  └──────────────────────────┘                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BƯỚC 2: Gọi increment(count) — COPY value!                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Stack:                                        │             ║
║  │                                              │             ║
║  │  ┌──────────────────────────┐                │             ║
║  │  │  main() FRAME            │                │             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ count = 10         │  │ addr: fa4      │             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  ├──────────────────────────┤                │             ║
║  │  │  increment() FRAME       │ ← Active      │             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ inc = 10  (COPY!)  │  │ addr: f98      │             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  └──────────────────────────┘                │             ║
║  │                                              │             ║
║  │  → count ở addr fa4 = 10                     │             ║
║  │  → inc   ở addr f98 = 10 (BẢN SAO!)        │             ║
║  │  → HAI BIẾN, HAI ĐỊA CHỈ, CÙNG GIÁ TRỊ!  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BƯỚC 3: Sau inc++ (chỉ tăng BẢN SAO!)                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Stack:                                        │             ║
║  │                                              │             ║
║  │  ┌──────────────────────────┐                │             ║
║  │  │  main() FRAME            │                │             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ count = 10 ← KHÔNG │  │ KHÔNG ĐỔI!   │             ║
║  │  │  │           ĐỔI!     │  │                │             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  ├──────────────────────────┤                │             ║
║  │  │  increment() FRAME       │                │             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ inc = 11 ← ĐỔI!   │  │ Chỉ bản sao! │             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  └──────────────────────────┘                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  OUTPUT:                                                       ║
║  count:  Value Of[ 10 ]  Addr Of[ 0x10429fa4 ]               ║
║  inc:    Value Of[ 11 ]  Addr Of[ 0x10429f98 ]               ║
║  count:  Value Of[ 10 ]  Addr Of[ 0x10429fa4 ]               ║
║                                                               ║
║  → count KHÔNG ĐỔI sau khi increment() return!              ║
║  → Vì increment() làm việc với BẢN SAO!                     ║
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Pass by value = WYSIWYG!"                   │             ║
║  │ (What You See Is What You Get!)               │             ║
║  │                                              │             ║
║  │ → Nhìn function call = BIẾT cái gì được copy│             ║
║  │ → KHÔNG có chi phí ẩn!                      │             ║
║  │ → Code READABLE!                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Addresses & Biến

### 4.1 Mọi biến đều có địa chỉ

```
╔═══════════════════════════════════════════════════════════════╗
║   ADDRESSES — ĐỊA CHỈ BỘ NHỚ                                ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Biến = TÊN gán cho 1 VỊ TRÍ memory!                       ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ count := 10                                   │             ║
║  │                                              │             ║
║  │ → "count" = TÊN                              │             ║
║  │ → 10      = GIÁ TRỊ                         │             ║
║  │ → &count  = ĐỊA CHỈ (0x10429fa4)           │             ║
║  │                                              │             ║
║  │ Memory:                                       │             ║
║  │ ┌────────────┬───────┐                       │             ║
║  │ │ Addr       │ Value │                       │             ║
║  │ ├────────────┼───────┤                       │             ║
║  │ │ 0x10429fa4 │  10   │ ← "count"            │             ║
║  │ └────────────┴───────┘                       │             ║
║  │                                              │             ║
║  │ Toán tử &:                                   │             ║
║  │ → & = "address of" = lấy ĐỊA CHỈ          │             ║
║  │ → &count = 0x10429fa4                        │             ║
║  │                                              │             ║
║  │ Nếu có value trong memory → PHẢI có address!│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Function Returns — Sau Khi Return

### 5.1 Memory KHÔNG bị dọn dẹp!

```
╔═══════════════════════════════════════════════════════════════╗
║   FUNCTION RETURNS — STACK SAU KHI RETURN                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  SAU KHI increment() RETURN:                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Stack:                                        │             ║
║  │                                              │             ║
║  │  ┌──────────────────────────┐                │             ║
║  │  │  main() FRAME            │ ← Active      │             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ count = 10         │  │                │             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  ├──────────────────────────┤                │             ║
║  │  │  (increment frame)       │ ← INVALID!    │             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ inc = 11 ← VẪN CÒN│  │ Nhưng INVALID!│             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  └──────────────────────────┘                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CHUYỆN GÌ XẢY RA?                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Memory của increment frame VẪN CÒN ĐÓ!  │             ║
║  │ → KHÔNG bị dọn dẹp!                         │             ║
║  │ → Giá trị 11 vẫn nằm ở addr f98!           │             ║
║  │                                              │             ║
║  │ TẠI SAO KHÔNG DỌN?                            │             ║
║  │ → Phí phạm thời gian! Vì chưa chắc memory  │             ║
║  │   đó có được dùng lại!                       │             ║
║  │ → Go dọn KHI CẦN: khi function MỚI được    │             ║
║  │   gọi → frame MỚI lấy vùng đó → TẤT CẢ   │             ║
║  │   biến được KHỞI TẠO (zero value)!          │             ║
║  │                                              │             ║
║  │ → "Stacks clean themselves properly on       │             ║
║  │    every function call" — vì mọi value       │             ║
║  │    được init to zero value!                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Pointers — Sharing Values

### 6.1 Con trỏ = chia sẻ

```
╔═══════════════════════════════════════════════════════════════╗
║   POINTERS — CON TRỎ = CHIA SẺ                              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Pointers serve ONE PURPOSE: to SHARE a      │             ║
║  │  value with a function so the function can   │             ║
║  │  READ and WRITE to that value even though    │             ║
║  │  the value does NOT exist directly inside    │             ║
║  │  its own frame."                             │             ║
║  │                                              │             ║
║  │ → Con trỏ CHỈ có 1 mục đích: CHIA SẺ!     │             ║
║  │                                              │             ║
║  │ "If the word SHARE doesn't come out of your  │             ║
║  │  mouth, you don't need to use a pointer."    │             ║
║  │                                              │             ║
║  │ → Không cần "share" = KHÔNG cần pointer!     │             ║
║  │                                              │             ║
║  │ "Replace the & operator for the word         │             ║
║  │  SHARING as you read code."                  │             ║
║  │                                              │             ║
║  │ → Đọc & = "chia sẻ"!                       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  POINTER TYPES:                                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Mỗi type → có MIỄN PHÍ 1 pointer type!     │             ║
║  │                                              │             ║
║  │ int     → *int     (pointer to int)          │             ║
║  │ string  → *string  (pointer to string)       │             ║
║  │ User    → *User    (pointer to User)         │             ║
║  │ []byte  → *[]byte  (pointer to slice)        │             ║
║  │                                              │             ║
║  │ Đặc điểm chung TẤT CẢ pointer types:        │             ║
║  │ → Bắt đầu bằng *                            │             ║
║  │ → CÙNG kích thước: 4 bytes (32-bit)         │             ║
║  │                      8 bytes (64-bit)         │             ║
║  │ → Lưu trữ ĐỊA CHỈ (address)               │             ║
║  │                                              │             ║
║  │ ❌ Pointer types KHÔNG hoán đổi được!       │             ║
║  │ → *int KHÔNG thể nhận address của string!    │             ║
║  │ → Cần TYPE INFO để read/write value!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Indirect Memory Access — Truy Cập Gián Tiếp

### 7.1 Chương trình với Pointer

```go
// ═══ PASS BY VALUE — NHƯNG VALUE LÀ ADDRESS! ═══

package main

func main() {
    count := 10

    println("count:\tValue Of[", count, "]\tAddr Of[", &count, "]")

    // Truyền ADDRESS của count! ("SHARING" count!)
    increment(&count)  // ← & = SHARING!

    // count ĐÃ ĐỔI! Vì increment ghi trực tiếp!
    println("count:\tValue Of[", count, "]\tAddr Of[", &count, "]")
}

//go:noinline
func increment(inc *int) {  // inc = pointer to int!
    // Đọc/ghi value mà pointer TRỎ TỚI!
    *inc++  // ← * = dereference = "value that pointer points to"!
    println("inc:\tValue Of[", inc, "]\tAddr Of[", &inc, "]\tPoints To[", *inc, "]")
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   INDIRECT MEMORY ACCESS — TRUY CẬP GIÁN TIẾP               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  BƯỚC 1: Gọi increment(&count) — SHARE address!             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Stack:                                        │             ║
║  │                                              │             ║
║  │  ┌──────────────────────────┐                │             ║
║  │  │  main() FRAME            │                │             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ count = 10         │◄─┼──────────┐    │             ║
║  │  │  │ addr: 0xfa4        │  │           │    │             ║
║  │  │  └────────────────────┘  │           │    │             ║
║  │  ├──────────────────────────┤           │    │             ║
║  │  │  increment() FRAME       │           │    │             ║
║  │  │  ┌────────────────────┐  │           │    │             ║
║  │  │  │ inc = 0xfa4 ───────┼──┼───────────┘    │             ║
║  │  │  │ (pointer to int!)  │  │                │             ║
║  │  │  │ addr: 0xf98        │  │                │             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  └──────────────────────────┘                │             ║
║  │                                              │             ║
║  │  → inc CHỨA address 0xfa4                    │             ║
║  │  → 0xfa4 = address của count trong main()!  │             ║
║  │  → inc TRỎ TỚI count!                       │             ║
║  │                                              │             ║
║  │  ĐÂY VẪN LÀ PASS BY VALUE!                  │             ║
║  │  → Value được copy = ADDRESS (0xfa4)         │             ║
║  │  → Addresses cũng là VALUES!                 │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  BƯỚC 2: *inc++ — Dereference & Modify!                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Stack:                                        │             ║
║  │                                              │             ║
║  │  ┌──────────────────────────┐                │             ║
║  │  │  main() FRAME            │                │             ║
║  │  │  ┌────────────────────┐  │                │             ║
║  │  │  │ count = 11 ← ĐỔI! │◄─┼──────────┐    │             ║
║  │  │  │ addr: 0xfa4        │  │           │    │             ║
║  │  │  └────────────────────┘  │           │    │             ║
║  │  ├──────────────────────────┤           │    │             ║
║  │  │  increment() FRAME       │           │    │             ║
║  │  │  ┌────────────────────┐  │           │    │             ║
║  │  │  │ inc = 0xfa4 ───────┼──┼───────────┘    │             ║
║  │  │  │ *inc++ → GHI qua  │  │                │             ║
║  │  │  │ pointer TRỰC TIẾP │  │                │             ║
║  │  │  │ vào main() frame!  │  │                │             ║
║  │  │  └────────────────────┘  │                │             ║
║  │  └──────────────────────────┘                │             ║
║  │                                              │             ║
║  │  → *inc = "value mà inc TRỎ TỚI"           │             ║
║  │  → *inc++ = tăng value đó!                  │             ║
║  │  → count trong main() = 11!                  │             ║
║  │  → increment() GHI VÀO frame của main()!    │             ║
║  │  → Đây gọi là INDIRECT MEMORY ACCESS!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  OUTPUT:                                                       ║
║  count: Value Of[ 10 ]           Addr Of[ 0x10429fa4 ]       ║
║  inc:   Value Of[ 0x10429fa4 ]   Addr Of[ 0x10429f98 ]       ║
║                                  Value Points To[ 11 ]        ║
║  count: Value Of[ 11 ]           Addr Of[ 0x10429fa4 ]       ║
║                                                               ║
║  → count ĐÃ ĐỔI thành 11! Vì increment ghi                 ║
║    TRỰC TIẾP qua pointer!                                    ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 7.2 Toán tử \* — hai ý nghĩa

```
╔═══════════════════════════════════════════════════════════════╗
║   TOÁN TỬ * — HAI Ý NGHĨA                                   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  * CÓ 2 Ý NGHĨA KHÁC NHAU:                                 ║
║                                                               ║
║  1. KHAI BÁO KIỂU (Type declaration):                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ func increment(inc *int)                      │             ║
║  │                    ↑                         │             ║
║  │              *int = "pointer to int" TYPE!    │             ║
║  │              → Khai báo inc LÀ pointer!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  2. TOÁN TỬ (Operator — dereference):                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ *inc++                                        │             ║
║  │ ↑                                            │             ║
║  │ * = "value mà pointer TRỎ TỚI"              │             ║
║  │ → Truy cập value GIÁN TIẾP qua pointer!     │             ║
║  │ → Gọi là "dereferencing"!                    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  PHÂN BIỆT:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ *int   → TYPE (pointer to int)               │             ║
║  │ *inc   → OPERATOR (value at pointer)         │             ║
║  │ &count → OPERATOR (address of count)         │             ║
║  │                                              │             ║
║  │ Nếu phân biệt rõ KHAI BÁO vs TOÁN TỬ       │             ║
║  │ → bớt nhầm lẫn RẤT NHIỀU!                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  POINTER VARIABLE = BIẾN BÌNH THƯỜNG:                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Kennedy:                                      │             ║
║  │ "Pointer variables are NOT SPECIAL because   │             ║
║  │  they are variables like any other variable.  │             ║
║  │  They have a memory allocation and they      │             ║
║  │  hold a value."                              │             ║
║  │                                              │             ║
║  │ → Pointer = biến giống mọi biến khác!       │             ║
║  │ → Có địa chỉ riêng (addr f98)              │             ║
║  │ → Chứa value (addr fa4)                     │             ║
║  │ → Chỉ khác: value = ĐỊA CHỈ!              │             ║
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
║  Q1: "Go là pass by value hay pass by reference?"            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Go LUÔN là PASS BY VALUE!                     │             ║
║  │                                              │             ║
║  │ → Mọi thứ đều được COPY khi truyền!        │             ║
║  │ → Kể cả khi truyền pointer:                  │             ║
║  │   increment(&count)                           │             ║
║  │   → COPY address 0xfa4 vào inc                │             ║
║  │   → Address cũng là VALUE!                   │             ║
║  │                                              │             ║
║  │ → Go KHÔNG CÓ pass by reference!             │             ║
║  │ → "Pass by value = WYSIWYG"                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Frame boundary là gì? Tại sao quan trọng?"            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = Không gian memory RIÊNG cho MỖI function!  │             ║
║  │                                              │             ║
║  │ → Function ĐỌC/GHI TRỰC TIẾP trong frame   │             ║
║  │ → KHÔNG THỂ truy cập frame khác (trừ khi   │             ║
║  │   được SHARE qua pointer!)                    │             ║
║  │ → Tạo CÁCH LY giữa functions!               │             ║
║  │ → Nền tảng cho Go's memory safety!            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Pointer trong Go dùng để làm gì?"                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ CHỈ 1 mục đích: SHARING!                    │             ║
║  │                                              │             ║
║  │ → Chia sẻ value TỪ frame này SANG frame khác│             ║
║  │ → Function có thể ĐỌC/GHI value ngoài frame│             ║
║  │ → Indirect memory access!                     │             ║
║  │                                              │             ║
║  │ "If SHARE doesn't come out of your mouth,    │             ║
║  │  you don't need a pointer."                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Goroutine stack hoạt động thế nào?"                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Mỗi goroutine: stack riêng 2KB ban đầu    │             ║
║  │ → Contiguous block of memory                  │             ║
║  │ → GROW khi cần (double size)                 │             ║
║  │ → Memory DƯỚI active frame = INVALID         │             ║
║  │ → Khi function call: frame MỚI dưới         │             ║
║  │ → Khi return: frame đánh dấu INVALID        │             ║
║  │   (nhưng memory không xóa ngay!)             │             ║
║  │ → Zero value initialization khi frame MỚI   │             ║
║  │   được tạo = self-cleaning!                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: STACKS AND POINTERS                          │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Go = LUÔN pass by value!               │            │
    │  │    → Copy value qua frame boundary!       │            │
    │  │    → WYSIWYG! Readable!                   │            │
    │  │                                          │            │
    │  │ 2. Frame Boundary = memory RIÊNG!         │            │
    │  │    → Truy cập TRONG = trực tiếp          │            │
    │  │    → Truy cập NGOÀI = cần pointer!       │            │
    │  │                                          │            │
    │  │ 3. Pointer = SHARING!                     │            │
    │  │    → & = "chia sẻ address"               │            │
    │  │    → * (type) = "pointer to X"            │            │
    │  │    → * (operator) = "value at pointer"    │            │
    │  │                                          │            │
    │  │ 4. Stack tự dọn dẹp!                     │            │
    │  │    → Return = frame INVALID               │            │
    │  │    → New call = zero value init!          │            │
    │  │                                          │            │
    │  │ 5. Pointer variables = biến BÌNH THƯỜNG! │            │
    │  │    → Có address, có value                 │            │
    │  │    → Value = address (4 or 8 bytes)       │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "Without a strong understanding of pointers,           │
    │   you will struggle to write clean, simple              │
    │   and efficient code."                                  │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
