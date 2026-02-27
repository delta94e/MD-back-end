# Code Must Never Lie — Code Không Bao Giờ Được Nói Dối

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Code Must Never Lie" của Nate Finch (2017)
> **Góc nhìn:** Code phải TRUNG THỰC — đọc code phải hiểu đúng code làm gì
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                              | Mô tả                                  |
| --- | ----------------------------------- | -------------------------------------- |
| §1  | Nguyên tắc: Code không được nói dối | Cardinal sin & tại sao quan trọng      |
| §2  | Câu chuyện assert vs require        | Ví dụ gốc: import alias gây hiểu lầm   |
| §3  | Code nói dối vs Code gây hiểu lầm   | Lie vs Mislead — 2 mức độ              |
| §4  | Naming — Tên phải nói sự thật       | Write() không phải io.Writer = nói dối |
| §5  | Transaction không cần thiết         | Consistency vs Clarity                 |
| §6  | Khi bắt buộc phải "dối"             | Document the hell out of it!           |
| §7  | Áp dụng cho Go codebase             | Patterns & anti-patterns trong Go      |
| §8  | Tổng kết & Câu hỏi phỏng vấn Senior | Ôn tập & thực hành                     |

---

## §1. Nguyên Tắc: Code Không Được Nói Dối

### 1.1 Cardinal Sin — Tội chết trong lập trình

```
╔═══════════════════════════════════════════════════════════════╗
║   CODE MUST NEVER LIE — NGUYÊN TẮC BẤT KHẢ XÂM PHẠM       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Mark Twain:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "If you tell the TRUTH, you don't have to    │             ║
║  │  REMEMBER anything."                         │             ║
║  │                                              │             ║
║  │ → Nói thật = không cần NHỚ gì cả!          │             ║
║  │ → Code trung thực = không cần GHI CHÚ ĐẶC  │             ║
║  │   BIỆT = ít bugs!                            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Nate Finch:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Code must NEVER lie. This is a CARDINAL SIN │             ║
║  │  amongst programmers."                       │             ║
║  │                                              │             ║
║  │ → CARDINAL SIN = TỘI CHẾT!                  │             ║
║  │ → Không có ngoại lệ!                         │             ║
║  │ → Đây là MỞ RỘNG của: "Code should be      │             ║
║  │   written to be READ."                       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CODE NÓI DỐI = GÌ?                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ Code NÓI DỐI khi:                            │             ║
║  │ → Code TRÔNG NHƯ làm việc A                 │             ║
║  │ → Nhưng THỰC TẾ làm việc B!                │             ║
║  │                                              │             ║
║  │ HẬU QUẢ:                                     │             ║
║  │ → Người đọc HIỂU SAI → dùng SAI            │             ║
║  │ → Sửa code SAI CÁCH → tạo BUGS             │             ║
║  │ → May mắn: bug NGAY LẬP TỨC, rõ ràng     │             ║
║  │ → Xui xẻo: bug TINH VI, phải debug         │             ║
║  │   HÀNG GIỜ, đập đầu vào bàn phím!         │             ║
║  │                                              │             ║
║  │ "That someone might be YOU, even if it was  │             ║
║  │  YOUR code in the first place."              │             ║
║  │ → Người bị lừa có thể CHÍNH LÀ BẠN!       │             ║
║  │ → Lúc 2 giờ sáng debug production!          │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Câu Chuyện Assert vs Require

### 2.1 Ví dụ gốc — Import alias

```
╔═══════════════════════════════════════════════════════════════╗
║   ASSERT vs REQUIRE — CÂU CHUYỆN GỐC                         ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CONTEXT (testify library trong Go):                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ assert package:                               │             ║
║  │ → Test FAIL nhưng VẪN CHẠY TIẾP!           │             ║
║  │ → "Ghi nhận lỗi, kiểm tra tiếp"            │             ║
║  │                                              │             ║
║  │ require package:                              │             ║
║  │ → Test FAIL và DỪNG NGAY LẬP TỨC!          │             ║
║  │ → "Lỗi này nghiêm trọng, dừng lại!"       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TÌNH HUỐNG:                                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Reviewer (Nate): "Đổi assert → require      │             ║
║  │ cho mấy chỗ cần dừng test ngay."            │             ║
║  │                                              │             ║
║  │ Author: "OK, mình đổi thế này cho nhanh!"   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  "GIẢI PHÁP" CỦA AUTHOR:                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │   import (                                    │             ║
║  │       assert "github.com/stretchr/           │             ║
║  │               testify/require"               │             ║
║  │   )                                           │             ║
║  │                                              │             ║
║  │ → Import require, nhưng GỌI NÓ LÀ assert!  │             ║
║  │ → Code gọi assert.Equal() nhưng bên trong  │             ║
║  │   thực tế là require.Equal()!               │             ║
║  │                                              │             ║
║  │ Nate: "HELL NO!"                           │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TẠI SAO ĐÂY LÀ NÓI DỐI?                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ Người đọc code ở DÒNG 200:                  │             ║
║  │                                              │             ║
║  │   assert.Equal(t, expected, actual)           │             ║
║  │                                              │             ║
║  │ Họ NGHĨ: "Test sẽ tiếp tục nếu fail"       │             ║
║  │ THỰC TẾ: "Test DỪNG NGAY nếu fail!"        │             ║
║  │                                              │             ║
║  │ → Viết thêm assert bên dưới → KHÔNG CHẠY! │             ║
║  │ → Test có vẻ pass nhưng thực ra SKIP phần  │             ║
║  │   quan trọng phía sau!                       │             ║
║  │ → Tests FAIL với error message KHÁC HƯỚNG!  │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Code Nói Dối vs Code Gây Hiểu Lầm

### 3.1 Lie vs Mislead — Hai mức độ

```
╔═══════════════════════════════════════════════════════════════╗
║   LIE vs MISLEAD — HAI MỨC ĐỘ                               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Mức 1: LIE (NÓI DỐI)                                       ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Code RÕ RÀNG nói điều SAI!               │             ║
║  │ → Tên hàm nói 1 đằng, làm 1 nẻo!          │             ║
║  │                                              │             ║
║  │ Ví dụ:                                        │             ║
║  │ • assert thực ra là require                  │             ║
║  │ • saveToMemory() lưu vào database AWS!      │             ║
║  │ • IsValid() thực ra CÓ side effects!        │             ║
║  │ • GetUser() thực ra TẠO user nếu chưa có!  │             ║
║  │                                              │             ║
║  │ MỨC ĐỘ: ⚠️ NGHIÊM TRỌNG — không tha thứ! │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Mức 2: MISLEAD (GÂY HIỂU LẦM)                             ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Code không SAI nhưng TAO GIẢ ĐỊNH SAI!   │             ║
║  │ → Người đọc tự SUY LUẬN sai!               │             ║
║  │                                              │             ║
║  │ Ví dụ:                                        │             ║
║  │ • Transaction cho 1 query → "Chắc phải      │             ║
║  │   save nhiều bảng" (thực ra không!)         │             ║
║  │ • Write() nhưng không phải io.Writer         │             ║
║  │ • Mutex lock ở chỗ không cần concurrent     │             ║
║  │                                              │             ║
║  │ MỨC ĐỘ: ⛔ TINH VI HƠN — khó phát hiện!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Nate Finch:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Don't lie. Furthermore, try not to even     │             ║
║  │  MISLEAD. Humans make ASSUMPTIONS all the    │             ║
║  │  time, it's built into how we perceive       │             ║
║  │  the world."                                 │             ║
║  │                                              │             ║
║  │ → Con người TỰ ĐỘNG giả định!              │             ║
║  │ → Lập trình viên PHẢI dự đoán người đọc   │             ║
║  │   sẽ giả định gì, và đảm bảo giả định     │             ║
║  │   đó KHÔNG SAI!                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TRÁCH NHIỆM CỦA PROGRAMMER:                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Dự đoán ASSUMPTIONS người đọc sẽ có     │             ║
║  │ 2. Đảm bảo assumptions KHÔNG SAI            │             ║
║  │ 3. Nếu assumptions SAI → structure code     │             ║
║  │    để DISABUSE (chỉnh lại) cho họ!          │             ║
║  │ 4. Dùng CODE để chỉnh, KHÔNG dùng comment!│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Naming — Tên Phải Nói Sự Thật

### 4.1 Write() không phải io.Writer

```
╔═══════════════════════════════════════════════════════════════╗
║   NAMING — TÊN PHẢI TRUNG THỰC                               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Nate Finch:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "If your type has a                          │             ║
║  │  Write(b []byte) (int, error)                │             ║
║  │  method that is NOT compatible with          │             ║
║  │  io.Writer, consider calling it something    │             ║
║  │  OTHER than Write — because everyone seeing  │             ║
║  │  foo.Write is going to ASSUME that function  │             ║
║  │  will work like an io.Writer."               │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VÍ DỤ MINH HỌA:                                             ║
║                                                               ║
║  ❌ NÓI DỐI:                                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ type Logger struct { /* ... */ }              │             ║
║  │                                              │             ║
║  │ // TRÔNG NHƯ io.Writer nhưng KHÔNG PHẢI!    │             ║
║  │ func (l *Logger) Write(b []byte) (int, error)│             ║
║  │ → Signature GIỐNG io.Writer                  │             ║
║  │ → Nhưng behavior KHÁC!                      │             ║
║  │ → Người dùng sẽ TRY:                       │             ║
║  │   io.Copy(logger, reader)  // → BROKEN!     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  ✅ TRUNG THỰC:                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ type Logger struct { /* ... */ }              │             ║
║  │                                              │             ║
║  │ // Tên KHÁC → không ai nhầm với io.Writer!  │             ║
║  │ func (l *Logger) WriteLog(b []byte)          │             ║
║  │ // hoặc                                      │             ║
║  │ func (l *Logger) PrintOut(b []byte)          │             ║
║  │ // hoặc                                      │             ║
║  │ func (l *Logger) Emit(b []byte)              │             ║
║  │                                              │             ║
║  │ → Tên KHÁC = không tạo giả định sai!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NGUYÊN TẮC:                                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Trong Go, TÊN có ý nghĩa CONVENTION:        │             ║
║  │                                              │             ║
║  │ Read()   → ai cũng nghĩ io.Reader           │             ║
║  │ Write()  → ai cũng nghĩ io.Writer           │             ║
║  │ String() → ai cũng nghĩ fmt.Stringer        │             ║
║  │ Error()  → ai cũng nghĩ error interface     │             ║
║  │ Close()  → ai cũng nghĩ io.Closer           │             ║
║  │ Len()    → ai cũng nghĩ trả về length      │             ║
║  │                                              │             ║
║  │ → Dùng tên này MÀ KHÔNG đúng convention     │             ║
║  │   = NÓI DỐI!                                │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Transaction Không Cần Thiết

### 5.1 Consistency vs Clarity

```
╔═══════════════════════════════════════════════════════════════╗
║   TRANSACTION KHÔNG CẦN THIẾT — GÂY HIỂU LẦM               ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  CÂU CHUYỆN:                                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Code review: Author wrap 1 DB update trong  │             ║
║  │ transaction.                                  │             ║
║  │                                              │             ║
║  │ Nate (reviewer) thấy transaction → NGHĨ:     │             ║
║  │ "Chắc phải save dữ liệu liên quan ở       │             ║
║  │  NHIỀU BẢNG, nên cần transaction!"           │             ║
║  │                                              │             ║
║  │ THỰC TẾ: Chỉ CÓ 1 update!                  │             ║
║  │ Lý do dùng transaction: "Cho consistent     │             ║
║  │ với code khác cũng dùng transaction."        │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VẤN ĐỀ:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Transaction = TÍN HIỆU mạnh cho reader:     │             ║
║  │ "Có NHIỀU operations cần ATOMIC!"            │             ║
║  │                                              │             ║
║  │ Khi dùng cho 1 operation:                     │             ║
║  │ → Reader HIỂU SAI ý định code!              │             ║
║  │ → Reader TÌM KIẾM operations khác mà       │             ║
║  │   KHÔNG TỒN TẠI!                            │             ║
║  │ → Reader THÊM operations vào transaction    │             ║
║  │   vì nghĩ "nó đã designed cho multi-op"    │             ║
║  │                                              │             ║
║  │ Nate: "Being CONSISTENT was actually        │             ║
║  │ CONFUSING — because it caused the reader    │             ║
║  │ to make assumptions that were ultimately     │             ║
║  │ INCORRECT."                                  │             ║
║  │                                              │             ║
║  │ → CONSISTENCY ≠ luôn luôn TỐT!             │             ║
║  │ → Khi consistency GÂY HIỂU LẦM,            │             ║
║  │   CLARITY thắng!                             │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  VÍ DỤ GO:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ❌ GÂY HIỂU LẦM:                            │             ║
║  │ tx, _ := db.Begin()                           │             ║
║  │ tx.Exec("UPDATE users SET name=$1", name)    │             ║
║  │ tx.Commit()                                   │             ║
║  │ // → Reader: "Sao chỉ có 1 query?"         │             ║
║  │ // → "Chắc thiếu code?"                     │             ║
║  │                                              │             ║
║  │ ✅ TRUNG THỰC:                                │             ║
║  │ db.Exec("UPDATE users SET name=$1", name)    │             ║
║  │ // → Reader: "À, chỉ 1 update, đơn giản!" │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Khi Bắt Buộc Phải "Dối"

### 6.1 Document the hell out of it!

```
╔═══════════════════════════════════════════════════════════════╗
║   KHI BẮT BUỘC PHẢI "DỐI" — DOCUMENT MẠNH!                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Nate Finch:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "If, for some reason, you HAVE TO make code  │             ║
║  │  that lies (to fulfill an interface or some  │             ║
║  │  such), DOCUMENT THE HELL OUT OF IT."        │             ║
║  │                                              │             ║
║  │ → GIANT YELLING COMMENTS!                    │             ║
║  │ → Không thể bỏ sót lúc 2am debugging!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  KHI NÀO BẮT BUỘC:                                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Fulfill interface requirement              │             ║
║  │    → Interface yêu cầu method tên X         │             ║
║  │    → Nhưng implementation khác ý nghĩa gốc │             ║
║  │                                              │             ║
║  │ 2. Legacy compatibility                       │             ║
║  │    → Phải giữ API cũ                         │             ║
║  │    → Behavior bên trong đã thay đổi         │             ║
║  │                                              │             ║
║  │ 3. Framework constraints                      │             ║
║  │    → Framework bắt implement method tên cố  │             ║
║  │      định                                     │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  CÁCH DOCUMENT:                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ❌ TỆ: Không comment gì                      │             ║
║  │                                              │             ║
║  │ ❌ TỆ: Comment nhỏ dễ bỏ qua               │             ║
║  │ // note: this doesn't actually save to memory│             ║
║  │                                              │             ║
║  │ ✅ TỐT: GIANT YELLING COMMENT!              │             ║
║  │ // ⚠️ WARNING: Despite the name,             │             ║
║  │ // SaveToMemory() actually persists           │             ║
║  │ // data to a PostgreSQL database!             │             ║
║  │ // This naming exists because it implements  │             ║
║  │ // the LegacyStore interface.                 │             ║
║  │ // DO NOT assume data is in-memory!          │             ║
║  │ // See ticket: TECH-1234                      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  → Nhưng TỐT NHẤT: ĐỔI tên cho đúng!                      ║
║  → Comment = LAST RESORT!                                     ║
║  → "If possible, don't resort to comments        ║
║     but instead, STRUCTURE THE CODE ITSELF."     ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §7. Áp Dụng Cho Go Codebase

### 7.1 Anti-patterns — Code nói dối trong Go

```go
// ═══ ANTI-PATTERNS: CODE NÓI DỐI TRONG GO ═══

// ❌ 1. IMPORT ALIAS LỪA DỐI
import (
    assert "github.com/stretchr/testify/require"
    // → assert.Equal() thực ra là require.Equal()!
    // → ĐỪNG BAO GIỜ làm thế!
)

// ❌ 2. GETTER CÓ SIDE EFFECTS
func (u *UserService) GetUser(id string) (*User, error) {
    user, err := u.db.FindByID(id)
    if err != nil {
        // "Get" nhưng lại TẠO MỚI user!
        user = &User{ID: id, Name: "default"}
        u.db.Create(user) // ← SIDE EFFECT!
    }
    return user, nil
}
// → Người đọc NGHĨ: Get = chỉ đọc, không thay đổi gì!
// → THỰC TẾ: Get = có thể TẠO user mới!
// → Tên đúng: GetOrCreateUser()

// ❌ 3. IsValid() CÓ MUTATION
func (o *Order) IsValid() bool {
    if o.Total < 0 {
        o.Total = 0    // ← MUTATION trong Is-check!
        o.Status = "invalid"
    }
    return o.Total > 0
}
// → "Is" prefix = PURE check, không thay đổi gì!
// → NHƯNG code lại THAY ĐỔI object!
// → Tên đúng: ValidateAndFix()

// ❌ 4. Close() KHÔNG CLOSE
func (c *Connection) Close() error {
    // Thực ra chỉ đánh dấu, KHÔNG close thật!
    c.marked = true
    return nil
    // → Connection vẫn CÒN MỞ!
    // → Resource leak!
}
// → Người dùng NGHĨ: resource đã được giải phóng!
// → THỰC TẾ: resource CHƯA được giải phóng!
```

### 7.2 Good patterns — Code trung thực trong Go

```go
// ═══ GOOD PATTERNS: CODE TRUNG THỰC TRONG GO ═══

// ✅ 1. TÊN PHẢN ÁNH ĐÚNG HÀNH VI
func (u *UserService) GetOrCreate(id string) (*User, error) {
    // Tên NÓI RÕ: có thể get HOẶC create!
    user, err := u.db.FindByID(id)
    if errors.Is(err, ErrNotFound) {
        return u.createDefault(id)
    }
    return user, err
}

// ✅ 2. INTERFACE ĐÚNG GO CONVENTION
type MessageWriter struct { /* ... */ }

// Nếu TƯƠNG THÍCH io.Writer → dùng Write()
func (w *MessageWriter) Write(p []byte) (int, error) {
    // Behavior ĐÚNG như io.Writer mong đợi!
    return w.conn.Write(p)
}

// Nếu KHÔNG tương thích → dùng tên KHÁC!
func (w *MessageWriter) SendMessage(msg []byte) error {
    // Tên KHÁC → không ai nhầm với io.Writer!
    return w.publish(msg)
}

// ✅ 3. ERROR TRUNG THỰC — Context đầy đủ
func processOrder(ctx context.Context, id string) error {
    order, err := fetchOrder(ctx, id)
    if err != nil {
        // Error NÓI THẬT chuyện gì xảy ra!
        return fmt.Errorf("process order %s: fetch: %w", id, err)
    }

    if err := validate(order); err != nil {
        return fmt.Errorf("process order %s: validate: %w", id, err)
    }
    // → Đọc error = BIẾT NGAY bước nào lỗi!
    return nil
}

// ✅ 4. KHI BẮT BUỘC "DỐI" → DOCUMENT RÕ RÀNG!
// ⚠️ WARNING: Ping does NOT actually send a network ping!
// It checks if the service is registered in the local cache.
// This method implements the HealthChecker interface which
// requires the name "Ping". For actual network health checks,
// use CheckConnectivity() instead.
// See: https://tickets.internal/ARCH-567
func (s *CacheService) Ping() error {
    if s.cache == nil {
        return errors.New("cache not initialized")
    }
    return nil
}
```

---

## §8. Tổng Kết & Câu Hỏi Phỏng Vấn Senior Golang

### 8.1 Câu hỏi phỏng vấn

```
╔═══════════════════════════════════════════════════════════════╗
║   CÂU HỎI PHỎNG VẤN SENIOR GOLANG                            ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Q1: "'Code must never lie' nghĩa là gì?"                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ = Code TRÔNG NHƯ làm gì thì phải LÀM ĐÚNG │             ║
║  │   điều đó!                                   │             ║
║  │                                              │             ║
║  │ → Tên hàm phải phản ánh đúng behavior      │             ║
║  │ → Import alias không được lừa người đọc    │             ║
║  │ → Conventions (Write, Read, Close...) phải  │             ║
║  │   đúng ý nghĩa trong Go ecosystem          │             ║
║  │ → "Cardinal sin" = tội chết nếu vi phạm!   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Lie vs Mislead — khác nhau thế nào?"                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ LIE: Code RÕ RÀNG nói điều sai              │             ║
║  │   → GetUser() tạo user, Close() không close │             ║
║  │                                              │             ║
║  │ MISLEAD: Code tạo GIẢ ĐỊNH SAI              │             ║
║  │   → Transaction cho 1 query (reader nghĩ    │             ║
║  │     phải có multi-table operations)           │             ║
║  │   → Mutex ở chỗ không cần concurrent        │             ║
║  │                                              │             ║
║  │ CẢ HAI đều cần TRÁNH!                       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Khi nào consistency gây hại?"                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Khi consistency tạo GIẢ ĐỊNH SAI!            │             ║
║  │                                              │             ║
║  │ Ví dụ: Luôn dùng transaction "cho consistent"│             ║
║  │ → Query đơn lẻ wrapped trong transaction    │             ║
║  │ → Reader nghĩ phải có multi-table logic     │             ║
║  │                                              │             ║
║  │ → CLARITY > CONSISTENCY khi 2 cái xung đột!│             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Làm sao đảm bảo code trung thực?"                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ 1. Tên hàm = ĐÚNG behavior (Get chỉ Get)   │             ║
║  │ 2. Tuân thủ Go conventions (io.Writer...)    │             ║
║  │ 3. Không import alias lừa đảo              │             ║
║  │ 4. Code review: hỏi "có gây hiểu lầm?"    │             ║
║  │ 5. Nếu BẮT BUỘC: document rõ ràng!         │             ║
║  │ 6. Structure code > comments                  │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 8.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: CODE MUST NEVER LIE                          │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. Code NEVER LIE = CARDINAL SIN!        │            │
    │  │    → Code phải làm ĐÚNG những gì nó     │            │
    │  │      TRÔNG NHƯ đang làm!                │            │
    │  │                                          │            │
    │  │ 2. Don't even MISLEAD!                    │            │
    │  │    → Con người TỰ ĐỘNG giả định         │            │
    │  │    → Dự đoán assumptions + đảm bảo      │            │
    │  │      chúng KHÔNG SAI!                    │            │
    │  │                                          │            │
    │  │ 3. Naming = SỰ THẬT                       │            │
    │  │    → Go conventions: Write, Read, Close  │            │
    │  │    → Dùng sai convention = nói dối!      │            │
    │  │                                          │            │
    │  │ 4. CLARITY > CONSISTENCY                   │            │
    │  │    → Consistency gây hiểu lầm = có hại │            │
    │  │                                          │            │
    │  │ 5. Bắt buộc phải "dối":                  │            │
    │  │    → DOCUMENT THE HELL OUT OF IT!         │            │
    │  │    → Giant yelling comments!              │            │
    │  │    → Nhưng TỐT NHẤT = đổi tên!          │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  Nate Finch:                                             │
    │  "Do the poor sap that has to maintain your             │
    │   code 6 months down the road a favor —                 │
    │   DON'T LIE. Because even if that poor sap             │
    │   isn't you, they still don't deserve the              │
    │   2am headache you'll likely be inflicting."            │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
