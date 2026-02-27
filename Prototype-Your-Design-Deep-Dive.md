# Prototype Your Design — Xây Dựng Thay Vì Suy Nghĩ

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài nói chuyện "Prototype Your Design!" của Robert Griesemer tại GopherCon
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                    | Mô tả                                       |
| --- | ----------------------------------------- | ------------------------------------------- |
| §1  | Giới thiệu                                | Tại sao thiết kế phần mềm cần prototyping?  |
| §2  | Quy trình thiết kế phần mềm truyền thống  | Design document → Review → Iteration        |
| §3  | Design Thinking & Prototyping             | Triết lý "Xây dựng thay vì suy nghĩ"        |
| §4  | Bài toán cụ thể: Multi-dimensional Slices | Hỗ trợ ứng dụng số học trong Go             |
| §5  | Hiện trạng Go: Arrays vs Slices           | Tại sao Go chưa hỗ trợ matrix?              |
| §6  | Vấn đề Notation — Cú pháp đẹp             | Thiếu cú pháp indexing đa chiều             |
| §7  | Giải pháp: Rewriter — Prototype           | Viết lại code tự động thay vì sửa compiler  |
| §8  | Thiết kế Prototype: Indexed Getter/Setter | Method names đặc biệt cho indexing          |
| §9  | Hiện thực hóa: Parser → AST → Printer     | Pipeline xử lý syntax tree                  |
| §10 | Type Checker — Cứu tinh của Prototype     | Dùng type checker để biết cần rewrite ở đâu |
| §11 | Vòng lặp Rewrite: Parse → Check → Rewrite | Iterative rewriting cho đến khi hết lỗi     |
| §12 | Ứng dụng: Matrix Multiplication           | Nhân ma trận với cú pháp đẹp                |
| §13 | Bài học bất ngờ từ Prototype              | Phát hiện không lường trước được            |
| §14 | Triết lý Go & Prototyping                 | Go là ngôn ngữ tuyệt vời cho prototyping    |
| §15 | Tổng kết & Câu hỏi phỏng vấn              | Ôn tập & thực hành cho Senior Golang        |

---

## §1. Giới Thiệu — Tại Sao Thiết Kế Phần Mềm Cần Prototyping?

### 1.1 Bối cảnh

Robert Griesemer — một trong ba người tạo ra ngôn ngữ Go (cùng Rob Pike và Ken Thompson) — đã trình bày bài nói chuyện này tại GopherCon. Thông điệp cốt lõi: **Đừng chỉ NGHĨ về thiết kế — hãy XÂY DỰNG nó!**

```
╔═══════════════════════════════════════════════════════════════╗
║     THÔNG ĐIỆP CỐT LÕI CỦA BÀI NÓI CHUYỆN                  ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Câu hỏi trung tâm:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Làm sao biết thiết kế của chúng ta TỐT?"  │             ║
║  │                                              │             ║
║  │ How can we tell that our design is any good? │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Câu trả lời:                                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Đừng CHỈ suy nghĩ → hãy XÂY DỰNG!          │             ║
║  │                                              │             ║
║  │ "A designer does not THINK her way to a      │             ║
║  │  good design. She BUILDS her way to the      │             ║
║  │  good design."                               │             ║
║  │                                              │             ║
║  │ → Designers are really DOERS.                │             ║
║  │ → We might even call them HACKERS!           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 1.2 Tại sao Senior Golang Developer cần biết điều này?

```
    ┌──────────────────────────────────────────────────────────┐
    │  TẠI SAO CẦN HIỂU VỀ PROTOTYPING?                       │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  1. THIẾT KẾ HỆ THỐNG khi phỏng vấn                    │
    │     → Senior phải giải thích QUY TRÌNH thiết kế         │
    │     → Không chỉ "viết code" mà phải "thiết kế trước"   │
    │                                                          │
    │  2. HIỂU SÂU về Go toolchain                            │
    │     → Bài nói dùng go/parser, go/ast, go/types          │
    │     → Đây là standard library MẠNH MẼ của Go            │
    │                                                          │
    │  3. TƯ DUY PROTOTYPE                                    │
    │     → Giải quyết vấn đề PHỨC TẠP bằng cách             │
    │       xây prototype ĐƠN GIẢN trước                     │
    │     → Tiết kiệm thời gian và chi phí                    │
    │                                                          │
    │  4. TRIẾT LÝ GO                                          │
    │     → "Go brought back the fun to programming"          │
    │     → Go = ngôn ngữ tuyệt vời cho prototyping           │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

---

## §2. Quy Trình Thiết Kế Phần Mềm Truyền Thống

### 2.1 Quy trình phổ biến trong doanh nghiệp

Trong bất kỳ môi trường phần mềm chuyên nghiệp nào đều có quy trình thiết kế, và mục tiêu thường giống nhau: **đảm bảo thiết kế vững chắc TRƯỚC KHI bắt đầu implement**.

```
╔═══════════════════════════════════════════════════════════════╗
║   QUY TRÌNH THIẾT KẾ PHẦN MỀM TRUYỀN THỐNG                   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Bước 1: Design Document (Tài liệu thiết kế)                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ • Đặc tả phần mềm muốn xây dựng            │             ║
║  │ • Kiến trúc tổng thể                        │             ║
║  │ • API contracts, data models                 │             ║
║  │ • Ước tính scope, timeline                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                   │                                           ║
║                   ▼                                           ║
║  Bước 2: Review Process (Phản hồi từ chuyên gia)            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ • Experts + Design reviewers đọc document    │             ║
║  │ • Phản hồi, góp ý, phát hiện lỗ hổng       │             ║
║  │ • Q&A sessions                               │             ║
║  └──────────────────────────────────────────────┘             ║
║                   │                                           ║
║                   ▼                                           ║
║  Bước 3: Iteration (Lặp lại quy trình)                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ • Cập nhật document dựa trên feedback        │             ║
║  │ • Review lại                                 │             ║
║  │ • Lặp cho đến khi "đủ tốt"                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VẤN ĐỀ:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ⚠️ Đây thường là "dry exercise"              │             ║
║  │    (bài tập khô khan)                        │             ║
║  │                                              │             ║
║  │ → KHÔNG có code nào được viết!               │             ║
║  │ → Chỉ suy nghĩ trên giấy                    │             ║
║  │ → Không biết SAI ở đâu cho đến khi          │             ║
║  │   implement thật!                            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 2.2 Bắt lỗi sớm = Rẻ hơn

```
    ┌──────────────────────────────────────────────────────────┐
    │  CHI PHÍ SỬA LỖI THEO GIAI ĐOẠN                         │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  Chi phí                                                 │
    │    ▲                                                     │
    │    │                                        ████         │
    │    │                                        ████         │
    │    │                                  ████  ████         │
    │    │                           ████   ████  ████         │
    │    │                    ████   ████   ████  ████         │
    │    │             ████   ████   ████   ████  ████         │
    │    │      ████   ████   ████   ████   ████  ████         │
    │    │ ██   ████   ████   ████   ████   ████  ████         │
    │    └──────────────────────────────────────────────►       │
    │      Design Review Implement  Test  Deploy Production    │
    │                                                          │
    │  → Lỗi phát hiện ở giai đoạn Design = RẺ NHẤT          │
    │  → Lỗi phát hiện ở Production = ĐẮT NHẤT               │
    │                                                          │
    │  NHƯNG: Design trên giấy KHÔNG phát hiện được           │
    │  tất cả lỗi! → Cần PROTOTYPE để phát hiện sớm hơn!     │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

---

## §3. Design Thinking & Prototyping — Triết Lý "Xây Dựng Thay Vì Suy Nghĩ"

### 3.1 Thiết kế NGOÀI ngành phần mềm

Robert Griesemer nhấn mạnh: **NGOÀI ngành phần mềm**, thiết kế và prototyping luôn đi ĐÔI với nhau. Không ai thiết kế quần áo, đồ nội thất, xe hơi, hay kiến trúc mà KHÔNG tạo prototype trước khi sản xuất!

```
╔═══════════════════════════════════════════════════════════════╗
║   DESIGN THINKING — QUY TRÌNH 5 BƯỚC (Stanford d.school)     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Quy trình thiết kế theo Stanford Design School:              ║
║                                                               ║
║  ┌──────────┐  ┌──────────┐  ┌──────────────────────────┐    ║
║  │ 1.EMPATHY│─►│ 2.DEFINE │─►│ 3. IDEATE → 4. PROTOTYPE│    ║
║  │ (Hiểu   │  │ (Định   │  │      → 5. TEST           │    ║
║  │ người   │  │ nghĩa   │  │ (Lặp lại 3→4→5)         │    ║
║  │ dùng)   │  │ vấn đề) │  │                          │    ║
║  └──────────┘  └──────────┘  └──────────────────────────┘    ║
║                                                               ║
║  Bước 1-2: Tìm hiểu vấn đề                                  ║
║  → Google gọi: "Focus on the user and all else will follow"  ║
║                                                               ║
║  Bước 3-4-5: LẶP LẠI liên tục:                              ║
║                                                               ║
║      ┌────────┐                                               ║
║      │ IDEATE │◄────────────────────┐                         ║
║      │ (Ý    │                     │                         ║
║      │ tưởng)│                     │                         ║
║      └───┬────┘                     │                         ║
║          │                          │                         ║
║          ▼                          │                         ║
║      ┌──────────┐                   │                         ║
║      │PROTOTYPE │                   │                         ║
║      │(Xây dựng │                   │                         ║
║      │ mẫu)     │                   │ Feedback                ║
║      └───┬──────┘                   │ loop!                   ║
║          │                          │                         ║
║          ▼                          │                         ║
║      ┌────────┐                     │                         ║
║      │ TEST   │─────────────────────┘                         ║
║      │(Thử   │                                               ║
║      │ nghiệm)│                                               ║
║      └────────┘                                               ║
║                                                               ║
║  INSIGHT QUAN TRỌNG:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "A designer does not THINK her way to a      │             ║
║  │  good design. She BUILDS her way to the      │             ║
║  │  good design."                               │             ║
║  │                                              │             ║
║  │ → Thiết kế TỐT đến từ XÂY DỰNG + THỬ       │             ║
║  │ → KHÔNG phải từ suy nghĩ trên giấy!          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 3.2 So sánh: Thế giới bên ngoài vs Phần mềm

```
    ┌──────────────────────────────────────────────────────────┐
    │  SO SÁNH QUY TRÌNH THIẾT KẾ                              │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  THẾ GIỚI BÊN NGOÀI:                                    │
    │  ┌──────────────────────────────────────────┐            │
    │  │ Quần áo  → Mẫu thử (fitting)            │            │
    │  │ Đồ gỗ   → Mô hình thu nhỏ              │            │
    │  │ Xe hơi   → Prototype, crash test         │            │
    │  │ Kiến trúc → Mô hình 3D, mockup          │            │
    │  │ Nghệ thuật → Phác thảo (sketch)          │            │
    │  │                                          │            │
    │  │ → LUÔN CÓ prototype trước sản xuất!     │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  PHẦN MỀM (thường):                                     │
    │  ┌──────────────────────────────────────────┐            │
    │  │ Document → Review → ... → Implement     │            │
    │  │                                          │            │
    │  │ ⚠️ Nhảy từ thiết kế THẲNG sang code!     │            │
    │  │ ⚠️ Không có bước prototyping!             │            │
    │  │ ⚠️ Phát hiện vấn đề khi đã quá muộn!    │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  → BÀI HỌC: Phần mềm cũng CẦN prototype!              │
    │  → Đặc biệt cho những thay đổi LỚN và PHỨC TẠP!       │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

---

## §4. Bài Toán Cụ Thể: Multi-dimensional Slices Cho Go

### 4.1 Nhu cầu thực tế

Cộng đồng Go (đặc biệt GoNum) đã nộp **proposal** yêu cầu hỗ trợ **multi-dimensional slices** (mảng đa chiều) trong Go — cần cho các ứng dụng số học như ma trận, xử lý ảnh, machine learning.

```
╔═══════════════════════════════════════════════════════════════╗
║   BÀI TOÁN: MULTI-DIMENSIONAL SLICES CHO GO                   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Mong muốn: Khai báo slices đa chiều                         ║
║                                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ // Khai báo slice 2 chiều (ma trận)          │             ║
║  │ var matrix [][]float64                       │             ║
║  │                                              │             ║
║  │ // Tạo ma trận 3x4 với make                  │             ║
║  │ m := make([][]float64, 3, 4)                 │             ║
║  │                                              │             ║
║  │ // Truy cập phần tử bằng 2 chỉ số           │             ║
║  │ val := m[i, j]     // ← KHÔNG thể trong Go! │             ║
║  │ m[i, j] = 3.14     // ← KHÔNG thể trong Go! │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Mục tiêu:                                                    ║
║  1. Readability tốt hơn qua cú pháp đẹp (nice notation)     ║
║  2. Performance tốt qua native implementation                ║
║                                                               ║
║  Câu hỏi then chốt:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "But again, how do we know they are the      │             ║
║  │  RIGHT answers?"                             │             ║
║  │                                              │             ║
║  │ → Có design document rồi, nhưng làm sao     │             ║
║  │   biết thiết kế ĐÃ ĐÚNG chưa?               │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Hiện Trạng Go: Arrays vs Slices

### 5.1 Go hiện tại có gì?

```
╔═══════════════════════════════════════════════════════════════╗
║   HIỆN TRẠNG: ARRAYS vs SLICES TRONG GO                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ARRAY — Kích thước CỐ ĐỊNH:                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ var grid [3][4]float64  // 3 hàng, 4 cột     │             ║
║  │                                              │             ║
║  │ Bộ nhớ: liên tục (contiguous)                │             ║
║  │ ┌────┬────┬────┬────┬────┬────┬────┬────┐    │             ║
║  │ │r0c0│r0c1│r0c2│r0c3│r1c0│r1c1│r1c2│r1c3│...│             ║
║  │ └────┴────┴────┴────┴────┴────┴────┴────┘    │             ║
║  │                                              │             ║
║  │ ✅ Hỗ trợ 2D: grid[i][j]                     │             ║
║  │ ✅ Bộ nhớ liên tục → cache-friendly          │             ║
║  │ ❌ Kích thước PHẢI biết lúc compile!         │             ║
║  │ ❌ Không thể thay đổi kích thước runtime     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SLICE — Kích thước ĐỘNG:                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ s := make([]float64, 12)  // 1D slice         │             ║
║  │                                              │             ║
║  │ Slice header:                                 │             ║
║  │ ┌─── ptr ──────► [0][1][2]...[11]            │             ║
║  │ │ len = 12       underlying array             │             ║
║  │ │ cap = 12                                    │             ║
║  │ └────────────                                 │             ║
║  │                                              │             ║
║  │ ✅ Kích thước ĐỘNG (runtime)                  │             ║
║  │ ✅ Có thể grow bằng append                    │             ║
║  │ ❌ Chỉ có 1 chiều (1D)!                      │             ║
║  │ ❌ s[i, j] ← KHÔNG HỢP LỆ!                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  KẾT LUẬN:                                                    ║
║  → Array: 2D nhưng kích thước cố định                        ║
║  → Slice: kích thước động nhưng chỉ 1D                       ║
║  → KHÔNG CÓ cấu trúc vừa 2D vừa dynamic!                   ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 5.2 Quan sát quan trọng của Robert Griesemer

```
    ┌──────────────────────────────────────────────────────────┐
    │  QUAN SÁT: CHÚNG TA ĐÃ CÓ GÌ?                          │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  Những gì ĐÃ CÓ THỂ LÀM trong Go hiện tại:            │
    │                                                          │
    │  ✅ Định nghĩa slice descriptors cho 2D, 3D, 4D        │
    │     → Chỉ là abstract data type!                        │
    │     → type Matrix struct { data []float64, ... }        │
    │                                                          │
    │  ✅ Định nghĩa operations trên data types đó            │
    │     → Methods: func (m *Matrix) Get(i, j int)           │
    │     → GoNum community đã làm điều này!                  │
    │                                                          │
    │  ❌ THIẾU: Nice notation (cú pháp đẹp)                  │
    │     → m[i, j]  ← KHÔNG THỂ!                            │
    │     → Phải dùng m.At(i, j) ← xấu, clunky!             │
    │                                                          │
    │  → Vấn đề CỐT LÕI chỉ là CÚ PHÁP!                    │
    │  → Logic đã có, chỉ thiếu "syntactic sugar"!           │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

---

## §6. Vấn Đề Notation — Cú Pháp Đẹp

### 6.1 So sánh: Cú pháp mong muốn vs thực tế

```go
// ❌ Thực tế hiện tại: CLUNKY (rườm rà)
result := m.At(i, j)          // Đọc phần tử
m.AtSet(i, j, 3.14)          // Gán phần tử

// → Code số học trông "xấu" và "khó đọc"!

// ✅ Mong muốn: CLEAN (sạch đẹp)
result := m[i, j]             // Đọc phần tử — tự nhiên!
m[i, j] = 3.14               // Gán phần tử — trực quan!

// → Giống như viết toán trên giấy!
```

### 6.2 Ba lựa chọn

```
╔═══════════════════════════════════════════════════════════════╗
║   BA CÁCH GIẢI QUYẾT VẤN ĐỀ NOTATION                        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Lựa chọn 1: "Không có vấn đề gì"                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Chấp nhận cú pháp hiện tại                │             ║
║  │ → Có thể OK cho một số người                │             ║
║  │ → NHƯNG không tốt cho long-term!            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Lựa chọn 2: Sửa Go compiler                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Thêm multi-dimensional index vào ngôn ngữ │             ║
║  │ → ĐẮT ĐỎ! Phải sửa compiler, spec, tools   │             ║
║  │ → Chỉ để XEM thiết kế có đúng không?        │             ║
║  │ → Nếu sai → MẤT RẤT NHIỀU công sức!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Lựa chọn 3: ★ REWRITER (Viết lại code) ★                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Viết code bằng cú pháp MỚI                │             ║
║  │ → Tự động REWRITE thành Go hợp lệ           │             ║
║  │ → RẺ! Không cần sửa compiler!               │             ║
║  │ → Có thể KHÁM PHÁ thiết kế trước!           │             ║
║  │ → ĐÂY LÀ PROTOTYPE!                        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  → Robert Griesemer chọn lựa chọn 3!                        ║
║  → "Explore the design space BEFORE making hard decisions"   ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Giải Pháp: Rewriter — Prototype Thay Vì Sửa Compiler

### 7.1 Ý tưởng cốt lõi

Thay vì sửa Go compiler (cực kỳ phức tạp và tốn kém), Robert Griesemer xây dựng một **rewriter** — chương trình tự động chuyển đổi code từ "Go mở rộng" sang "Go chuẩn".

```
╔═══════════════════════════════════════════════════════════════╗
║   REWRITER — PROTOTYPE CỦA LANGUAGE CHANGE                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Ý tưởng:                                                     ║
║                                                               ║
║  Code "Go mở rộng"            Code Go chuẩn                  ║
║  (chưa hợp lệ)         →     (hợp lệ, compile được)        ║
║                                                               ║
║  ┌───────────────┐     Rewriter     ┌───────────────┐         ║
║  │ m[i, j] = 5.0 │ ──────────────►  │ m.AtSet(i,j,  │         ║
║  │               │                  │   5.0)        │         ║
║  │ val := m[i,j] │ ──────────────►  │ val := m.At(  │         ║
║  │               │                  │   i, j)       │         ║
║  │ a + b         │ ──────────────►  │ a.Add(b)      │         ║
║  └───────────────┘                  └───────────────┘         ║
║                                                               ║
║  LỢI ÍCH:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ✅ Không cần sửa compiler!                   │             ║
║  │ ✅ Rẻ — chỉ cần ~20-30 dòng thay đổi        │             ║
║  │ ✅ Có thể "chơi" với thiết kế MỚI            │             ║
║  │ ✅ Nếu sai → thay đổi prototype, không ảnh   │             ║
║  │    hưởng gì đến Go chính thức                │             ║
║  │ ✅ Phát hiện vấn đề TRƯỚC KHI commit          │             ║
║  │    vào compiler!                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §8. Thiết Kế Prototype: Indexed Getter/Setter & Operator Methods

### 8.1 Ba thành phần thiết kế

Robert Griesemer thiết kế prototype với 3 thành phần chính:

```
╔═══════════════════════════════════════════════════════════════╗
║   THIẾT KẾ PROTOTYPE: 3 THÀNH PHẦN                           ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Thành phần 1: Method names đặc biệt                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ // Indexed getter: toán tử []                │             ║
║  │ func (m Matrix) [](i, j int) float64         │             ║
║  │ // → Cho phép: val := m[i, j]                │             ║
║  │                                              │             ║
║  │ // Indexed setter: toán tử []=                │             ║
║  │ func (m *Matrix) []=(i, j int, val float64)  │             ║
║  │ // → Cho phép: m[i, j] = 5.0                │             ║
║  │                                              │             ║
║  │ // Plus operator: toán tử +                   │             ║
║  │ func (p Point) +(other Point) Point           │             ║
║  │ // → Cho phép: result := a + b               │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Thành phần 2: Nhiều index trong index expression            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ m[i, j]     // 2 chỉ số                     │             ║
║  │ t[i, j, k]  // 3 chỉ số                     │             ║
║  │ // → Go hiện tại CHỈ cho phép 1 chỉ số      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Thành phần 3: Quy tắc rewrite                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ NẾU x có type implement method "[]":         │             ║
║  │   x[i, j] → x.[](i, j) → x.at__(i, j)     │             ║
║  │                                              │             ║
║  │ NẾU x có type implement method "+":          │             ║
║  │   x + y   → x.+(y)     → x.add__(y)        │             ║
║  │                                              │             ║
║  │ → Phần QUAN TRỌNG NHẤT: chỉ rewrite khi    │             ║
║  │   type của x THỰC SỰ implement method!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Ví dụ cụ thể: Rewrite toán tử +

```go
// ═══ TRƯỚC KHI REWRITE (Go mở rộng, không hợp lệ) ═══

type Point struct {
    X, Y float64
}

// Method name đặc biệt: toán tử +
func (p Point) +(other Point) Point {
    return Point{p.X + other.X, p.Y + other.Y}
}

func main() {
    a := Point{1, 2}
    b := Point{3, 4}
    c := a + b  // Dùng toán tử + cho Point
}

// ═══ SAU KHI REWRITE (Go chuẩn, hợp lệ) ═══

type Point struct {
    X, Y float64
}

// Method name được đổi thành identifier hợp lệ
func (p Point) add__(other Point) Point {
    return Point{p.X + other.X, p.Y + other.Y}
}

func main() {
    a := Point{1, 2}
    b := Point{3, 4}
    c := a.add__(b)  // Binary expression → method call
}
```

---

## §9. Hiện Thực Hóa: Parser → AST → Printer

### 9.1 Pipeline xử lý

Go standard library cung cấp TẤT CẢ công cụ cần thiết!

```
╔═══════════════════════════════════════════════════════════════╗
║   PIPELINE: SOURCE → AST → MODIFIED AST → SOURCE             ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Bước 1: go/parser                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Source code (text) ──► Syntax Tree (AST)     │             ║
║  │                                              │             ║
║  │ "c := a + b"  ──►  BinaryExpr {              │             ║
║  │                       Op: +                   │             ║
║  │                       X: Ident{Name: "a"}    │             ║
║  │                       Y: Ident{Name: "b"}    │             ║
║  │                     }                        │             ║
║  └──────────────────────────────────────────────┘             ║
║                   │                                           ║
║                   ▼                                           ║
║  Bước 2: MODIFY AST (phần rewriter viết)                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ BinaryExpr{+, a, b}  ──►  CallExpr{         │             ║
║  │                              Fun: a.add__    │             ║
║  │                              Args: [b]       │             ║
║  │                            }                 │             ║
║  └──────────────────────────────────────────────┘             ║
║                   │                                           ║
║                   ▼                                           ║
║  Bước 3: go/printer                                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Modified AST  ──►  New source code (text)    │             ║
║  │                                              │             ║
║  │ CallExpr{...}  ──►  "c := a.add__(b)"       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  THAY ĐỔI CẦN THIẾT cho parser & printer:                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ • Parser: chấp nhận operator method names    │             ║
║  │   (func (p Point) +(...)) và multi-index     │             ║
║  │ • Printer: in đẹp các syntax mới            │             ║
║  │                                              │             ║
║  │ → Chỉ ~20-30 dòng code thay đổi!            │             ║
║  │ → RẤT NHỎ so với sửa compiler!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 9.2 Go Standard Library — Công cụ mạnh mẽ

```
    ┌──────────────────────────────────────────────────────────┐
    │  GO STANDARD LIBRARY CHO CODE ANALYSIS                   │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  Package          │ Chức năng                           │
    │  ─────────────────┼─────────────────────────────────────│
    │  go/token          │ Tokens (keywords, operators, etc)   │
    │  go/scanner        │ Chuyển text → tokens               │
    │  go/parser         │ Chuyển tokens → AST                │
    │  go/ast            │ Cấu trúc dữ liệu cho AST          │
    │  go/types          │ Type checker (khổng lồ!)            │
    │  go/printer        │ Chuyển AST → text (like gofmt)     │
    │  go/format         │ Format code chuẩn                  │
    │                                                          │
    │  → Đây là lý do Go TUYỆT VỜI cho prototyping!          │
    │  → Standard library cho bạn TOÀN BỘ pipeline!          │
    │  → Không cần thư viện bên ngoài!                        │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

### 9.3 Rewrite trên AST, KHÔNG phải trên text

```
    ┌──────────────────────────────────────────────────────────┐
    │  TẠI SAO REWRITE TRÊN AST?                               │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ❌ Rewrite trên TEXT (string manipulation):             │
    │  ┌──────────────────────────────────────────┐            │
    │  │ • Dễ sai vì không hiểu cấu trúc code    │            │
    │  │ • "a + b" trong string literal → KHÔNG   │            │
    │  │   nên rewrite!                           │            │
    │  │ • "a + b" trong comment → KHÔNG nên       │            │
    │  │   rewrite!                               │            │
    │  │ • Rất khó xử lý nested expressions       │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  ✅ Rewrite trên AST (abstract syntax tree):             │
    │  ┌──────────────────────────────────────────┐            │
    │  │ • AST chỉ chứa CẤU TRÚC code thật       │            │
    │  │ • Strings, comments ĐÃ ĐƯỢC tách riêng  │            │
    │  │ • Biết chính xác đâu là BinaryExpr       │            │
    │  │ • Biết chính xác đâu là IndexExpr        │            │
    │  │ • Nested expressions = nested nodes      │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  → AST = bản đồ CÓ CẤU TRÚC của source code!           │
    │  → Thao tác trên AST = AN TOÀN và CHÍNH XÁC!           │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

---

## §10. Type Checker — Cứu Tinh Của Prototype

### 10.1 Vấn đề: Rewrite đâu, bỏ đâu?

Không phải MỌI phép `+` đều cần rewrite! Chỉ rewrite khi left operand có type implement method `+`. Làm sao biết?

```
╔═══════════════════════════════════════════════════════════════╗
║   TYPE CHECKER — BIẾT CẦN REWRITE Ở ĐÂU                      ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Vấn đề:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ a := 5                                       │             ║
║  │ b := 3                                       │             ║
║  │ c := a + b     // int + int → KHÔNG rewrite! │             ║
║  │                                              │             ║
║  │ p := Point{1,2}                              │             ║
║  │ q := Point{3,4}                              │             ║
║  │ r := p + q     // Point + Point → CẦN rewrite│            ║
║  │                // vì Point có method +()     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Giải pháp: Dùng go/types (type checker)                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Parse source → AST                        │             ║
║  │ 2. Chạy type checker trên AST                │             ║
║  │    → Xác định TYPE của mỗi expression       │             ║
║  │ 3. Với mỗi BinaryExpr (a + b):              │             ║
║  │    → Kiểm tra: type(a) có method "+"?       │             ║
║  │    → NẾU CÓ: rewrite a + b → a.add__(b)    │             ║
║  │    → NẾU KHÔNG: giữ nguyên                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TRICK THÔNG MINH:                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Type checker hiện tại KHÔNG HIỂU "+"        │             ║
║  │ cho custom types → nó BÁO LỖI!             │             ║
║  │                                              │             ║
║  │ → Griesemer dùng CHÍNH lỗi này làm tín hiệu│             ║
║  │ → "Ở đâu có type error cho +,               │             ║
║  │    ở đó cần rewrite!"                       │             ║
║  │                                              │             ║
║  │ → KHÔNG CẦN sửa type checker!               │             ║
║  │ → Prototype giữ càng đơn giản càng tốt!     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 10.2 Triết lý Prototype: Giả định thông minh

```
    ┌──────────────────────────────────────────────────────────┐
    │  TRIẾT LÝ: "THIS IS A PROTOTYPE. KEEP IT CHEAP."        │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  Type checker (go/types) là thư viện                     │
    │  PHỨC TẠP NHẤT trong Go standard library.               │
    │                                                          │
    │  "It's a complex beast. We really don't want to touch    │
    │   it." — Robert Griesemer                                │
    │                                                          │
    │  Thay vì SỬA type checker:                               │
    │  ┌──────────────────────────────────────────┐            │
    │  │ Giả định: Nếu type checker báo lỗi      │            │
    │  │ ở phép toán nào đó → có thể do phép toán│            │
    │  │ đó CẦN ĐƯỢC rewrite!                     │            │
    │  │                                          │            │
    │  │ → Giả định đơn giản                      │            │
    │  │ → Hoạt động rất tốt trong THỰC TẾ       │            │
    │  │ → Tiết kiệm HÀNG NGÀN dòng code thay đổi│           │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  BÀI HỌC cho Senior Developer:                           │
    │  → Prototype KHÔNG CẦN hoàn hảo!                        │
    │  → Chỉ cần "good enough" để khám phá thiết kế!         │
    │  → Giả định thông minh > Implementation phức tạp        │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

---

## §11. Vòng Lặp Rewrite: Parse → Check → Rewrite → Repeat

### 11.1 Thuật toán lặp

Đây là phần CỐT LÕI của prototype — một vòng lặp đơn giản nhưng rất hiệu quả:

```
╔═══════════════════════════════════════════════════════════════╗
║   VÒNG LẶP REWRITE — THUẬT TOÁN CỐT LÕI                     ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Thuật toán:                                                   ║
║                                                               ║
║  ┌──────────────┐                                             ║
║  │ 1. PARSE     │── Source code → AST                         ║
║  └──────┬───────┘                                             ║
║         │                                                     ║
║         ▼                                                     ║
║  ┌──────────────┐                                             ║
║  │ 2. TYPE CHECK│── Xác định types, phát hiện lỗi            ║
║  └──────┬───────┘                                             ║
║         │                                                     ║
║         ▼                                                     ║
║  ┌──────────────┐     Có lỗi?                                ║
║  │ 3. CÓ LỖI?  │────────────────────┐                        ║
║  └──────┬───────┘                    │                        ║
║         │ Không                      │ Có                     ║
║         ▼                            ▼                        ║
║  ┌──────────────┐         ┌──────────────────┐                ║
║  │ 4. XONG!     │         │ 4. REWRITE       │                ║
║  │ → In ra code │         │ Tìm expressions  │                ║
║  │    hợp lệ    │         │ có lỗi type →    │                ║
║  └──────────────┘         │ chuyển thành     │                ║
║                           │ method calls     │                ║
║                           └────────┬─────────┘                ║
║                                    │                          ║
║                                    │ Quay lại bước 2          ║
║                                    │                          ║
║                                    ▼                          ║
║                           Lặp lại TYPE CHECK                  ║
║                           trên AST đã sửa...                 ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 11.2 Ví dụ chi tiết: Rewrite `x + y + z`

```
    ┌──────────────────────────────────────────────────────────┐
    │  VÍ DỤ: REWRITE x + y + z (tất cả là Point type)       │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ═══ VÒNG 1 ═══                                          │
    │                                                          │
    │  Sau PARSE:                                              │
    │                   BinaryExpr(+)                           │
    │                   /           \                           │
    │          BinaryExpr(+)        z: Point                   │
    │          /          \            (type OK)                │
    │     x: Point      y: Point                               │
    │     (type OK)     (type OK)                               │
    │                                                          │
    │  Sau TYPE CHECK:                                         │
    │  → x+y: LỖI! (Point không hỗ trợ +)                    │
    │  → (x+y)+z: LỖI! (type không xác định)                 │
    │                                                          │
    │  REWRITE x + y → x.add__(y):                            │
    │                   BinaryExpr(+)                           │
    │                   /           \                           │
    │          CallExpr              z: Point                   │
    │          x.add__(y)            (type OK)                  │
    │          (type: Point)                                    │
    │                                                          │
    │  ═══ VÒNG 2 ═══                                          │
    │                                                          │
    │  TYPE CHECK lại:                                         │
    │  → x.add__(y): OK! (trả về Point)                       │
    │  → Point + z: LỖI! (Point không hỗ trợ +)              │
    │                                                          │
    │  REWRITE (x.add__(y)) + z → (x.add__(y)).add__(z):      │
    │                   CallExpr                                │
    │                   (x.add__(y)).add__(z)                   │
    │                   type: Point ← OK!                      │
    │                                                          │
    │  ═══ VÒNG 3 ═══                                          │
    │                                                          │
    │  TYPE CHECK lần cuối:                                    │
    │  → KHÔNG CÒN LỖI! → XONG!                              │
    │                                                          │
    │  KẾT QUẢ:                                                │
    │  "x + y + z" → "x.add__(y).add__(z)"                    │
    │                                                          │
    │  → Compile được! Run được!                               │
    │  → Trong khi code gốc KHÔNG compile!                    │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

### 11.3 Pseudocode Go

```go
// Pseudocode cho core rewriter
func rewrite(src []byte) ([]byte, error) {
    // Bước 1: Parse
    fset := token.NewFileSet()
    file, err := parser.ParseFile(fset, "input.go", src, 0)
    if err != nil {
        return nil, err
    }

    for {
        // Bước 2: Type check
        conf := types.Config{}
        info := &types.Info{Types: make(map[ast.Expr]types.TypeAndValue)}
        _, errs := conf.Check("main", fset, []*ast.File{file}, info)

        if len(errs) == 0 {
            break // Không còn lỗi → XONG!
        }

        // Bước 3: Tìm & rewrite expressions có lỗi type
        changed := false
        ast.Inspect(file, func(n ast.Node) bool {
            if binExpr, ok := n.(*ast.BinaryExpr); ok {
                if hasOperatorMethod(binExpr.X, binExpr.Op, info) {
                    // Rewrite: a + b → a.add__(b)
                    rewriteBinaryToCall(binExpr)
                    changed = true
                }
            }
            return true
        })

        if !changed {
            return nil, fmt.Errorf("cannot resolve remaining errors")
        }
    }

    // Bước 4: Print kết quả
    var buf bytes.Buffer
    printer.Fprint(&buf, fset, file)
    return buf.Bytes(), nil
}
```

---

## §12. Ứng Dụng Thực Tế: Matrix Type & Matrix Multiplication

### 12.1 Định nghĩa Matrix type — 2D "slice"

Với prototype rewriter, ta có thể định nghĩa matrix type giống như **slice descriptor mở rộng**:

```go
// Matrix — "2D slice" descriptor
type Matrix struct {
    data    []float64  // Underlying 1D array (giống slice.array)
    rows    int        // Số hàng
    cols    int        // Số cột
    strideR int        // Bước nhảy giữa các hàng
    strideC int        // Bước nhảy giữa các cột
}

// So sánh với Slice descriptor thông thường:
//
//   Slice header:          Matrix header:
//   ┌─────────────┐       ┌─────────────┐
//   │ ptr → array │       │ data → []f64│
//   │ len         │       │ rows        │
//   │ cap         │       │ cols        │
//   └─────────────┘       │ strideR     │
//                         │ strideC     │
//                         └─────────────┘
//
// → Matrix = "2D version" của slice!
// → strideR, strideC cho phép sub-matrix views!
```

### 12.2 Indexed getter & setter

```go
// Indexed getter: m[i, j] → m.at__(i, j)
func (m Matrix) [](i, j int) float64 {
    return m.data[i*m.strideR + j*m.strideC]
}

// Indexed setter: m[i, j] = val → m.atset__(i, j, val)
func (m *Matrix) []=(i, j int, val float64) {
    m.data[i*m.strideR + j*m.strideC] = val
}
```

### 12.3 Matrix Multiplication — Cú pháp đẹp!

```
╔═══════════════════════════════════════════════════════════════╗
║   MATRIX MULTIPLICATION — SO SÁNH CÚ PHÁP                    ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ★ CÚ PHÁP MỚI (Go mở rộng):                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ // Core of textbook matrix multiplication     │             ║
║  │ for i := 0; i < m; i++ {                     │             ║
║  │     for j := 0; j < p; j++ {                 │             ║
║  │         var sum float64                      │             ║
║  │         for k := 0; k < n; k++ {             │             ║
║  │             sum += A[i, k] * B[k, j]         │             ║
║  │         }                                    │             ║
║  │         C[i, j] = sum                        │             ║
║  │     }                                        │             ║
║  │ }                                            │             ║
║  │                                              │             ║
║  │ → Giống TOÁN HỌC! Tự nhiên, dễ đọc!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  → SAU KHI REWRITE (Go chuẩn):                              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ for i := 0; i < m; i++ {                     │             ║
║  │     for j := 0; j < p; j++ {                 │             ║
║  │         var sum float64                      │             ║
║  │         for k := 0; k < n; k++ {             │             ║
║  │             sum += A.at__(i, k) * B.at__(k, j)│            ║
║  │         }                                    │             ║
║  │         C.atset__(i, j, sum)                 │             ║
║  │     }                                        │             ║
║  │ }                                            │             ║
║  │                                              │             ║
║  │ → Compile được! Run được! Performance OK!    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  → Code trên cùng = code dưới sau rewrite                   ║
║  → Viết code ĐẸP, chạy code THẬT!                           ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §13. Bài Học Bất Ngờ Từ Prototype — "The Unexpected"

### 13.1 Điều không lường trước

Robert Griesemer nhấn mạnh: Khi xây prototype, BẠN SẼ gặp những điều **BẤT NGỜ** — những câu hỏi mà bạn **KHÔNG BIẾT cần hỏi** trước khi bắt tay vào xây dựng!

```
╔═══════════════════════════════════════════════════════════════╗
║   "THE UNEXPECTED" — ĐIỀU BẤT NGỜ TỪ PROTOTYPE              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  "During the implementation of the prototype, one will       ║
║   invariably encounter THE UNEXPECTED."                      ║
║                                                               ║
║  Nghĩa là:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Khi build prototype, bạn CHẮC CHẮN sẽ gặp  │             ║
║  │ những câu hỏi bạn KHÔNG BIẾT cần hỏi       │             ║
║  │ TRƯỚC KHI bắt đầu!                          │             ║
║  │                                              │             ║
║  │ Với design document: câu hỏi này CHỈ xuất  │             ║
║  │ hiện khi implement THẬT → quá muộn, quá đắt│             ║
║  │                                              │             ║
║  │ Với prototype: câu hỏi xuất hiện SỚM       │             ║
║  │ → có thể thay đổi thiết kế DỄ DÀNG!        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Bất ngờ CỤ THỂ trong ví dụ:                                ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Robert phát hiện: index operators (syntactic │             ║
║  │ sugar) ĐƠN GIẢN và RẺ đến nỗi:            │             ║
║  │                                              │             ║
║  │ "Is it really MAYBE ALL that we need?"       │             ║
║  │                                              │             ║
║  │ → Có KHẢ NĂNG chỉ cần thêm syntax sugar    │             ║
║  │   cho indexing, KHÔNG CẦN thay đổi gì lớn! │             ║
║  │ → Câu hỏi này CHƯA BAO GIỜ được đặt ra    │             ║
║  │   trong design document!                     │             ║
║  │ → Chỉ phát hiện NHỜN prototype!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Sơ đồ so sánh:                                               ║
║                                                               ║
║  KHÔNG CÓ prototype:                                         ║
║  ┌────────┐   ┌──────────┐   ┌──────────┐                    ║
║  │ Design │──►│Implement │──►│ Surprise!│                    ║
║  │ Doc    │   │ (tốn kém)│   │ (quá muộn│                    ║
║  └────────┘   └──────────┘   │  để sửa!)│                    ║
║                               └──────────┘                    ║
║                                                               ║
║  CÓ prototype:                                                ║
║  ┌────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    ║
║  │ Design │──►│Prototype │──►│ Surprise!│──►│ Sửa     │    ║
║  │ Doc    │   │ (rẻ!)    │   │ (phát hiện│  │ thiết kế│    ║
║  └────────┘   └──────────┘   │  sớm!)   │   │ (rẻ!)   │    ║
║                               └──────────┘   └──────────┘    ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §14. Triết Lý Go & Prototyping

### 14.1 Ba kết luận của Robert Griesemer

```
╔═══════════════════════════════════════════════════════════════╗
║   BA KẾT LUẬN TỪ ROBERT GRIESEMER                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  KẾT LUẬN 1: Go = Ngôn ngữ TUYỆT VỜI cho Prototyping       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Go có standard library mạnh mẽ cho code   │             ║
║  │   analysis: parser, ast, types, printer      │             ║
║  │ → Go compile NHANH → feedback loop nhanh    │             ║
║  │ → Go đơn giản → prototype đơn giản          │             ║
║  │ → "Go has brought back the fun to            │             ║
║  │    programming"                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  KẾT LUẬN 2: Prototyping = Con đường đến thiết kế TỐT      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → ĐỪNG chỉ suy nghĩ → hãy XÂY DỰNG        │             ║
║  │ → Prototype phát hiện vấn đề trước design   │             ║
║  │   document thuần túy                         │             ║
║  │ → Iterate: build → test → refine → repeat  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  KẾT LUẬN 3: Nếu prototype được LANGUAGE CHANGES...         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Thì prototype được BẤT CỨ THỨ GÌ!       │             ║
║  │ → "If we can prototype even language         │             ║
║  │    changes, we can prototype pretty much     │             ║
║  │    everything."                              │             ║
║  │ → KHÔNG CÓ DỰ ÁN NÀO quá lớn để prototype!│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TRÍCH DẪN KINH ĐIỂN:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Frederick Brooks (1975):                     │             ║
║  │                                              │             ║
║  │ "Plan to throw one away (hopefully your      │             ║
║  │  prototype); you will, anyhow."              │             ║
║  │                                              │             ║
║  │ → Hãy LÊN KẾ HOẠCH vứt bỏ 1 bản           │             ║
║  │   (hy vọng là prototype), vì dù sao bạn    │             ║
║  │   cũng SẼ VỨT BỎ 1 bản!                    │             ║
║  │                                              │             ║
║  │ → Tốt hơn: vứt prototype (rẻ)              │             ║
║  │   hơn là vứt implementation (đắt)!          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 14.2 Áp dụng cho Senior Developer hàng ngày

```
    ┌──────────────────────────────────────────────────────────┐
    │  ÁP DỤNG PROTOTYPING TRONG CÔNG VIỆC HÀNG NGÀY          │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  1. Thiết kế API mới?                                    │
    │     → Viết prototype client code TRƯỚC!                 │
    │     → Xem API "feel" như thế nào khi SỬ DỤNG           │
    │     → Sửa API design DỰA TRÊN trải nghiệm thật        │
    │                                                          │
    │  2. Kiến trúc hệ thống mới?                              │
    │     → Build "walking skeleton" trước                    │
    │     → Kết nối các components với stub implementations   │
    │     → Phát hiện vấn đề integration sớm                 │
    │                                                          │
    │  3. Database schema mới?                                  │
    │     → Viết queries prototype trước                      │
    │     → Test với data giả                                 │
    │     → Phát hiện missing indexes, bad joins sớm          │
    │                                                          │
    │  4. Refactoring lớn?                                     │
    │     → Branch riêng, thử refactor 1 module trước         │
    │     → Xem pattern có hoạt động không                    │
    │     → Nếu OK → áp dụng cho toàn bộ codebase            │
    │                                                          │
    │  5. Language/Framework evaluation?                        │
    │     → Build 1 feature nhỏ bằng framework mới           │
    │     → So sánh trải nghiệm thực tế                       │
    │     → Quyết định dựa trên prototype, không phải docs   │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

---

## §15. Tổng Kết & Câu Hỏi Phỏng Vấn Senior Golang

### 15.1 Tóm tắt toàn bộ bài học

```
╔═══════════════════════════════════════════════════════════════╗
║               TỔNG KẾT: 7 BÀI HỌC THEN CHỐT                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  1. Design document KHÔNG ĐỦ để đảm bảo thiết kế tốt       ║
║     → Cần PROTOTYPE để phát hiện vấn đề sớm                ║
║                                                               ║
║  2. "Build your way to good design" > "Think your way"       ║
║     → Xây dựng > Suy nghĩ trên giấy                        ║
║                                                               ║
║  3. Prototype KHÔNG CẦN hoàn hảo                             ║
║     → "Keep it cheap and simple"                             ║
║     → Giả định thông minh > implementation phức tạp         ║
║                                                               ║
║  4. Go standard library = CÔNG CỤ MẠNH MẼ                   ║
║     → go/parser, go/ast, go/types, go/printer               ║
║     → Prototype language changes chỉ với ~30 dòng sửa đổi  ║
║                                                               ║
║  5. Type checker = trợ thủ đắc lực                           ║
║     → Dùng type errors làm SIGNAL cho rewriting             ║
║     → Không cần sửa type checker!                            ║
║                                                               ║
║  6. Prototype phát hiện "the unexpected"                     ║
║     → Câu hỏi bạn không biết cần hỏi                       ║
║     → Chỉ xuất hiện khi XÂY DỰNG, không khi SUY NGHĨ      ║
║                                                               ║
║  7. "Plan to throw one away" — Frederick Brooks              ║
║     → Tốt hơn vứt prototype (rẻ) hơn implementation (đắt)  ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 15.2 Sơ đồ tổng kết toàn bài

```
    ┌──────────────────────────────────────────────────────────┐
    │  BẢN ĐỒ TOÀN CẢNH: PROTOTYPE YOUR DESIGN                │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  VẤN ĐỀ: Thiết kế phần mềm                             │
    │  ├── Design document (dry exercise)                     │
    │  ├── Review → Feedback → Iteration                      │
    │  └── NHƯNG: "How do we know it's any good?"             │
    │       │                                                  │
    │       ▼                                                  │
    │  GIẢI PHÁP: Prototyping                                  │
    │  ├── Stanford d.school: Ideate→Prototype→Test           │
    │  ├── "Build your way to good design"                    │
    │  └── Dùng trong MỌI ngành: quần áo, xe, kiến trúc      │
    │       │                                                  │
    │       ▼                                                  │
    │  VÍ DỤ CỤ THỂ: Multi-dimensional slices cho Go         │
    │  ├── Go có arrays (2D, static) & slices (1D, dynamic)   │
    │  ├── THIẾU: 2D + dynamic + nice notation                │
    │  └── 3 lựa chọn: ignore / sửa compiler / REWRITER      │
    │       │                                                  │
    │       ▼                                                  │
    │  IMPLEMENTATION: Rewriter prototype                      │
    │  ├── Operator methods: +, [], []=                       │
    │  ├── Pipeline: go/parser → go/ast → go/types→go/printer │
    │  ├── Trick: dùng type errors làm rewrite signal         │
    │  └── Iterative: parse→check→rewrite→repeat→done!        │
    │       │                                                  │
    │       ▼                                                  │
    │  KẾT QUẢ: Matrix multiplication với cú pháp đẹp!       │
    │  └── BẤT NGỜ: "Có lẽ syntax sugar là ĐỦ?"              │
    │       │                                                  │
    │       ▼                                                  │
    │  BÀI HỌC chung:                                         │
    │  ├── Go = tuyệt vời cho prototyping                     │
    │  ├── Prototype bất cứ thứ gì, kể cả language changes   │
    │  └── "Plan to throw one away" — F. Brooks               │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

### 15.3 Câu hỏi phỏng vấn — Cấp độ Junior → Senior

---

**Q1: Prototyping trong software design là gì? Tại sao quan trọng?**

> **Trả lời mẫu:**
> Prototyping là quá trình xây dựng phiên bản ĐƠN GIẢN HÓA của sản phẩm để khám phá và kiểm chứng thiết kế TRƯỚC KHI implement đầy đủ.
>
> Quan trọng vì 3 lý do:
>
> 1. **Phát hiện lỗi sớm**: Lỗi ở giai đoạn design rẻ hơn NHIỀU lần so với production. Prototype giúp phát hiện vấn đề mà design document thuần túy bỏ sót.
> 2. **Khám phá "the unexpected"**: Khi build, ta gặp câu hỏi KHÔNG BIẾT CẦN HỎI trước đó. Gặp sớm ở prototype (rẻ) tốt hơn gặp ở implementation (đắt).
> 3. **Iterate nhanh**: Thay đổi prototype dễ hơn thay đổi implementation. "Build your way to good design" thay vì "think your way."

---

**Q2: Go standard library có gì hỗ trợ code analysis? Giải thích pipeline xử lý.**

> **Trả lời mẫu:**
> Go cung cấp bộ công cụ hoàn chỉnh trong standard library:
>
> - `go/token`: Định nghĩa tokens (keywords, operators, identifiers)
> - `go/scanner`: Lexical analysis — chuyển source text thành tokens
> - `go/parser`: Syntax analysis — chuyển tokens thành AST (Abstract Syntax Tree)
> - `go/ast`: Định nghĩa cấu trúc dữ liệu cho syntax tree nodes
> - `go/types`: Type checking — xác định kiểu của mọi expression
> - `go/printer`: Code generation — chuyển AST thành source text (như gofmt)
>
> Pipeline: Source → Scanner → Parser → AST → Type Checker → (modify AST) → Printer → New Source
>
> Đây là lý do Go tuyệt vời cho code analysis tools, linters, formatters, và prototyping language features.

---

**Q3: AST là gì? Tại sao rewrite trên AST thay vì string manipulation?**

> **Trả lời mẫu:**
> AST (Abstract Syntax Tree) là biểu diễn CẤU TRÚC của source code dưới dạng cây, trong đó mỗi node đại diện cho một construct trong ngôn ngữ (expression, statement, declaration...).
>
> Rewrite trên AST tốt hơn string manipulation vì:
>
> 1. **An toàn**: AST chỉ chứa cấu trúc code thật. Strings trong string literals và comments đã được tách riêng, không bị rewrite nhầm.
> 2. **Chính xác**: Biết chính xác đâu là BinaryExpr, IndexExpr, CallExpr. Không phải đoán từ text.
> 3. **Nested**: Expressions lồng nhau = nested nodes. Xử lý tự nhiên qua tree traversal.
> 4. **Type-aware**: Kết hợp với type checker, biết TYPE của mỗi expression → rewrite có điều kiện.

---

**Q4: Giải thích cách Robert Griesemer dùng type errors làm rewrite signal.**

> **Trả lời mẫu:**
> Đây là một trick thông minh trong prototype design. Thay vì sửa type checker (thư viện phức tạp nhất trong Go stdlib), Griesemer tận dụng lỗi type checking hiện có.
>
> Khi type checker gặp `a + b` mà `a` là custom type (ví dụ Point), nó sẽ báo lỗi vì Go không hỗ trợ operator `+` cho custom types.
>
> Griesemer dùng CHÍNH những lỗi này làm tín hiệu: "Ở đâu có type error cho operator, ở đó cần rewrite thành method call."
>
> Quy trình lặp: Parse → Type check → Có lỗi? → Rewrite expressions bị lỗi → Type check lại → Lặp cho đến hết lỗi → Xong!
>
> Bài học: Prototype không cần hoàn hảo. Giả định thông minh giúp tiết kiệm hàng ngàn dòng code thay đổi.

---

**Q5: "Plan to throw one away" (Frederick Brooks) nghĩa là gì? Áp dụng trong Go projects ra sao?**

> **Trả lời mẫu:**
> Frederick Brooks viết trong "The Mythical Man-Month" (1975): "Plan to throw one away; you will, anyhow."
>
> Nghĩa là: Trong bất kỳ dự án phần mềm nào, phiên bản đầu tiên luôn có vấn đề và PHẢI bỏ đi. Tốt hơn là LÊN KẾ HOẠCH cho điều này — xây prototype rẻ tiên để vứt bỏ, thay vì vứt bỏ implementation đắt đỏ.
>
> Áp dụng trong Go:
>
> - **API design**: Viết prototype client code trước, thử dùng API, rồi mới finalize
> - **Database schema**: Viết prototype queries, test với fake data trước
> - **Architecture**: Build "walking skeleton" — kết nối components với stub implementations
> - **Refactoring**: Branch riêng, thử pattern trên 1 module trước khi áp dụng toàn bộ
> - **Performance**: Benchmark prototype trước khi optimize

---

**Q6: Hãy so sánh 3 cách tiếp cận khi muốn thêm tính năng mới vào Go: ignore, sửa compiler, prototype.**

> **Trả lời mẫu:**
>
> | Tiêu chí         | Ignore          | Sửa Compiler            | Prototype/Rewriter     |
> | ---------------- | --------------- | ----------------------- | ---------------------- |
> | Chi phí          | Zero            | Rất cao                 | Thấp                   |
> | Rủi ro           | Mất user        | Sai thiết kế = phí công | Thấp — vứt đi được     |
> | Feedback         | Không có        | Muộn (sau khi commit)   | Sớm (trước khi commit) |
> | Reversibility    | N/A             | Rất khó revert          | Dễ thay đổi/vứt bỏ     |
> | "The Unexpected" | Không phát hiện | Phát hiện muộn          | Phát hiện SỚM          |
>
> Khi đánh giá tính năng mới (feature proposal), Senior developer nên: Prototype trước → Test → Thu thập feedback → Rồi mới quyết định có implement vào compiler/core không.

---

**Q7: Khi nào bạn nên prototype? Khi nào không cần?**

> **Trả lời mẫu:**
> **Nên prototype khi:**
>
> - Thay đổi kiến trúc lớn (architectural changes)
> - API design mới chưa rõ ràng
> - Tích hợp hệ thống bên ngoài (external integration)
> - Có nhiều unknowns — "chúng ta không biết những gì không biết"
> - Chi phí sai lầm cao (critical systems)
>
> **Không cần prototype khi:**
>
> - Thay đổi nhỏ, well-understood
> - Pattern đã được chứng minh trong codebase
> - Bug fix đơn giản
> - CRUD operations thông thường
>
> Quy tắc: uncertainty CAO → prototype. Uncertainty THẤP → implement trực tiếp.

---

**Q8: Tình huống phỏng vấn — "Team của bạn muốn chuyển từ REST sang gRPC. Bạn sẽ tiếp cận như thế nào?"**

> **Trả lời mẫu:**
> Tôi sẽ áp dụng prototyping approach:
>
> **Bước 1: Prototype**
>
> - Chọn 1 service nhỏ nhất (ít endpoints nhất)
> - Implement song song: giữ REST, thêm gRPC
> - Viết prototype clients cho cả hai
>
> **Bước 2: Test & Đánh giá**
>
> - Benchmark: latency, throughput, binary size
> - Developer experience: code generation, error handling
> - Operations: monitoring, debugging, deployment
>
> **Bước 3: "The Unexpected"**
>
> - Phát hiện vấn đề chỉ thấy khi build: load balancer config, browser compatibility, streaming complexity...
>
> **Bước 4: Quyết định**
>
> - Dựa trên DATA từ prototype, KHÔNG phải ý kiến chủ quan
> - Nếu OK → migration plan cho toàn bộ services
> - Nếu không → "Plan to throw one away" — vứt prototype, giữ REST
>
> → Quan trọng: quyết định dựa trên TRẢI NGHIỆM THỰC TẾ từ prototype!

---

### 15.4 Bảng tham chiếu nhanh

| Khái niệm           | Giải thích                             | Ứng dụng Go           |
| ------------------- | -------------------------------------- | --------------------- |
| Design Document     | Đặc tả phần mềm trên giấy              | RFC, proposal         |
| Prototyping         | Xây mẫu để thử nghiệm                  | Build before finalize |
| Design Thinking     | Empathize→Define→Ideate→Prototype→Test | User-first design     |
| Rewriter            | Chuyển đổi code tự động                | go/parser + go/ast    |
| AST                 | Cây cú pháp trừu tượng                 | go/ast package        |
| Type Checker        | Kiểm tra kiểu                          | go/types package      |
| Iterative Rewriting | Parse→Check→Rewrite→Repeat             | Fix-point algorithm   |
| Syntactic Sugar     | Cú pháp đẹp, cùng ngữ nghĩa            | Operator overloading  |
| "The Unexpected"    | Điều bất ngờ chỉ thấy khi build        | Unknown unknowns      |
| "Throw one away"    | Vứt bỏ prototype                       | Frederick Brooks 1975 |

---

> _"Plan to throw one away; you will, anyhow."_
> — Frederick Brooks, The Mythical Man-Month (1975)
>
> → **Prototype là bản mà bạn DỰ ĐỊNH vứt bỏ — để bản implementation THẬT sự tốt hơn!**

---

_Tài liệu được biên soạn từ bài nói chuyện "Prototype Your Design!" của Robert Griesemer tại GopherCon, kết hợp với kiến thức thiết kế phần mềm và Go standard library._
