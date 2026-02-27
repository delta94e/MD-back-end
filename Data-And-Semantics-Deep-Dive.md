# Design Philosophy On Data And Semantics — Triết Lý Về Dữ Liệu & Ngữ Nghĩa

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Design Philosophy On Data And Semantics" của William Kennedy (2017)
> **Góc nhìn:** Value vs Pointer Semantics — quyết định thiết kế quan trọng nhất trong Go
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                  | Mô tả                                    |
| --- | --------------------------------------- | ---------------------------------------- |
| §1  | Value vs Pointer Semantics              | Định nghĩa & tại sao phải chọn           |
| §2  | Mental Models & Debugging               | Mô hình tư duy & code dễ debug           |
| §3  | Data Oriented Design                    | "Nếu không hiểu data, không hiểu vấn đề" |
| §4  | Type Is Life                            | Hệ thống kiểu = nền tảng integrity       |
| §5  | Semantic Guidelines                     | Quy tắc chọn Value vs Pointer            |
| §6  | Built-In, Reference, User-Defined Types | Áp dụng guidelines cho từng loại         |
| §7  | Ví dụ thực tế: time.Time vs os.File     | Value semantic vs Pointer semantic       |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior     | Ôn tập & thực hành                       |

---

## §1. Value Semantics vs Pointer Semantics

### 1.1 Hai cách truyền dữ liệu

```
╔═══════════════════════════════════════════════════════════════╗
║   VALUE vs POINTER SEMANTICS — HAI CÁCH TRUYỀN DATA          ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  William Kennedy:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Value semantics keep values on the STACK,   │             ║
║  │  which reduces pressure on the GC.           │             ║
║  │  However, value semantics require copies.    │             ║
║  │                                              │             ║
║  │  Pointer semantics place values on the HEAP, │             ║
║  │  which can put pressure on the GC.           │             ║
║  │  However, pointer semantics are efficient    │             ║
║  │  because only ONE value needs to be stored." │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VALUE SEMANTICS (Ngữ nghĩa giá trị):                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  func process(t Time) {                       │             ║
║  │      // t là BẢN SAO! Original KHÔNG đổi!  │             ║
║  │  }                                            │             ║
║  │                                              │             ║
║  │  ┌─────────┐    COPY    ┌─────────┐         │             ║
║  │  │ Original│ ──────────→│  Copy   │         │             ║
║  │  │ (Stack) │            │ (Stack) │         │             ║
║  │  └─────────┘            └─────────┘         │             ║
║  │                                              │             ║
║  │  ✅ Trên Stack → ít GC pressure             │             ║
║  │  ✅ An toàn — không ai thay đổi original   │             ║
║  │  ✅ Predictable — mỗi function có bản riêng│             ║
║  │  ❌ Tốn memory nếu data LỚN (nhiều copies) │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  POINTER SEMANTICS (Ngữ nghĩa con trỏ):                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  func process(f *File) {                      │             ║
║  │      // f trỏ tới CÙNG value! Thay đổi     │             ║
║  │      // ảnh hưởng TẤT CẢ!                  │             ║
║  │  }                                            │             ║
║  │                                              │             ║
║  │  ┌─────────┐  SHARE    ┌─────────┐         │             ║
║  │  │ Pointer │ ─────────→│  Value  │         │             ║
║  │  │ (Stack) │           │ (Heap)  │         │             ║
║  │  └─────────┘           └─────────┘         │             ║
║  │       ▲                      ▲              │             ║
║  │       └── Func A             └── Func B     │             ║
║  │       Cả 2 CHIA SẺ cùng 1 value!           │             ║
║  │                                              │             ║
║  │  ✅ Hiệu quả — chỉ 1 value, chia sẻ       │             ║
║  │  ✅ Mutation được thấy ở mọi nơi           │             ║
║  │  ❌ GC pressure (Heap allocation)            │             ║
║  │  ❌ Data races nếu concurrent!               │             ║
║  │  ❌ Side effects khó kiểm soát              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NGUYÊN TẮC BẤT KHẢ XÂM PHẠM:                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "A consistent use of value/pointer semantics │             ║
║  │  for a given type of data is CRITICAL if you │             ║
║  │  want to maintain INTEGRITY and READABILITY. │             ║
║  │                                              │             ║
║  │  If you are CHANGING the semantics for a     │             ║
║  │  piece of data as it passes between          │             ║
║  │  functions, you make it difficult to         │             ║
║  │  maintain a clear mental model."             │             ║
║  │                                              │             ║
║  │  → CHỌN 1 semantic cho 1 type               │             ║
║  │  → DÙNG NHẤT QUÁN trong toàn bộ API!      │             ║
║  │  → KHÔNG trộn lẫn!                          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Mental Models & Debugging

### 2.1 Mental model = nền tảng mọi thứ

```
╔═══════════════════════════════════════════════════════════════╗
║   MENTAL MODELS — MÔ HÌNH TƯ DUY                             ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Tom Love (inventor of Objective-C):                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "A project with a MILLION lines of code.     │             ║
║  │  Probability of success in the US: WELL      │             ║
║  │  UNDER 50%."                                 │             ║
║  │                                              │             ║
║  │ 1 hộp giấy A4 = 100,000 dòng code!         │             ║
║  │ 1 triệu dòng = 10 HỘP GIẤY!               │             ║
║  │                                              │             ║
║  │ Bạn có thể GIỮ TRONG ĐẦU bao nhiêu %     │             ║
║  │ code trong 1 hộp giấy??                      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ~10,000 LOC / developer = giới hạn hợp lý  │             ║
║  │                                              │             ║
║  │ 1 triệu LOC → 100 developers cần phối hợp!│             ║
║  │ → Coordinated, grouped, tracked!            │             ║
║  │                                              │             ║
║  │ Team 5 người? Codebase NÊN ~50K LOC!       │             ║
║  │ Codebase 500K LOC với 5 người? NGUY HIỂM!  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Brian Kernighan (co-creator C, Unix):                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ VỀ DEBUGGING:                                 │             ║
║  │ "The HARDEST bugs are those where your       │             ║
║  │  MENTAL MODEL of the situation is just       │             ║
║  │  WRONG, so you can't see the problem at all."│             ║
║  │                                              │             ║
║  │ → Bug KHÓ NHẤT = khi bạn HIỂU SAI code!   │             ║
║  │ → Debugger = evil khi lạm dụng!            │             ║
║  │ → LOGS phải hoạt động = bạn CÓ mental model│             ║
║  │ → Production bugs → logs → đọc code →     │             ║
║  │   tìm lỗi = CẦN mental model!              │             ║
║  │                                              │             ║
║  │ VỀ READABILITY:                               │             ║
║  │ "C is the best balance between power and     │             ║
║  │  expressiveness... you will have a very good │             ║
║  │  MENTAL MODEL of what's going to happen."    │             ║
║  │                                              │             ║
║  │ Kennedy: Quote này áp dụng cho GO y hệt!    │             ║
║  │ → Go = đọc code = BIẾT machine sẽ làm gì! │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VALUE/POINTER SEMANTICS + MENTAL MODEL:                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Nhất quán semantics = DỄ giữ mental model!  │             ║
║  │ Trộn lẫn semantics = mental model VỠ!       │             ║
║  │                                              │             ║
║  │ Time value semantic: mọi nơi COPY           │             ║
║  │ → Dễ hiểu: "Time không đổi, tạo mới!"    │             ║
║  │                                              │             ║
║  │ File pointer semantic: mọi nơi SHARE         │             ║
║  │ → Dễ hiểu: "File chỉ có 1, chia sẻ!"     │             ║
║  │                                              │             ║
║  │ ❌ Nếu Time lúc value lúc pointer:           │             ║
║  │ → "Time này copy hay share??" → BUGS!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Data Oriented Design

### 3.1 Mọi vấn đề = Data Transformation

```
╔═══════════════════════════════════════════════════════════════╗
║   DATA ORIENTED DESIGN — THIẾT KẾ HƯỚNG DỮ LIỆU            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "If you don't understand the DATA, you don't │             ║
║  │  understand the PROBLEM."                    │             ║
║  │                                              │             ║
║  │ → KHÔNG HIỂU data = KHÔNG HIỂU vấn đề!    │             ║
║  │                                              │             ║
║  │ "All problems are UNIQUE and SPECIFIC to     │             ║
║  │  the data you are working with."             │             ║
║  │                                              │             ║
║  │ → Data thay đổi → vấn đề thay đổi         │             ║
║  │ → Vấn đề thay đổi → algorithms thay đổi   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  MỌI VẤN ĐỀ = DATA TRANSFORMATION:                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌───────┐    ┌──────────────┐   ┌────────┐ │             ║
║  │  │ INPUT │ →  │ FUNCTION     │ → │ OUTPUT │ │             ║
║  │  │ DATA  │    │ (transform)  │   │ DATA   │ │             ║
║  │  └───────┘    └──────────────┘   └────────┘ │             ║
║  │                                              │             ║
║  │ → Mọi function = INPUT → biến đổi → OUTPUT│             ║
║  │ → Mọi program = chuỗi data transformations │             ║
║  │ → Mental model = hiểu cách data CHẢY QUA   │             ║
║  │   code!                                      │             ║
║  │                                              │             ║
║  │ "LESS IS MORE" attitude:                      │             ║
║  │ → Ít layers, ít statements                  │             ║
║  │ → Ít generalizations, ít complexity         │             ║
║  │ → Ít effort → dễ cho bạn, team, và PHẦN   │             ║
║  │   CỨNG!                                      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Type Is Life & Data With Capability

### 4.1 Type = nền tảng integrity

```
╔═══════════════════════════════════════════════════════════════╗
║   TYPE IS LIFE — KIỂU DỮ LIỆU LÀ SỰ SỐNG                  ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "In my world, TYPE IS LIFE, because type     │             ║
║  │  provides the compiler with the ability to   │             ║
║  │  ensure the INTEGRITY of the data."          │             ║
║  │                                              │             ║
║  │ → Type = Compiler BẢO VỆ data integrity!    │             ║
║  │ → Type ĐIỀU KHIỂN semantic rules!            │             ║
║  │ → Value/Pointer semantics BẮT ĐẦU từ Type! │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  DATA WITH CAPABILITY (Dữ liệu có khả năng):               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Methods are valid when it is practical or   │             ║
║  │  reasonable for a piece of DATA to have a    │             ║
║  │  CAPABILITY."                                │             ║
║  │                                              │             ║
║  │ → Focus vào DATA chứ không phải behavior!   │             ║
║  │ → Data DRIVES functionality!                 │             ║
║  │ → Data DRIVES algorithms!                    │             ║
║  │ → Data DRIVES performance!                   │             ║
║  │                                              │             ║
║  │ Ví dụ Go:                                     │             ║
║  │ type File struct { fd int; name string }     │             ║
║  │ → File CÓ KHẢ NĂNG Read, Write, Close      │             ║
║  │ → Data (file descriptor) QUYẾT ĐỊNH hành vi│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  POLYMORPHISM:                                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Tom Kurtz (inventor of BASIC):                │             ║
║  │ "Polymorphism means that you write a certain │             ║
║  │  program and it behaves DIFFERENTLY depending│             ║
║  │  on the DATA that it operates on."           │             ║
║  │                                              │             ║
║  │ → Behavior của data = decouple functions    │             ║
║  │ → Đây là lý do DATA cần CAPABILITY!        │             ║
║  │ → Cornerstone cho systems thích ứng thay đổi│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  PROTOTYPE FIRST:                                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Hiểu CONCRETE data + algorithms TRƯỚC   │             ║
║  │ 2. Viết concrete implementation TRƯỚC        │             ║
║  │ 3. Sau khi CHẠY ĐƯỢC → refactor             │             ║
║  │ 4. Decouple bằng cách cho data CAPABILITY   │             ║
║  │    (interfaces)                               │             ║
║  │                                              │             ║
║  │ → ĐỪNG abstract trước khi hiểu data!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Semantic Guidelines — Quy Tắc Chọn

### 5.1 Năm quy tắc vàng

```
╔═══════════════════════════════════════════════════════════════╗
║   SEMANTIC GUIDELINES — 5 QUY TẮC VÀNG                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ QUY TẮC 1:                                    │             ║
║  │ Khi KHAI BÁO type → QUYẾT ĐỊNH semantic!   │             ║
║  │ → Value hay Pointer? CHỌN NGAY!             │             ║
║  │                                              │             ║
║  │ QUY TẮC 2:                                    │             ║
║  │ Functions và methods PHẢI TÔN TRỌNG         │             ║
║  │ semantic đã chọn cho type!                   │             ║
║  │                                              │             ║
║  │ QUY TẮC 3:                                    │             ║
║  │ TRÁNH method receivers dùng semantic         │             ║
║  │ KHÁC với type!                                │             ║
║  │                                              │             ║
║  │ QUY TẮC 4:                                    │             ║
║  │ TRÁNH functions accept/return data bằng     │             ║
║  │ semantic KHÁC với type!                       │             ║
║  │                                              │             ║
║  │ QUY TẮC 5:                                    │             ║
║  │ TRÁNH thay đổi semantic cho 1 type!         │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NGOẠI LỆ DUY NHẤT:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ UNMARSHALING luôn cần POINTER SEMANTICS!     │             ║
║  │                                              │             ║
║  │ func (t *Time) UnmarshalJSON(data []byte)    │             ║
║  │ → PHẢI dùng pointer receiver!                │             ║
║  │ → Vì cần GHI vào value gốc!                │             ║
║  │                                              │             ║
║  │ Marshaling/Unmarshaling = LUÔN là ngoại lệ! │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CÁCH CHỌN SEMANTIC:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Hỏi: "Copy data có ĐÚNG và HỢP LÝ?"      │             ║
║  │        │                                     │             ║
║  │     ┌──┴──┐                                  │             ║
║  │     │     │                                  │             ║
║  │    CÓ    KHÔNG / KHÔNG CHẮC                  │             ║
║  │     │         │                              │             ║
║  │     ▼         ▼                              │             ║
║  │  VALUE    POINTER                            │             ║
║  │  SEM.     SEM.                               │             ║
║  │                                              │             ║
║  │  "If you are NOT 100% sure it is correct    │             ║
║  │   or reasonable to make copies,              │             ║
║  │   then use POINTER semantics."               │             ║
║  │                                              │             ║
║  │  → KHÔNG CHẮC = dùng POINTER!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Ba Loại Type — Áp Dụng Guidelines

### 6.1 Built-In Types → VALUE semantics

```go
// ═══ BUILT-IN TYPES → VALUE SEMANTICS ═══

// int, float, string, bool, byte, rune
// → LUÔN dùng VALUE semantics!
// → ĐỪNG dùng pointer trừ khi lý do RẤT tốt!

// Ví dụ từ strings package:
func Replace(s, old, new string, n int) string  // VALUE in, VALUE out!
func LastIndex(s, sep string) int                // VALUE in, VALUE out!
func ContainsRune(s string, r rune) bool         // VALUE in, VALUE out!

// ❌ XẤU: Pointer cho built-in type
func badProcess(name *string) *int  // TẠI SAO pointer cho string/int??

// ✅ TỐT: Value cho built-in type
func goodProcess(name string) int   // VALUE in, VALUE out!
```

### 6.2 Reference Types → VALUE semantics

```go
// ═══ REFERENCE TYPES → VALUE SEMANTICS ═══

// slice, map, interface, function, channel
// → Bản thân đã chứa POINTER bên trong!
// → LUÔN dùng VALUE semantics!
// → ĐỪNG dùng pointer to slice/map!

// Ví dụ từ net package:
type IP []byte      // Slice = reference type
type IPMask []byte  // Slice = reference type

// Method dùng VALUE RECEIVER cho reference type!
func (ip IP) Mask(mask IPMask) IP {
    // ip là COPY của header slice (nhưng share underlying array!)
    // Mutation → tạo IP MỚI và return!
    n := len(ip)
    out := make(IP, n)
    for i := 0; i < n; i++ {
        out[i] = ip[i] & mask[i]
    }
    return out // Return BẢN MỚI!
}

// append() cũng dùng VALUE semantics!
var data []string
data = append(data, "hello")
// → Truyền slice VALUE vào append()
// → Nhận slice VALUE MỚI về!
// → VALUE semantic cho mutation!
```

```
╔═══════════════════════════════════════════════════════════════╗
║   REFERENCE TYPES — TẠI SAO VALUE SEMANTICS?                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Slice internal:                                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ┌─────────────────────┐                      │             ║
║  │ │ Slice Header (Stack)│ → VALUE semantic!    │             ║
║  │ │ ┌─────────────────┐ │                      │             ║
║  │ │ │ Pointer to array│─┼───→ [a][b][c][d]    │             ║
║  │ │ │ Length: 4        │ │     (Heap data)     │             ║
║  │ │ │ Capacity: 8     │ │                      │             ║
║  │ │ └─────────────────┘ │                      │             ║
║  │ └─────────────────────┘                      │             ║
║  │                                              │             ║
║  │ → Header là SMALL (3 words = 24 bytes)      │             ║
║  │ → COPY header, SHARE underlying data!       │             ║
║  │ → Giữ trên Stack → ít GC pressure!         │             ║
║  │ → Mỗi function có BẢN SAO header riêng!    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NGOẠI LỆ: Unmarshal!                                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ func (ip *IP) UnmarshalText(text []byte) error│            ║
║  │ → POINTER receiver cho Unmarshal!            │             ║
║  │ → Vì cần GHI VÀO value gốc!                │             ║
║  │ → ĐÂY LÀ NGOẠI LỆ hợp lệ!                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 6.3 User-Defined Types → CẦN QUYẾT ĐỊNH!

```
╔═══════════════════════════════════════════════════════════════╗
║   USER-DEFINED TYPES — BẠN PHẢI CHỌN!                        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Đây là nơi BẠN phải quyết định!                            ║
║  → Khi khai báo type → CHỌN semantic!                       ║
║  → Factory function = DẤU HIỆU #1!                          ║
║                                                               ║
║  CÁCH NHẬN BIẾT SEMANTIC TỪ FACTORY FUNCTION:               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ // Return VALUE → VALUE semantics!           │             ║
║  │ func Now() Time { ... }                       │             ║
║  │ → Trả về Time (không phải *Time)            │             ║
║  │ → Copy value cho caller!                      │             ║
║  │ → Dùng VALUE semantics toàn bộ!            │             ║
║  │                                              │             ║
║  │ // Return POINTER → POINTER semantics!       │             ║
║  │ func Open(name string) (*File, error) { ... }│             ║
║  │ → Trả về *File (pointer!)                   │             ║
║  │ → Share value với caller!                    │             ║
║  │ → Dùng POINTER semantics toàn bộ!          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Ví Dụ Thực Tế: time.Time vs os.File

### 7.1 time.Time — VALUE semantics

```go
// ═══ time.Time — VALUE SEMANTICS (HOÀN TOÀN) ═══

// Type definition:
type Time struct {
    sec  int64
    nsec int32
    loc  *Location
}

// Factory: trả VALUE → dấu hiệu VALUE semantics!
func Now() Time {
    sec, nsec := now()
    return Time{sec + unixToInternal, nsec, Local}
    // → Return COPY of Time!
}

// Mutation method: VALUE receiver → copy → mutate → return new!
func (t Time) Add(d Duration) Time {
    t.sec += int64(d / 1e9)      // Mutate BẢN SAO!
    nsec := t.nsec + int32(d%1e9)
    if nsec >= 1e9 {
        t.sec++
        nsec -= 1e9
    }
    t.nsec = nsec
    return t // Return BẢN MỚI!
}
// → t original KHÔNG BỊ THAY ĐỔI!
// → Caller nhận TIME MỚI!

// Functions accepting Time: cũng VALUE!
func div(t Time, d Duration) (qmod2 int, r Duration) { ... }
// → Accept Time by VALUE!

// CHỈ CÓ Unmarshal dùng POINTER:
func (t *Time) UnmarshalBinary(data []byte) error { ... }
func (t *Time) UnmarshalJSON(data []byte) error { ... }
// → NGOẠI LỆ hợp lệ duy nhất!
```

```
╔═══════════════════════════════════════════════════════════════╗
║   time.Time — VALUE SEMANTIC FLOW                             ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  t1 := time.Now()        // t1 = value trên Stack            ║
║  t2 := t1.Add(5*time.Second) // t2 = NEW value!             ║
║                                                               ║
║  ┌─────────┐  Add()   ┌─────────┐                           ║
║  │ t1      │ ──copy──→│ t (copy)│──mutate──→ return t2      ║
║  │ 10:00:00│          │ 10:00:00│           │ 10:00:05│      ║
║  └─────────┘          └─────────┘           └─────────┘      ║
║       │                                                       ║
║       └→ KHÔNG ĐỔI! Vẫn 10:00:00!                          ║
║                                                               ║
║  → t1 được COPY vào Add()                                    ║
║  → Add() mutate COPY, return VALUE MỚI                      ║
║  → t1 NGUYÊN VẸN!                                            ║
║  → PREDICTABLE! SAFE! NO SIDE EFFECTS!                       ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 7.2 os.File — POINTER semantics

```go
// ═══ os.File — POINTER SEMANTICS (HOÀN TOÀN) ═══

// Factory: trả POINTER → dấu hiệu POINTER semantics!
func Open(name string) (file *File, err error) {
    return OpenFile(name, O_RDONLY, 0)
    // → Return *File (POINTER!)
    // → "Bạn CHIA SẺ File này, ĐỪNG copy!"
}

// Methods: POINTER receiver TOÀN BỘ!
func (f *File) Chdir() error {
    // → Dù KHÔNG mutate, vẫn dùng pointer!
    // → Tôn trọng semantic convention của type!
    if f == nil { return ErrInvalid }
    if e := syscall.Fchdir(f.fd); e != nil {
        return &PathError{"chdir", f.name, e}
    }
    return nil
}

func (f *File) Read(b []byte) (n int, err error) { ... }
func (f *File) Write(b []byte) (n int, err error) { ... }
func (f *File) Close() error { ... }
// → TẤT CẢ đều *File (pointer receiver!)

// Functions accepting File: cũng POINTER!
func epipecheck(file *File, e error) {
    // → Accept *File (POINTER!)
    // → NHẤT QUÁN với toàn bộ API!
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   os.File — POINTER SEMANTIC FLOW                             ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  f, _ := os.Open("data.txt")  // f = *File (pointer!)       ║
║  f.Read(buf)                   // SHARE f!                    ║
║  processFile(f)                // SHARE f!                    ║
║                                                               ║
║  ┌─────────┐                                                  ║
║  │ f (ptr) │──┐                                               ║
║  └─────────┘  │                                               ║
║               ▼                                               ║
║  ┌──────────────────┐                                         ║
║  │ File struct      │ ← Trên HEAP, CHIA SẺ!                 ║
║  │ fd: 3            │                                         ║
║  │ name: "data.txt" │                                         ║
║  └──────────────────┘                                         ║
║         ▲        ▲                                            ║
║         │        │                                            ║
║  processFile(f)  main()                                       ║
║                                                               ║
║  → TẤT CẢ SHARE cùng 1 File!                                ║
║  → COPY File = THẢM HỌA! (2 nơi cùng fd!)                  ║
║  → "Changing semantic from pointer to value                  ║
║     could be DEVASTATING."                                   ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 7.3 Bảng tổng hợp

```
╔═══════════════════════════════════════════════════════════════╗
║   BẢNG TỔNG HỢP — CHỌN SEMANTIC THEO TYPE                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌────────────────┬───────────┬───────────────────┐          ║
║  │ LOẠI TYPE      │ SEMANTIC  │ VÍ DỤ             │          ║
║  ├────────────────┼───────────┼───────────────────┤          ║
║  │ Built-in       │ VALUE     │ int, string, bool │          ║
║  │ (int, string)  │           │ float64, byte     │          ║
║  ├────────────────┼───────────┼───────────────────┤          ║
║  │ Reference      │ VALUE     │ []byte, map[k]v   │          ║
║  │ (slice, map)   │           │ chan, func, iface  │          ║
║  ├────────────────┼───────────┼───────────────────┤          ║
║  │ Small struct   │ VALUE     │ time.Time          │          ║
║  │ (no resource)  │           │ Color, Point       │          ║
║  ├────────────────┼───────────┼───────────────────┤          ║
║  │ Resource type  │ POINTER   │ os.File            │          ║
║  │ (fd, conn)     │           │ net.Conn           │          ║
║  │                │           │ sql.DB             │          ║
║  ├────────────────┼───────────┼───────────────────┤          ║
║  │ Large struct   │ POINTER   │ Config (100 field) │          ║
║  │                │           │                    │          ║
║  ├────────────────┼───────────┼───────────────────┤          ║
║  │ Unmarshal      │ POINTER   │ *Time, *IP         │          ║
║  │ (NGOẠI LỆ!)   │ (always!) │ (luôn ngoại lệ!)  │          ║
║  └────────────────┴───────────┴───────────────────┘          ║
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
║  Q1: "Value vs Pointer semantics — khi nào chọn cái nào?"   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ VALUE khi:                                    │             ║
║  │ → Built-in types (int, string, bool)         │             ║
║  │ → Reference types (slice, map, chan)          │             ║
║  │ → Small structs không giữ resource           │             ║
║  │ → Copy là ĐÚNG và HỢP LÝ                   │             ║
║  │                                              │             ║
║  │ POINTER khi:                                  │             ║
║  │ → Struct giữ resource (fd, conn, mutex)      │             ║
║  │ → Large structs (copy tốn kém)              │             ║
║  │ → Không chắc copy có đúng không!            │             ║
║  │ → Unmarshaling (luôn pointer!)               │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Tại sao NHẤT QUÁN semantics quan trọng?"              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Mental model RÕ RÀNG!                      │             ║
║  │ → Trộn semantics → bugs, data races,        │             ║
║  │   side effects KHÔNG THẤY!                   │             ║
║  │ → Codebase lớn → team lớn → PHẢI nhất quán │             ║
║  │ → Kennedy: "Integrity and readability"       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Factory function cho biết điều gì?"                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Factory = DẤU HIỆU #1 cho semantic!          │             ║
║  │                                              │             ║
║  │ func Now() Time → VALUE semantics cho Time   │             ║
║  │ func Open() *File → POINTER semantics cho File│            ║
║  │                                              │             ║
║  │ → Return value = VALUE semantic              │             ║
║  │ → Return pointer = POINTER semantic          │             ║
║  │ → TOÀN BỘ API phải FOLLOW factory!          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "'Data Oriented Design' nghĩa là gì?"                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → "Không hiểu data = không hiểu vấn đề"   │             ║
║  │ → Mọi program = data transformations         │             ║
║  │ → Focus vào DATA trước, behavior sau         │             ║
║  │ → Type = Life, Type drives semantics         │             ║
║  │ → Prototype concrete TRƯỚC, abstract SAU!    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: DESIGN PHILOSOPHY ON DATA AND SEMANTICS       │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. CHỌN semantic KHI khai báo type!       │            │
    │  │    → Value hoặc Pointer, KHÔNG trộn!     │            │
    │  │                                          │            │
    │  │ 2. NHẤT QUÁN trong toàn bộ API!           │            │
    │  │    → Factory, methods, functions          │            │
    │  │    → TẤT CẢ cùng semantic!               │            │
    │  │                                          │            │
    │  │ 3. Factory function = DẤU HIỆU #1        │            │
    │  │    → Return value = VALUE semantic        │            │
    │  │    → Return *ptr = POINTER semantic       │            │
    │  │                                          │            │
    │  │ 4. Ngoại lệ: UNMARSHAL = luôn pointer!   │            │
    │  │                                          │            │
    │  │ 5. KHÔNG CHẮC? → Dùng POINTER!           │            │
    │  │                                          │            │
    │  │ 6. Data Oriented Design:                  │            │
    │  │    → Hiểu data TRƯỚC                     │            │
    │  │    → "Type Is Life"                        │            │
    │  │    → Data drives functionality             │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "The consistent use of value/pointer semantics         │
    │   is something I LOOK FOR in code reviews.              │
    │   It helps you keep code CONSISTENT and                 │
    │   PREDICTABLE over time."                               │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
