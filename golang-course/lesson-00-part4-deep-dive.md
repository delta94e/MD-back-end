# Part 4 Deep Dive — Methods, Pointers & Interfaces

> File này là phần mở rộng của `lesson-00-go-fundamentals.md`, đi sâu vào
> phần quan trọng nhất: **Pointer**, **Pointer Receivers**, và **Interfaces**.
> Nắm vững phần này = nắm được 80% tư duy của Go.

---

## 4A. Pointer — Từ Gốc Rễ

### Bộ Nhớ Trong Go: Stack vs Heap

```
╔══════════════════════════════════════════════════════════════════════╗
║  BỘ NHỚ CỦA MỘT GOROUTINE                                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Stack (nhanh, tự quản lý):                                         ║
║  ┌──────────────────────────┐                                        ║
║  │ main()                   │  ← frame của main                     ║
║  │   x int = 42     [0x100] │  ← x ở địa chỉ 0x100                 ║
║  │   s string = "hi" [0x108]│                                        ║
║  ├──────────────────────────┤                                        ║
║  │ foo()                    │  ← frame của foo (gọi từ main)        ║
║  │   y int = 10     [0x200] │                                        ║
║  └──────────────────────────┘                                        ║
║                                                                      ║
║  Heap (chậm hơn, GC quản lý):                                       ║
║  ┌──────────────────────────┐                                        ║
║  │ User{...}        [0x500] │  ← struct lớn, tồn tại lâu hơn       ║
║  │ []int{1,2,3}     [0x600] │  ← slice backing array                ║
║  └──────────────────────────┘                                        ║
║                                                                      ║
║  Pointer = GIÁ TRỊ chứa ĐỊA CHỈ bộ nhớ (0x100, 0x500...)          ║
╚══════════════════════════════════════════════════════════════════════╝
```

### & và * — Hai Operator Cốt Lõi

```go
x := 42           // x là int, nằm ở địa chỉ, ví dụ 0x100
                  // x lưu giá trị: 42

p := &x           // & = "address of" — lấy địa chỉ của x
                  // p là *int, lưu giá trị: 0x100 (địa chỉ của x)

fmt.Println(x)    // 42     — giá trị của x
fmt.Println(p)    // 0x100  — địa chỉ (dạng hex)
fmt.Println(*p)   // 42     — * = "dereference" (đọc giá trị tại địa chỉ đó)

*p = 100          // * bên trái = ghi vào địa chỉ đó
fmt.Println(x)    // 100 — x đã thay đổi!
```

```
Visualize bộ nhớ:

Trước khi gán *p = 100:
                x                       p
        ┌───────────────┐       ┌───────────────┐
 0x100  │      42       │       │     0x100     │  ← p trỏ đến x
        └───────────────┘       └───────────────┘
                ▲               p trỏ = p chứa địa chỉ của x
                └───────────────┘

Sau khi *p = 100 (ghi vào địa chỉ 0x100):
                x                       p
        ┌───────────────┐       ┌───────────────┐
 0x100  │     100       │       │     0x100     │  ← p vẫn trỏ đến x
        └───────────────┘       └───────────────┘
                                x đã đổi thành 100!
```

### Pass-by-Value vs Pass-by-Pointer

```go
// GO CHỈ CÓ PASS-BY-VALUE — nhưng pointer là value chứa địa chỉ!
// → Pass pointer = copy địa chỉ, vẫn truy cập được original

// ── PASS-BY-VALUE ───────────────────────────────────────────────────

func doubleWrong(n int) {
    n = n * 2  // n là bản sao — thay đổi không ảnh hưởng caller
}

x := 5
doubleWrong(x)
fmt.Println(x) // 5 — không đổi!

// Tại sao? Go copy giá trị 5 vào parameter n
// doubleWrong nhận copy, thao tác trên copy, caller không biết

// ── PASS-BY-POINTER ─────────────────────────────────────────────────

func doubleCorrect(n *int) {
    *n = *n * 2  // *n = đọc địa chỉ → lấy giá trị → nhân 2 → ghi lại
}

x := 5
doubleCorrect(&x)  // truyền địa chỉ của x
fmt.Println(x)    // 10 — đã thay đổi!

// Tại sao? Go copy địa chỉ (0x100) vào parameter n
// Cả caller và doubleCorrect đều trỏ đến cùng một vùng nhớ
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  SO SÁNH PASS-BY-VALUE vs PASS-BY-POINTER                           ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  PASS-BY-VALUE:                                                      ║
║  caller:  [x = 5 @ 0x100]                                           ║
║                │ copy                                                ║
║  function: [n = 5 @ 0x200]  ← bản sao, địa chỉ khác               ║
║            n = 10           ← chỉ thay 0x200                        ║
║  caller:  [x = 5 @ 0x100]  ← 0x100 vẫn = 5                        ║
║                                                                      ║
║  PASS-BY-POINTER:                                                    ║
║  caller:  [x = 5 @ 0x100]                                           ║
║                │ &x = 0x100 (copy địa chỉ)                          ║
║  function: [n = 0x100 @ 0x200]  ← n lưu địa chỉ của x             ║
║            *n = 10              ← ghi vào 0x100                     ║
║  caller:  [x = 10 @ 0x100] ← 0x100 đã = 10!                       ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Nil Pointer — Nguy Hiểm!

```go
// nil pointer = pointer không trỏ đến đâu cả
var p *int        // p = nil (zero value của pointer)
fmt.Println(p)   // <nil>

// Dereference nil pointer → PANIC!
// *p = 10       // runtime panic: invalid memory address or nil pointer dereference

// Luôn kiểm tra nil trước khi dereference
func safeDouble(p *int) {
    if p == nil {
        return // hoặc trả về error
    }
    *p = *p * 2
}

// Pointer struct — nil thường dùng để biểu thị "không có giá trị"
type User struct { Name string }

var user *User = nil  // chưa load từ DB
if user != nil {
    fmt.Println(user.Name) // safe
}
```

---

## 4B. Pointer Receivers — Hiểu Thật Sâu

### Value Receiver vs Pointer Receiver

```go
type BankAccount struct {
    Owner   string
    Balance float64
}

// ── VALUE RECEIVER (T) ─────────────────────────────────────────────
// Nhận BẢN SAO của struct
// Thay đổi trong method KHÔNG ảnh hưởng original
// Phù hợp: read-only operations

func (a BankAccount) GetBalance() float64 {
    return a.Balance
}

func (a BankAccount) String() string {
    return fmt.Sprintf("%s: $%.2f", a.Owner, a.Balance)
}

// BUG ĐIỂN HÌNH — dùng value receiver để modify:
func (a BankAccount) DepositWrong(amount float64) {
    a.Balance += amount  // chỉ thay bản sao! caller không thấy
}

// ── POINTER RECEIVER (*T) ──────────────────────────────────────────
// Nhận ĐỊA CHỈ của struct
// Thay đổi trong method ẢNH HƯỞNG original
// Phù hợp: mutation operations

func (a *BankAccount) Deposit(amount float64) {
    if amount <= 0 {
        return // validation
    }
    a.Balance += amount  // thay đổi struct gốc
}

func (a *BankAccount) Withdraw(amount float64) error {
    if amount > a.Balance {
        return fmt.Errorf("insufficient funds: have %.2f, want %.2f",
            a.Balance, amount)
    }
    a.Balance -= amount
    return nil
}
```

```go
// Chứng minh sự khác biệt:
acc := BankAccount{Owner: "Alice", Balance: 1000}

acc.DepositWrong(500)
fmt.Println(acc.Balance) // 1000 — KHÔNG THAY ĐỔI!

acc.Deposit(500)
fmt.Println(acc.Balance) // 1500 — ĐÃ THAY ĐỔI

err := acc.Withdraw(2000)
fmt.Println(err)         // "insufficient funds: have 1500.00, want 2000.00"
fmt.Println(acc.Balance) // 1500 — không thay đổi (withdraw thất bại)
```

### Go Tự Động Lấy Address — Nhưng Không Phải Lúc Nào Cũng Được

```go
// Khi gọi method trên biến — Go TỰ ĐỘNG convert:

acc := BankAccount{Owner: "Bob", Balance: 500}  // local variable

// Gọi pointer receiver trên value variable — GO TỰ LẤY ADDRESS
acc.Deposit(100)
// Go ngầm rewrite thành: (&acc).Deposit(100)
// Tại sao được? vì acc là addressable (có địa chỉ trong bộ nhớ)

// Gọi value receiver trên pointer variable — GO TỰ DEREFERENCE
pacc := &acc
pacc.GetBalance()
// Go ngầm rewrite thành: (*pacc).GetBalance()

// ❌ Trường hợp KHÔNG thể tự động lấy address:
BankAccount{Owner: "Carol", Balance: 200}.Deposit(50)
// COMPILE ERROR: cannot take the address of BankAccount literal
// Literal không có địa chỉ lâu dài (non-addressable)
```

### Method Set — Quy Tắc Quan Trọng Với Interface

```go
/*
╔══════════════════════════════════════════════════════════════════════╗
║  METHOD SET — Quy tắc AI CŨNG PHẢI NHỚ                             ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Type T (value):                                                     ║
║  → Method set = chỉ các method có VALUE receiver (T)                ║
║                                                                      ║
║  Type *T (pointer):                                                  ║
║  → Method set = VALUE receiver (T) + POINTER receiver (*T)          ║
║  → *T có ĐẦY ĐỦ method set hơn T                                   ║
║                                                                      ║
║  Hệ quả với Interface:                                              ║
║  → Nếu interface cần method pointer receiver → chỉ *T thỏa mãn     ║
║  → Nếu interface chỉ cần method value receiver → cả T và *T đều OK ║
╚══════════════════════════════════════════════════════════════════════╝
*/

type Account interface {
    GetBalance() float64  // value receiver
    Deposit(float64)      // pointer receiver trên BankAccount
}

acc := BankAccount{Owner: "Dave", Balance: 300}

// ❌ BankAccount (value) KHÔNG satisfy Account
// Vì Account cần Deposit, mà Deposit có pointer receiver
var a Account = acc   // COMPILE ERROR!

// ✅ *BankAccount (pointer) THỎA MÃN Account
var a Account = &acc  // OK! *BankAccount có cả GetBalance và Deposit

// Tại sao Go không cho phép tự động?
// Interface lưu BẢN SAO của value.
// Nếu Go cho phép BankAccount → Account, Deposit() sẽ modify bản sao
// trong interface, không phải acc gốc → silent bug!
```

### Struct Lớn — Dùng Pointer Để Tránh Copy

```go
// Struct nhỏ (< ~4-8 words) — value receiver OK
type Point struct { X, Y float64 }           // 16 bytes
type Color struct { R, G, B, A uint8 }       // 4 bytes

func (p Point) Distance() float64 { return math.Sqrt(p.X*p.X + p.Y*p.Y) }

// Struct lớn — pointer receiver bắt buộc (tránh copy tốn kém)
type UserProfile struct {
    ID          string           // 16 bytes (string header)
    Name        string           // 16 bytes
    Email       string           // 16 bytes
    Permissions []string         // 24 bytes (slice header)
    Metadata    map[string]any   // 8 bytes (map pointer)
    CreatedAt   time.Time        // 24 bytes
    // ≈ 104+ bytes mỗi lần copy!
}

// Value receiver: Go COPY toàn bộ 104 bytes mỗi lần gọi
// func (u UserProfile) Display() → TỐN!

// Pointer receiver: chỉ copy 8 bytes (pointer)
func (u *UserProfile) Display() string {
    return fmt.Sprintf("%s <%s>", u.Name, u.Email)
}

func (u *UserProfile) HasPermission(perm string) bool {
    for _, p := range u.Permissions {
        if p == perm {
            return true
        }
    }
    return false
}

// Struct chứa mutex — BẮT BUỘC pointer receiver, KHÔNG ĐƯỢC copy
type SafeCounter struct {
    mu    sync.Mutex  // mutex không được copy sau lần đầu dùng
    count int
}

// ❌ Value receiver: Go copy mutex → undefined behavior!
// func (c SafeCounter) Increment() { c.mu.Lock() ... }

// ✅ Pointer receiver — không copy mutex
func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}
```

---

## 4C. Interface — Cơ Chế Bên Trong

### Interface Value Trong Bộ Nhớ

```go
/*
Interface value = 2 words (trên 64-bit system: 16 bytes):

  ┌─────────────────┬─────────────────┐
  │   type pointer  │  data pointer   │
  │    (8 bytes)    │   (8 bytes)     │
  └─────────────────┴─────────────────┘
         │                  │
         ▼                  ▼
   type metadata       actual value
   (method table,      (hoặc pointer
    type info)          đến value)

nil interface:
  ┌─────────────────┬─────────────────┐
  │      nil        │      nil        │
  └─────────────────┴─────────────────┘
  → i == nil: TRUE

non-nil interface holding nil pointer:
  ┌─────────────────┬─────────────────┐
  │  *BankAccount   │      nil        │  ← type = *BankAccount
  └─────────────────┴─────────────────┘
  → i == nil: FALSE (có type info!)
*/
```

```go
// Minh họa bằng code:
var a Account // nil interface: (nil, nil)
fmt.Println(a == nil) // true

var acc *BankAccount = nil  // nil pointer (có type)
a = acc  // interface = (*BankAccount, nil) — KHÔNG phải nil interface!
fmt.Println(a == nil) // false!  ← GOTCHA!

// Kiểm tra: a.GetBalance() sẽ gọi method với nil receiver
// Nếu method handle nil receiver thì OK, nếu không → PANIC
func (a *BankAccount) GetBalance() float64 {
    if a == nil {
        return 0  // safe với nil receiver
    }
    return a.Balance
}
```

### Dynamic Dispatch — Tại Sao Interface Chậm Hơn Direct Call?

```go
/*
DIRECT METHOD CALL (compile time):
acc.GetBalance()
→ Compiler biết chính xác: call BankAccount.GetBalance
→ 1 instruction: CALL <trực tiếp địa chỉ hàm>

INTERFACE METHOD CALL (runtime):
var a Account = &acc
a.GetBalance()
→ Runtime: đọc type pointer từ interface
→ Tìm method GetBalance trong method table (itab)
→ 2 instructions: LOAD type + CALL qua pointer
→ Chậm hơn ~1-5ns (thường không đáng kể)

BỒI THƯỜNG: Compiler có thể devirtualize interface call
nếu nhận ra concrete type tại compile time (rare)
*/

// Khi nào interface overhead đáng lo?
// → Hot path: millions of calls per second (audio processing, game loops)
// → Hầu hết web API: KHÔNG cần lo
```

### Implement Interface Implicitly — Sức Mạnh

```go
// Trong Java/C#: phải khai báo tường minh
// class BankAccount implements Account { ... }  // Java

// Trong Go: KHÔNG cần khai báo — chỉ cần có đủ methods
// → Bên thứ ba có thể implement interface của bạn mà không cần import!

// stdlib định nghĩa:
// package fmt
// type Stringer interface { String() string }

// Bạn implement trong package của mình:
type Temperature struct { Celsius float64 }
func (t Temperature) String() string {
    return fmt.Sprintf("%.1f°C (%.1f°F)", t.Celsius, t.Celsius*9/5+32)
}

// fmt package KHÔNG biết Temperature, nhưng tự nhiên nó hoạt động!
t := Temperature{Celsius: 37.5}
fmt.Println(t) // 37.5°C (99.5°F)

// Kiểm tra compile-time interface compliance:
var _ fmt.Stringer = Temperature{}  // ✅ OK: value receiver
var _ fmt.Stringer = (*Temperature)(nil)  // ✅ OK: pointer thỏa mãn value interface
```

---

## 4D. Patterns Thực Tế

### Pattern 1: Constructor + Pointer

```go
// Quy ước trong Go: hàm New* trả về pointer
// Lý do: struct thường có pointer receivers → cần *T để dùng interface

type UserService struct {
    repo   UserRepository
    logger *zap.Logger
    cache  *redis.Client
}

// Constructor — LUÔN trả về *UserService
func NewUserService(repo UserRepository, logger *zap.Logger) *UserService {
    return &UserService{
        repo:   repo,
        logger: logger,
    }
}

// Tất cả methods dùng pointer receiver (nhất quán)
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    s.logger.Info("getting user", zap.String("id", id))
    return s.repo.FindByID(ctx, id)
}

func (s *UserService) CreateUser(ctx context.Context, req CreateUserRequest) (*User, error) {
    // ...
}

// Sử dụng:
svc := NewUserService(repo, logger)  // *UserService
user, err := svc.GetUser(ctx, "123")
```

### Pattern 2: Interface + Mock Cho Testing

```go
// Định nghĩa interface nhỏ (Interface Segregation Principle)
type UserFinder interface {
    FindByID(ctx context.Context, id string) (*User, error)
}

type UserCreator interface {
    Save(ctx context.Context, user *User) error
}

// Phần lớn use case chỉ cần 1-2 methods
type getUserUseCase struct {
    repo UserFinder  // chỉ cần Finder, không cần cả Creator
}

// Mock cho testing — chỉ implement những gì cần test
type mockUserFinder struct {
    // map id → user để control từng test case
    users map[string]*User
    err   error  // inject error để test error path
}

func (m *mockUserFinder) FindByID(_ context.Context, id string) (*User, error) {
    if m.err != nil {
        return nil, m.err
    }
    user, ok := m.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return user, nil
}

// Compile-time check
var _ UserFinder = (*mockUserFinder)(nil)

// Test:
func TestGetUserUseCase(t *testing.T) {
    tests := []struct{
        name    string
        id      string
        mock    *mockUserFinder
        wantErr error
    }{
        {
            name: "found",
            id: "123",
            mock: &mockUserFinder{
                users: map[string]*User{"123": {ID: "123", Name: "Alice"}},
            },
        },
        {
            name: "not found",
            id: "999",
            mock: &mockUserFinder{users: map[string]*User{}},
            wantErr: ErrNotFound,
        },
        {
            name: "db error",
            id: "any",
            mock: &mockUserFinder{err: errors.New("connection refused")},
            wantErr: errors.New("connection refused"),
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            uc := &getUserUseCase{repo: tt.mock}
            _, err := uc.GetUser(context.Background(), tt.id)
            if tt.wantErr != nil && err == nil {
                t.Error("expected error, got nil")
            }
        })
    }
}
```

### Pattern 3: Functional Options (Closure + Interface)

```go
// Vấn đề: struct config có nhiều optional fields
// Không muốn constructor với 10 tham số

type HTTPClient struct {
    timeout     time.Duration
    retries     int
    baseURL     string
    userAgent   string
    roundTripper http.RoundTripper
}

// Option là function type (không phải interface, nhưng dùng như interface)
type Option func(*HTTPClient)

func WithTimeout(d time.Duration) Option {
    return func(c *HTTPClient) { c.timeout = d }
}

func WithRetries(n int) Option {
    return func(c *HTTPClient) { c.retries = n }
}

func WithBaseURL(url string) Option {
    return func(c *HTTPClient) { c.baseURL = url }
}

func NewHTTPClient(opts ...Option) *HTTPClient {
    // Giá trị mặc định
    c := &HTTPClient{
        timeout:   30 * time.Second,
        retries:   3,
        userAgent: "go-client/1.0",
    }
    // Apply từng option
    for _, opt := range opts {
        opt(c)
    }
    return c
}

// Sử dụng — rõ ràng, extensible, không breaking change
client := NewHTTPClient(
    WithTimeout(10 * time.Second),
    WithRetries(5),
    WithBaseURL("https://api.example.com"),
)
```

---

## 4E. Tổng Kết — Decision Tree

```
╔══════════════════════════════════════════════════════════════════════╗
║  KHI NÀO DÙNG POINTER?                                              ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Câu hỏi 1: Function/Method cần modify dữ liệu không?              ║
║  → CÓ  → Dùng *T (pointer) hoặc *receiver                          ║
║  → KHÔNG → Sang câu 2                                               ║
║                                                                      ║
║  Câu hỏi 2: Type có chứa mutex/chan/sync primitive không?           ║
║  → CÓ  → BẮT BUỘC dùng *T (không được copy!)                      ║
║  → KHÔNG → Sang câu 3                                               ║
║                                                                      ║
║  Câu hỏi 3: Size của struct > ~64 bytes không?                      ║
║  → CÓ  → Nên dùng *T (tránh copy tốn kém)                         ║
║  → KHÔNG → Sang câu 4                                               ║
║                                                                      ║
║  Câu hỏi 4: Type có dùng với interface không?                       ║
║  → CÓ, interface cần pointer recv → Dùng *T                        ║
║  → KHÔNG hoặc chỉ value recv → Dùng T (value) OK                   ║
║                                                                      ║
║  LƯU Ý QUAN TRỌNG:                                                  ║
║  Nếu BẤT KỲ method nào của type cần *T                             ║
║  → Toàn bộ method set nên dùng *T (nhất quán)                      ║
║  → Trong thực tế: 90% structs đều dùng pointer receiver            ║
╚══════════════════════════════════════════════════════════════════════╝
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  KHI NÀO CẦN INTERFACE?                                             ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. Cần SWAP implementation (Postgres ↔ MySQL, Redis ↔ Memcached)  ║
║  2. Cần MOCK để test (repo, external API, time...)                  ║
║  3. CÓ NHIỀU types hành xử giống nhau (Shape, Handler, Formatter)  ║
║  4. Thư viện cần NHẬN input mà không phụ thuộc concrete type       ║
║                                                                      ║
║  KHÔNG cần interface khi:                                           ║
║  → Chỉ có 1 implementation và không định mock                      ║
║  → Tạo interface trước khi có 2+ implementations                   ║
║    (Go Proverb: "Don't design with interfaces, discover them")      ║
╚══════════════════════════════════════════════════════════════════════╝
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  CHECKLIST TRƯỚC KHI VIẾT STRUCT                                    ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  □ Constructor: func New*() *MyStruct                               ║
║  □ Receivers: nhất quán — toàn value hoặc toàn pointer             ║
║  □ Mutation methods: dùng pointer receiver                          ║
║  □ Struct có mutex? → KHÔNG export, KHÔNG copy                      ║
║  □ Expose interface, hide implementation                            ║
║  □ Compile-time check: var _ Interface = (*Impl)(nil)               ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Câu Hỏi Tự Kiểm Tra

1. Sự khác biệt giữa `*p` (dereference) và `&x` (address-of) là gì?
2. Khi nào Go tự động `(&v).Method()` và khi nào KHÔNG tự động được?
3. Tại sao `var a Account = myConcreteValue` có thể fail với interface?
4. `(*BankAccount)(nil).GetBalance()` có panic không? Khi nào thì không?
5. Tại sao không nên return typed nil pointer từ func trả về interface?
6. Method set của `T` vs `*T` — cái nào có nhiều methods hơn?
7. Tại sao struct chứa `sync.Mutex` KHÔNG được copy?
8. `var _ MyInterface = (*MyStruct)(nil)` làm gì và tại sao hữu ích?

---

*Khi đã hiểu rõ phần này → Go sang [Lesson 01](./lesson-01-project-setup-and-architecture.md) để áp dụng vào Clean Architecture thực tế.*
