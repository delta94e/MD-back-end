# Ratio of Bugs Per Line of Code — Tỷ Lệ Bug Theo Dòng Code

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Ratio of bugs per line of code" của Dan Mayer + dữ liệu từ Steve McConnell (Code Complete)
> **Góc nhìn:** Tại sao ít code = ít bug, và ảnh hưởng đến thiết kế Go services
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                       | Mô tả                                     |
| --- | -------------------------------------------- | ----------------------------------------- |
| §1  | Tiền đề: LOC & Bugs                          | Mối quan hệ giữa số dòng code và số bugs  |
| §2  | Dữ liệu McConnell (Code Complete)            | Thống kê bug/KLOC theo cấp độ quy trình   |
| §3  | Tại sao code lớn = nguy hiểm                 | Cognitive load, complexity, bug tích lũy  |
| §4  | Verbose vs Succinct — Cuộc tranh luận        | Readable nhưng dài vs ngắn gọn nhưng mạnh |
| §5  | Refactoring — Khi nào tốt, khi nào xấu       | Thêm LOC ≠ cải thiện, bớt LOC = win       |
| §6  | SOA & Microservices — Giải pháp codebase lớn | Tại sao tách nhỏ service                  |
| §7  | Áp dụng cho Go                               | Go design philosophy & write less code    |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior          | Ôn tập & thực hành                        |

---

## §1. Tiền Đề: LOC & Bugs

### 1.1 Mối quan hệ tuyến tính

```
╔═══════════════════════════════════════════════════════════════╗
║   LOC & BUGS — MỐI QUAN HỆ                                   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Dan Mayer:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "The more development I do the more I feel   │             ║
║  │  like increased Lines Of Code (LOC), NEARLY  │             ║
║  │  ALWAYS results in increased bugs."          │             ║
║  │                                              │             ║
║  │ → Càng nhiều dòng code = càng nhiều bugs!   │             ║
║  │ → Nghe hiển nhiên nhưng KHÔNG AI làm theo!  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  SƠ ĐỒ:                                                       ║
║                                                               ║
║  Số bugs                                                       ║
║    ▲                          /                               ║
║    │                        /                                 ║
║    │                      /                                   ║
║    │                    /   ← Tỷ lệ gần như                  ║
║    │                  /       TUYẾN TÍNH!                     ║
║    │                /                                         ║
║    │              /                                           ║
║    │            /                                             ║
║    │          /                                               ║
║    │        /                                                 ║
║    │      /                                                   ║
║    │    /                                                     ║
║    │  /                                                       ║
║    │/                                                         ║
║    └──────────────────────────────────► Lines of Code         ║
║                                                               ║
║  NGHỊCH LÝ:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Nhiều developer nghĩ:                         │             ║
║  │ → "Code đơn giản 600 dòng = ÍT rủi ro"     │             ║
║  │                                              │             ║
║  │ Thực tế:                                      │             ║
║  │ → Code đơn giản 600 dòng VẪN nhiều rủi ro  │             ║
║  │   hơn code phức tạp 100 dòng!               │             ║
║  │ → "Reviewing a 'simple' 600 LOC change is   │             ║
║  │   still FAR MORE RISKY than a complex        │             ║
║  │   change of 100 LOC."                        │             ║
║  │                                              │             ║
║  │ → Mỗi dòng = 1 chỗ cho bug TỒN TẠI!       │             ║
║  │ → 600 dòng = 600 chỗ cho bugs!             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Dữ Liệu McConnell (Code Complete)

### 2.1 Bug rates theo cấp độ quy trình

```
╔═══════════════════════════════════════════════════════════════╗
║   STEVE McCONNELL — CODE COMPLETE STATS                       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  ┌────────────────────────────────────────────────────┐       ║
║  │ CẤP ĐỘ          │ TESTING PHASE  │ RELEASED      │       ║
║  │                  │ (bugs/KLOC)    │ (bugs/KLOC)   │       ║
║  ├──────────────────┼────────────────┼───────────────┤       ║
║  │ (a) Industry     │ 15–50          │ (not stated)  │       ║
║  │     Average      │                │               │       ║
║  ├──────────────────┼────────────────┼───────────────┤       ║
║  │ (b) Microsoft    │ 10–20          │ 0.5           │       ║
║  │     Applications │                │               │       ║
║  ├──────────────────┼────────────────┼───────────────┤       ║
║  │ (c) Cleanroom    │ 3              │ 0.1           │       ║
║  │     Development  │                │               │       ║
║  ├──────────────────┼────────────────┼───────────────┤       ║
║  │ (d) Space        │ ~0             │ 0             │       ║
║  │     Shuttle      │                │ (500K LOC!)   │       ║
║  └──────────────────┴────────────────┴───────────────┘       ║
║                                                               ║
║  KLOC = 1000 Lines Of Code                                    ║
║                                                               ║
║  GIẢI THÍCH TỪNG CẤP:                                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ (a) INDUSTRY AVERAGE:                         │             ║
║  │    15–50 bugs / 1000 dòng code               │             ║
║  │    → Structured programming cơ bản           │             ║
║  │    → Mix các kỹ thuật coding                 │             ║
║  │    → ĐÂY LÀ MỨC PHỔ BIẾN NHẤT!            │             ║
║  │                                              │             ║
║  │ (b) MICROSOFT:                                │             ║
║  │    10–20 bugs (testing) → 0.5 bugs (release) │             ║
║  │    → Code reading + independent testing      │             ║
║  │    → Quy trình NGHIÊM NGẶT hơn             │             ║
║  │                                              │             ║
║  │ (c) CLEANROOM (Harlan Mills):                 │             ║
║  │    3 bugs (testing) → 0.1 bugs (release)     │             ║
║  │    → Formal development methods              │             ║
║  │    → Peer reviews + statistical testing      │             ║
║  │                                              │             ║
║  │ (d) SPACE SHUTTLE:                            │             ║
║  │    0 bugs trong 500,000 dòng!                │             ║
║  │    → Chi phí CỰC KỲ CAO!                   │             ║
║  │    → Không khả thi cho web apps!             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ÁP DỤNG THỰC TẾ:                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Go microservice 10,000 LOC:                   │             ║
║  │                                              │             ║
║  │ Industry average: 150–500 bugs ẩn!          │             ║
║  │ Với code review+testing: 50–200 bugs        │             ║
║  │ Với cleanroom methods: 10–30 bugs            │             ║
║  │                                              │             ║
║  │ → CÁCH ĐƠN GIẢN NHẤT giảm bugs:           │             ║
║  │   = GIẢM SỐ DÒNG CODE!                     │             ║
║  │                                              │             ║
║  │ Nếu giảm từ 10K → 5K LOC:                   │             ║
║  │ → Bugs giảm 50%! (tuyến tính!)              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Tại Sao Code Lớn = Nguy Hiểm

### 3.1 Cognitive Load & Bug tích lũy

```
╔═══════════════════════════════════════════════════════════════╗
║   CODE LỚN = NGUY HIỂM — TẠI SAO?                           ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Steve Yegge:                                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "I happen to hold a hard-won minority        │             ║
║  │  opinion about code bases. I believe, quite  │             ║
║  │  STAUNCHLY, that the WORST thing that can    │             ║
║  │  happen to a code base is SIZE."             │             ║
║  │                                              │             ║
║  │ → Điều TỆ NHẤT cho codebase = KÍCH THƯỚC!  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  3 LÝ DO CHÍNH:                                              ║
║                                                               ║
║  1. COGNITIVE LOAD (Tải nhận thức):                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ Codebase nhỏ (5K LOC):                       │             ║
║  │ ┌─────────────────────────────────┐          │             ║
║  │ │ Đọc hiểu:   2 giờ              │          │             ║
║  │ │ Thay đổi:   30 phút            │          │             ║
║  │ │ Review:     15 phút            │          │             ║
║  │ │ Confidence: CAO                 │          │             ║
║  │ └─────────────────────────────────┘          │             ║
║  │                                              │             ║
║  │ Codebase lớn (100K LOC):                     │             ║
║  │ ┌─────────────────────────────────┐          │             ║
║  │ │ Đọc hiểu:   2 TUẦN             │          │             ║
║  │ │ Thay đổi:   2 ngày (sợ break)  │          │             ║
║  │ │ Review:     2 giờ (vẫn lo)     │          │             ║
║  │ │ Confidence: THẤP                │          │             ║
║  │ └─────────────────────────────────┘          │             ║
║  │                                              │             ║
║  │ Dan Mayer: "The cognitive load associated    │             ║
║  │ with understanding all the implications of   │             ║
║  │ a change, and who might be relying on a      │             ║
║  │ specific QUIRK in existing code."            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  2. BUG TÍCH LŨY (Bug Accumulation):                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Code lớn hơn → nhiều bugs TỒN TẠI hơn     │             ║
║  │ → Code mới phải WORKAROUND bugs cũ!        │             ║
║  │ → Code mới tạo bugs MỚI!                   │             ║
║  │ → Vòng lặp ÁC TÍNH!                        │             ║
║  │                                              │             ║
║  │ t=0:   Code 5K   → ~100 bugs               │             ║
║  │ t=6m:  Code 20K  → ~400 bugs + workarounds │             ║
║  │ t=1y:  Code 50K  → ~1000 bugs + hacks      │             ║
║  │ t=2y:  Code 100K → "Viết lại từ đầu?"     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  3. CHI PHÍ TĂNG THEO CẤP SỐ NHÂN:                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ McConnell:                                    │             ║
║  │ → Chi phí thêm code KHÔNG tăng tuyến tính! │             ║
║  │ → Mà tăng theo CẤP SỐ NHÂN!               │             ║
║  │                                              │             ║
║  │ Chi phí                                       │             ║
║  │   ▲             /                             │             ║
║  │   │           /                               │             ║
║  │   │         /    ← Thực tế!                  │             ║
║  │   │       /                                   │             ║
║  │   │     / .....                               │             ║
║  │   │   / .       ← Bạn TƯỞNG                 │             ║
║  │   │  /.           (tuyến tính)               │             ║
║  │   │/.                                         │             ║
║  │   └──────────────────► Kích thước codebase   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Verbose vs Succinct — Cuộc Tranh Luận

### 4.1 Hành trình từ Verbose đến Succinct

```
╔═══════════════════════════════════════════════════════════════╗
║   VERBOSE vs SUCCINCT — HÀNH TRÌNH DEVELOPER                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Dan Mayer quan sát:                                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Many great developers start to really       │             ║
║  │  favor the MOST SUCCINCT CODE to accomplish  │             ║
║  │  their task."                                │             ║
║  │                                              │             ║
║  │ → Dev giỏi dần chuyển sang code NGẮN GỌN!  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  HÀNH TRÌNH:                                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ JUNIOR (Verbose):                             │             ║
║  │ → Big objects, reusable modules              │             ║
║  │ → Nhiều comment, nhiều abstraction           │             ║
║  │ → "Mọi thứ phải RÕ RÀNG!"                  │             ║
║  │ → 200 dòng cho function đơn giản            │             ║
║  │       │                                      │             ║
║  │       ▼                                      │             ║
║  │ MID-LEVEL (Balanced):                         │             ║
║  │ → Design patterns everywhere                 │             ║
║  │ → "Clean Code" theo sách                     │             ║
║  │ → Nhiều layers, nhiều abstraction            │             ║
║  │       │                                      │             ║
║  │       ▼                                      │             ║
║  │ SENIOR (Succinct):                            │             ║
║  │ → Simpler objects, less abstraction          │             ║
║  │ → Dense but powerful code                    │             ║
║  │ → "Nhỏ nhất có thể, không nhỏ hơn!"       │             ║
║  │ → 50 dòng cho cùng function!                │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NHƯNG CÓ TRADE-OFF:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Code ngắn gọn:                                │             ║
║  │ ✅ Ít surface area cho bugs                   │             ║
║  │ ✅ Dễ review (ít dòng)                       │             ║
║  │ ❌ Có thể KHÓ ĐỌC hơn cho người mới       │             ║
║  │                                              │             ║
║  │ Code verbose:                                 │             ║
║  │ ✅ Dễ đọc cho MỌI cấp độ                    │             ║
║  │ ❌ Nhiều surface area cho bugs               │             ║
║  │ ❌ Cognitive load CAO (nhiều code để hiểu)  │             ║
║  │                                              │             ║
║  │ → Dan Mayer: "Error on the side of LESS     │             ║
║  │   code, unless there are CLEAR improvements │             ║
║  │   to having more code."                      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Refactoring — Khi Nào Tốt, Khi Nào Xấu

### 5.1 Refactoring tăng LOC = đáng ngờ

```
╔═══════════════════════════════════════════════════════════════╗
║   REFACTORING — KHI NÀO TỐT, KHI NÀO XẤU                   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Dan Mayer:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Refactorings that introduce MORE lines of   │             ║
║  │  code for the sake of readability are often  │             ║
║  │  just MOVING THE COMPLEXITY AROUND."         │             ║
║  │                                              │             ║
║  │ → Refactor + thêm LOC = di chuyển phức tạp │             ║
║  │ → CẢM GIÁC tốt hơn, nhưng THỰC TẾ không! │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ✅ REFACTORING TỐT:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. GIẢM LOC mà giữ nguyên chức năng!       │             ║
║  │    → Xóa code trùng lặp                     │             ║
║  │    → Xóa code không dùng                     │             ║
║  │    → Đơn giản hóa logic                     │             ║
║  │    → "Big win in maintainability!"           │             ║
║  │                                              │             ║
║  │ 2. Giảm "files cần sync"                     │             ║
║  │    → Nhiều files phải "giữ đồng bộ"        │             ║
║  │      = nguồn bug thường xuyên!              │             ║
║  │    → Refactor giảm coupling = WIN!           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ❌ REFACTORING XẤU:                                         ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Thêm LOC "cho dễ đọc"                    │             ║
║  │    → 500 LOC → 800 LOC "readable hơn"       │             ║
║  │    → Thêm 300 chỗ cho bugs!                 │             ║
║  │    → Có thể CHỈ dễ đọc cho người refactor! │             ║
║  │                                              │             ║
║  │ 2. Thêm ABSTRACTION layers                   │             ║
║  │    → Mỗi method chỉ gọi method khác        │             ║
║  │    → Layer on layer trước khi đến logic!    │             ║
║  │    → "Abstractions have a cognitive load     │             ║
║  │       and each leaky abstraction layer is    │             ║
║  │       another potential buggy line of code." │             ║
║  │                                              │             ║
║  │ 3. Design patterns cho MỌI THỨ              │             ║
║  │    → "Knowing when they are OVERKILL        │             ║
║  │       takes time."                            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. SOA & Microservices — Giải Pháp Codebase Lớn

### 6.1 Tách nhỏ để giảm LOC mỗi service

```
╔═══════════════════════════════════════════════════════════════╗
║   SOA & MICROSERVICES — GIẢI PHÁP CHO CODEBASE LỚN           ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Dan Mayer:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Bugs to LOC ratio is one of the main        │             ║
║  │  reasons having a large codebase is BAD.     │             ║
║  │  This is one of the reasons why SOA,         │             ║
║  │  Micro-Services, and building mini-apps      │             ║
║  │  is the solution."                           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  MONOLITH vs MICROSERVICES (góc nhìn LOC):                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ MONOLITH 200K LOC:                            │             ║
║  │ ┌──────────────────────────────────┐         │             ║
║  │ │ Users + Orders + Payments +      │         │             ║
║  │ │ Notifications + Analytics +      │         │             ║
║  │ │ Reports + Admin + ...            │         │             ║
║  │ │                                  │         │             ║
║  │ │ Bugs: 3,000–10,000 ẩn!         │         │             ║
║  │ │ Cognitive load: CỰC CAO!       │         │             ║
║  │ │ 1 thay đổi có thể ẢNH HƯỞNG  │         │             ║
║  │ │ BẤT KỲ module nào!            │         │             ║
║  │ └──────────────────────────────────┘         │             ║
║  │       │                                      │             ║
║  │       ▼ Tách thành                           │             ║
║  │                                              │             ║
║  │ MICROSERVICES (mỗi service ~5-15K LOC):      │             ║
║  │ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐       │             ║
║  │ │Users │ │Orders│ │Pay-  │ │Noti- │       │             ║
║  │ │ 10K  │ │ 15K  │ │ments │ │fy    │       │             ║
║  │ │      │ │      │ │ 8K   │ │ 5K   │       │             ║
║  │ └──────┘ └──────┘ └──────┘ └──────┘       │             ║
║  │ ~200    ~300     ~160    ~100 bugs          │             ║
║  │                                              │             ║
║  │ Lợi ích:                                     │             ║
║  │ → Cognitive load THẤP cho mỗi service       │             ║
║  │ → Bug trong Users KHÔNG ẢNH HƯỞNG Orders  │             ║
║  │ → Team độc lập deploy + fix                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CẢNH BÁO:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ SOA/Microservices GIẢI QUYẾT vấn đề LOC    │             ║
║  │ NHƯNG TẠO RA vấn đề mới:                    │             ║
║  │ → Network complexity                          │             ║
║  │ → Distributed debugging                      │             ║
║  │ → Data consistency                            │             ║
║  │ → Deployment complexity                       │             ║
║  │                                              │             ║
║  │ → Trade-off, KHÔNG phải silver bullet!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Áp Dụng Cho Go

### 7.1 Go design philosophy = Write Less

```
╔═══════════════════════════════════════════════════════════════╗
║   GO & WRITE LESS CODE                                        ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Go THIẾT KẾ ĐỂ VIẾT ÍT:                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ✅ Unused import = COMPILE ERROR             │             ║
║  │ ✅ Unused variable = COMPILE ERROR           │             ║
║  │ ✅ gofmt: 0 thời gian tranh luận style     │             ║
║  │ ✅ Không có class inheritance → ít layers   │             ║
║  │ ✅ Không có exceptions → ít hidden flow     │             ║
║  │ ✅ Composition > Inheritance                 │             ║
║  │ ✅ Standard library CỰC MẠNH               │             ║
║  │    → Ít dependency = ít code tổng!          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GO PROVERBS LIÊN QUAN:                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "A little copying is better than a little    │             ║
║  │  dependency."                                │             ║
║  │ → Copy 20 dòng < Import library 5000 dòng! │             ║
║  │                                              │             ║
║  │ "Clear is better than clever."               │             ║
║  │ → Code rõ ràng > code thông minh!           │             ║
║  │                                              │             ║
║  │ "Don't panic."                                │             ║
║  │ → Handle error, ĐỪNG crash!                 │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 7.2 Go patterns giảm LOC

```go
// ═══ GO: PATTERNS GIẢM LOC MÀ TĂNG INTEGRITY ═══

// 1. TABLE-DRIVEN TESTS — 1 pattern, N test cases!
// ❌ TỆ: 100 dòng test copy-paste
func TestAdd1(t *testing.T) { /* ... */ }
func TestAdd2(t *testing.T) { /* ... */ }
func TestAdd3(t *testing.T) { /* ... */ }

// ✅ TỐT: 20 dòng, cover N cases!
func TestAdd(t *testing.T) {
    tests := []struct {
        a, b, want int
    }{
        {1, 2, 3},
        {0, 0, 0},
        {-1, 1, 0},
        {100, -50, 50},
    }
    for _, tt := range tests {
        got := Add(tt.a, tt.b)
        if got != tt.want {
            t.Errorf("Add(%d,%d) = %d, want %d",
                tt.a, tt.b, got, tt.want)
        }
    }
}
// → Thêm test case = thêm 1 DÒNG, không phải 1 FUNCTION!

// 2. EMBEDDING — Composition thay vì abstraction layers
// ❌ TỆ: Wrapper pattern 200 dòng
type LoggingService struct {
    inner  Service
    logger Logger
}
func (s *LoggingService) GetUser(id string) (*User, error) {
    s.logger.Info("getting user", "id", id)
    return s.inner.GetUser(id) // Chỉ delegate!
}
// → Mỗi method = wrapper → NHIỀU code!

// ✅ TỐT: Middleware pattern, 1 lần cho TẤT CẢ
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter,
        r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}
// → 1 middleware = log MỌI request!

// 3. FUNCTIONAL OPTIONS — Thay vì builder 100 dòng
type Server struct {
    addr    string
    timeout time.Duration
    maxConn int
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func WithMaxConn(n int) Option {
    return func(s *Server) { s.maxConn = n }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{addr: addr, timeout: 30 * time.Second}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
// → Extensible MÀ KHÔNG phình code!

// 4. STANDARD LIBRARY FIRST — Ít dependency!
// ❌ TỆ: import 10 libraries cho HTTP server
// ✅ TỐT: net/http + encoding/json = ĐỦ!
func main() {
    http.HandleFunc("/api/users", handleUsers)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
// → 3 dòng = production HTTP server!
// → 0 external dependencies = 0 dependency bugs!
```

---

## §8. Tổng Kết & Câu Hỏi Phỏng Vấn Senior Golang

### 8.1 Câu hỏi phỏng vấn

```
╔═══════════════════════════════════════════════════════════════╗
║   CÂU HỎI PHỎNG VẤN SENIOR GOLANG                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Q1: "Bug rate trung bình ngành là bao nhiêu?"              ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ McConnell (Code Complete):                    │             ║
║  │ → Industry: 15–50 bugs / 1000 LOC            │             ║
║  │ → Microsoft: 10–20 (test) → 0.5 (release)   │             ║
║  │ → Cleanroom: 3 (test) → 0.1 (release)       │             ║
║  │                                              │             ║
║  │ Cách giảm: viết ÍT code hơn!                │             ║
║  │ + code review + testing + linters            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Tại sao large codebase nguy hiểm?"                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Cognitive load TĂNG theo cấp số nhân     │             ║
║  │ 2. Bug tích lũy → workarounds → more bugs  │             ║
║  │ 3. Chi phí thay đổi tăng KHÔNG tuyến tính  │             ║
║  │ 4. Test coverage GIẢM (Large = incomplete)  │             ║
║  │                                              │             ║
║  │ Yegge: "The WORST thing that can happen     │             ║
║  │ to a code base is SIZE."                     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Refactoring tăng LOC có tốt không?"                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Thường KHÔNG!                                 │             ║
║  │ → "Just moving complexity around"            │             ║
║  │ → Thêm LOC = thêm chỗ cho bugs             │             ║
║  │ → Chỉ "dễ đọc" cho người refactor          │             ║
║  │                                              │             ║
║  │ Refactoring TỐT = GIẢM LOC + giữ functionality│           ║
║  │ → Xóa dead code, hợp nhất trùng lặp       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Go giúp viết ít code thế nào?"                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Unused import/var = compile error!         │             ║
║  │ 2. Standard library mạnh → ít dependency    │             ║
║  │ 3. Composition > Inheritance → ít layers    │             ║
║  │ 4. Table-driven tests → ít test code        │             ║
║  │ 5. No exceptions → ít hidden code paths     │             ║
║  │ 6. gofmt → 0 style debates                  │             ║
║  │ 7. "A little copying > a little dependency" │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: RATIO OF BUGS PER LINE OF CODE               │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. 15–50 bugs / 1000 LOC = THỰC TẾ      │            │
    │  │    → Mỗi dòng = 1 chỗ cho bug tồn tại  │            │
    │  │    → "Simple" 600 LOC > risky hơn        │            │
    │  │      complex 100 LOC!                     │            │
    │  │                                          │            │
    │  │ 2. Cách ĐƠN GIẢN NHẤT giảm bugs:        │            │
    │  │    = VIẾT ÍT CODE HƠN!                  │            │
    │  │                                          │            │
    │  │ 3. Large codebase = KẺ THÙ               │            │
    │  │    → Cognitive load tăng                  │            │
    │  │    → Chi phí tăng theo cấp số nhân      │            │
    │  │    → Bug tích lũy + workarounds          │            │
    │  │                                          │            │
    │  │ 4. Refactoring ĐÚNG = GIẢM LOC           │            │
    │  │    → Thêm LOC ≠ cải thiện                │            │
    │  │    → Xóa code > thêm code               │            │
    │  │                                          │            │
    │  │ 5. Go = ngôn ngữ của "write less"        │            │
    │  │    → Compile enforce unused removal      │            │
    │  │    → Composition > layers                 │            │
    │  │    → Stdlib > dependencies                │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  Dan Mayer:                                              │
    │  "Keep your code tiny. Fight extra complexity           │
    │   and lines of code and strike down upon it            │
    │   with great vengeance & furious anger."                │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
