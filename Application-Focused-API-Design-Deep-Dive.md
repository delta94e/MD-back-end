# Application Focused API Design — Thiết Kế API Hướng Ứng Dụng

> **Tài liệu học dành cho:** Người mới bắt đầu, chuẩn bị phỏng vấn Senior Golang Software Engineer
> **Nguồn gốc:** Bài viết "Application Focused API Design" của William Kennedy (2016)
> **Góc nhìn:** API Design, Concrete Types vs Interface, Go Implicit Interface, Mocking
> **Ngôn ngữ:** Hoàn toàn bằng Tiếng Việt

---

## Mục lục

| #   | Chủ đề                                 | Mô tả                                            |
| --- | -------------------------------------- | ------------------------------------------------ |
| §1  | Triết lý: Application First            | Package API phục vụ ỨNG DỤNG, không phải test    |
| §2  | Code Example: pubsub Package           | Concrete type, factory function, không interface |
| §3  | Vấn đề: Team cần Mock                  | Developer muốn interface để mock — nói KHÔNG     |
| §4  | Giải pháp: Consumer Declares Interface | App developer TỰ declare interface               |
| §5  | Mock Implementation                    | Tạo mock struct trong app code                   |
| §6  | Go Implicit Interface — Sức Mạnh       | Convention over Configuration                    |
| §7  | Tổng kết & Câu hỏi phỏng vấn Senior    | Ôn tập & thực hành                               |

---

## §1. Triết Lý: Application First

```
╔═══════════════════════════════════════════════════════════════╗
║   APPLICATION FOCUSED API DESIGN                              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Kennedy:                                                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ "Package developers should focus on how the  │             ║
║  │  application developer needs to USE the API  │             ║
║  │  in their APPLICATIONS, not their TESTS."    │             ║
║  │                                              │             ║
║  │ → API phục vụ ỨNG DỤNG!                     │             ║
║  │ → KHÔNG phải phục vụ test!                   │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  HAI CÁCH TIẾP CẬN:                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ❌ TEST-FOCUSED:                             │             ║
║  │  → "API phải testable!"                      │             ║
║  │  → Thêm interface "cho dễ mock!"            │             ║
║  │  → API phức tạp hơn!                        │             ║
║  │  → Đoán user cần gì để test!                │             ║
║  │                                              │             ║
║  │  ✅ APPLICATION-FOCUSED:                      │             ║
║  │  → "API phải ĐƠN GIẢN để DÙNG!"            │             ║
║  │  → Concrete types khi đủ!                   │             ║
║  │  → Không đoán user cần gì!                  │             ║
║  │  → User tự lo test của mình!                │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NGUYÊN TẮC:                                                  ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Package = HỖ TRỢ application!               │             ║
║  │  ┌─────────────────────────┐                 │             ║
║  │  │ Package API             │                 │             ║
║  │  │ → Intuitive (trực quan)│                 │             ║
║  │  │ → Simple (đơn giản)   │                 │             ║
║  │  │ → Minimized (tối thiểu)│                │             ║
║  │  └─────────────────────────┘                 │             ║
║  │                                              │             ║
║  │  Tests = artifact of development!             │             ║
║  │  → Không build cùng app!                     │             ║
║  │  → Không chạy khi app chạy!                 │             ║
║  │  → Là TRÁCH NHIỆM của app developer!        │             ║
║  │  → KHÔNG PHẢI trách nhiệm package author!   │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §2. Code Example: pubsub Package

### 2.1 Package pubsub — chỉ dùng Concrete Types

```go
// ═══ PUBSUB PACKAGE — CONCRETE TYPES ONLY ═══

// Package pubsub simulates a package that provides
// publication/subscription type services.
package pubsub

// PubSub provides access to a queue system.
type PubSub struct {
    /* impl */
}

// New creates a pubsub value for use.
func New(/* impl */) *PubSub {
    ps := PubSub{
        /* impl */
    }
    /* impl */
    return &ps
}

// Publish sends the data for the specified key.
func (ps *PubSub) Publish(key string, v interface{}) error {
    /* impl */
}

// Subscribe requests the data for the specified key.
func (ps *PubSub) Subscribe(key string) error {
    /* impl */
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   PHÂN TÍCH PUBSUB PACKAGE                                   ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  TẠI SAO KHÔNG CÓ INTERFACE?                                 ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Kiểm tra 3 Guidelines:                       │             ║
║  │                                              │             ║
║  │  1. User cần cung cấp implementation?        │             ║
║  │     → KHÔNG! Hệ thống queue nội bộ duy nhất!│             ║
║  │                                              │             ║
║  │  2. Có nhiều implementations?                 │             ║
║  │     → KHÔNG! Chỉ 1 queuing system!          │             ║
║  │                                              │             ║
║  │  3. Phần code sẽ thay đổi?                  │             ║
║  │     → KHÔNG! "Nothing can change since we    │             ║
║  │       will not be required to support another │             ║
║  │       queuing system."                        │             ║
║  │                                              │             ║
║  │  → 3/3 = KHÔNG → KHÔNG CẦN interface!       │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  API DESIGN:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  pubsub package:                              │             ║
║  │  ┌─────────────────────┐                     │             ║
║  │  │ PubSub (struct)     │ ← EXPORTED concrete!│             ║
║  │  │ ┌─────────────────┐ │                     │             ║
║  │  │ │ Publish(...)    │ │                     │             ║
║  │  │ │ Subscribe(...)  │ │                     │             ║
║  │  │ └─────────────────┘ │                     │             ║
║  │  └─────────────────────┘                     │             ║
║  │                                              │             ║
║  │  func New() *PubSub    ← Return CONCRETE!   │             ║
║  │                                              │             ║
║  │  → Đơn giản!                                │             ║
║  │  → Trực quan!                               │             ║
║  │  → Tối thiểu!                               │             ║
║  │  → KHÔNG interface pollution!                │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §3. Vấn Đề: Team Cần Mock

```
╔═══════════════════════════════════════════════════════════════╗
║   VẤN ĐỀ — "TÔI CẦN INTERFACE ĐỂ MOCK!"                  ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  TÌNH HUỐNG:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Team member: "Tôi đang viết test cho app   │             ║
║  │  nhưng KHÔNG CÓ access tới hệ thống queue  │             ║
║  │  nội bộ khi test chạy! Anh thêm interface   │             ║
║  │  vào pubsub package để tôi mock được không?"│             ║
║  │                                              │             ║
║  │  Kennedy: "KHÔNG!"                            │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TẠI SAO KHÔNG?                                               ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  1. pubsub package KHÔNG CẦN interface!      │             ║
║  │     → Chỉ có 1 implementation!               │             ║
║  │     → Interface sẽ là POLLUTION!             │             ║
║  │                                              │             ║
║  │  2. Testing là TRÁCH NHIỆM của app developer!│             ║
║  │     → Package author KHÔNG BIẾT app cần gì  │             ║
║  │       để test!                                │             ║
║  │     → Đoán = slippery slope!                 │             ║
║  │                                              │             ║
║  │  3. Go có IMPLICIT interface!                 │             ║
║  │     → App developer TỰ declare interface!    │             ║
║  │     → KHÔNG CẦN package author làm!         │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  GIẢI PHÁP:                                                    ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  Kennedy: "Nếu bạn cần interface để mock,   │             ║
║  │  NOTHING IS STOPPING YOU from declaring one  │             ║
║  │  FOR YOURSELF."                               │             ║
║  │                                              │             ║
║  │  → BẠN tự declare interface!                 │             ║
║  │  → TRONG app code của BẠN!                   │             ║
║  │  → KHÔNG PHẢI trong pubsub package!          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §4. Giải Pháp: Consumer Declares Interface

### 4.1 App developer tự declare

```go
// ═══ TRONG APP CODE (KHÔNG PHẢI pubsub package!) ═══

// publisher is an interface to allow this package to mock the
// pubsub package support.
type publisher interface {
    Publish(key string, v interface{}) error
    Subscribe(key string) error
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   CONSUMER DECLARES INTERFACE                                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  SƠ ĐỒ KIẾN TRÚC:                                            ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  pubsub package (PRODUCER):                   │             ║
║  │  ┌─────────────────────┐                     │             ║
║  │  │ PubSub (struct)     │ ← Concrete only!   │             ║
║  │  │ • Publish(...)      │    Không interface! │             ║
║  │  │ • Subscribe(...)    │                     │             ║
║  │  └─────────────────────┘                     │             ║
║  │            │                                 │             ║
║  │            │ IMPLICIT implements!             │             ║
║  │            │ (Go tự nhận biết!)              │             ║
║  │            ▼                                 │             ║
║  │  app code (CONSUMER):                         │             ║
║  │  ┌─────────────────────┐                     │             ║
║  │  │ publisher (iface)   │ ← App tự declare!  │             ║
║  │  │ • Publish(...)      │                     │             ║
║  │  │ • Subscribe(...)    │                     │             ║
║  │  └─────────────────────┘                     │             ║
║  │       │           │                          │             ║
║  │       │           │                          │             ║
║  │  ┌────┴────┐ ┌────┴────┐                    │             ║
║  │  │ PubSub  │ │  mock   │                    │             ║
║  │  │(thật!) │ │(test!)  │                    │             ║
║  │  └─────────┘ └─────────┘                    │             ║
║  │                                              │             ║
║  │  PRODUCTION: dùng PubSub thật               │             ║
║  │  TEST: dùng mock!                            │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  TẠI SAO HOẠT ĐỘNG?                                          ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Go Interface = IMPLICIT (ngầm định)!         │             ║
║  │                                              │             ║
║  │ → pubsub.PubSub có Publish() + Subscribe()  │             ║
║  │ → App declare publisher interface với 2 methods│            ║
║  │ → Go compiler: "PubSub có cả 2 methods!"   │             ║
║  │ → PubSub TỰ ĐỘNG implements publisher!       │             ║
║  │ → KHÔNG CẦN keyword "implements"!           │             ║
║  │ → Convention over Configuration!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §5. Mock Implementation

### 5.1 Tạo mock trong app code

```go
// ═══ MOCK — TRONG APP CODE ═══

// mock is a concrete type to help support mocking.
type mock struct{}

// Publish implements the publisher interface for the mock.
func (m *mock) Publish(key string, v interface{}) error {
    // Test implementation — không gọi hệ thống thật!
    fmt.Println("MOCK: publish", key)
    return nil
}

// Subscribe implements the publisher interface for the mock.
func (m *mock) Subscribe(key string) error {
    // Test implementation — không gọi hệ thống thật!
    fmt.Println("MOCK: subscribe", key)
    return nil
}
```

```
╔═══════════════════════════════════════════════════════════════╗
║   MOCK TRONG ACTION                                           ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  APP CODE — DÙNG INTERFACE ĐÃ TỰ DECLARE:                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │ // App function nhận interface!               │             ║
║  │ func processEvents(pub publisher) error {     │             ║
║  │     return pub.Publish("events", data)        │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ // PRODUCTION:                                │             ║
║  │ func main() {                                 │             ║
║  │     ps := pubsub.New(/* config */)            │             ║
║  │     processEvents(ps) // ← PubSub thật!     │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  │ // TEST:                                      │             ║
║  │ func TestProcessEvents(t *testing.T) {        │             ║
║  │     m := &mock{}                              │             ║
║  │     processEvents(m) // ← Mock! Không queue! │             ║
║  │ }                                             │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  KẾT QUẢ:                                                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ ✅ pubsub package = ĐƠN GIẢN, không iface! │             ║
║  │ ✅ App developer TỰ mock = LINH HOẠT!       │             ║
║  │ ✅ Package author không cần đoán test needs! │             ║
║  │ ✅ Go implicit interface = MAGIC!             │             ║
║  │ ✅ Decoupling ở ĐÚNG NƠI CẦN!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## §6. Go Implicit Interface — Sức Mạnh

```
╔═══════════════════════════════════════════════════════════════╗
║   GO IMPLICIT INTERFACE — CONVENTION OVER CONFIGURATION       ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  JAVA vs GO:                                                   ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  JAVA (Explicit — Configuration):            │             ║
║  │  ┌────────────────────────────────┐          │             ║
║  │  │ // PRODUCER phải declare iface │          │             ║
║  │  │ public interface Publisher {    │          │             ║
║  │  │     void publish(String k);    │          │             ║
║  │  │ }                              │          │             ║
║  │  │                                │          │             ║
║  │  │ // PRODUCER phải implements!   │          │             ║
║  │  │ class PubSub IMPLEMENTS Publisher {│       │             ║
║  │  │     void publish(String k) {}  │          │             ║
║  │  │ }                              │          │             ║
║  │  │                                │          │             ║
║  │  │ → Producer BẮT BUỘC tạo iface │          │             ║
║  │  │ → Consumer KHÔNG THỂ tự tạo! │          │             ║
║  │  │ → Dẫn tới interface pollution! │          │             ║
║  │  └────────────────────────────────┘          │             ║
║  │                                              │             ║
║  │  GO (Implicit — Convention):                 │             ║
║  │  ┌────────────────────────────────┐          │             ║
║  │  │ // PRODUCER: chỉ struct!      │          │             ║
║  │  │ type PubSub struct { ... }     │          │             ║
║  │  │ func (ps *PubSub) Publish(...) │          │             ║
║  │  │                                │          │             ║
║  │  │ // CONSUMER tự declare iface!  │          │             ║
║  │  │ type publisher interface {     │          │             ║
║  │  │     Publish(...)               │          │             ║
║  │  │ }                              │          │             ║
║  │  │ // PubSub TỰ ĐỘNG thỏa mãn! │          │             ║
║  │  │                                │          │             ║
║  │  │ → Producer KHÔNG CẦN iface!  │          │             ║
║  │  │ → Consumer TỰ quyết decouple!│          │             ║
║  │  │ → Sạch! Đơn giản! Linh hoạt!│          │             ║
║  │  └────────────────────────────────┘          │             ║
║  │                                              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  NGUYÊN TẮC PHÂN CHIA TRÁCH NHIỆM:                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │                                              │             ║
║  │  ┌──────────────┐    ┌───────────────┐       │             ║
║  │  │ PACKAGE       │    │ APP           │       │             ║
║  │  │ AUTHOR        │    │ DEVELOPER     │       │             ║
║  │  ├──────────────┤    ├───────────────┤       │             ║
║  │  │ Concrete type│    │ Interface     │       │             ║
║  │  │ Factory func │    │ Mock struct   │       │             ║
║  │  │ Methods      │    │ Tests         │       │             ║
║  │  │ Docs         │    │ App code      │       │             ║
║  │  └──────────────┘    └───────────────┘       │             ║
║  │                                              │             ║
║  │  → Mỗi bên lo PHẦN CỦA MÌNH!              │             ║
║  │  → Package: simple API!                      │             ║
║  │  → App: interface khi CẦN mock!             │             ║
║  │                                              │             ║
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
║  Q1: "Package API nên thiết kế cho ai?"                      ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → Cho APPLICATION DEVELOPER!                  │             ║
║  │ → KHÔNG phải cho test của họ!                │             ║
║  │ → API phải simple, intuitive, minimized!     │             ║
║  │ → Test = trách nhiệm app developer!          │             ║
║  │ → Package author KHÔNG đoán test needs!      │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q2: "Accept interfaces, return structs' áp dụng            ║
║       thế nào với pubsub example?"                           ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ pubsub package:                               │             ║
║  │ → func New() *PubSub ← return STRUCT!        │             ║
║  │ → Không declare interface!                    │             ║
║  │                                              │             ║
║  │ App code:                                     │             ║
║  │ → func processEvents(pub publisher) ← IFACE! │             ║
║  │ → App tự declare publisher interface!        │             ║
║  │ → App tự tạo mock cho testing!              │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q3: "Khi team member yêu cầu thêm interface vào           ║
║       package, bạn sẽ trả lời thế nào?"                     ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ → KHÔNG thêm nếu không đáp ứng 3 Guidelines!│             ║
║  │ → Go implicit interface = consumer TỰ tạo!  │             ║
║  │ → "Nothing is stopping you from declaring    │             ║
║  │    one for yourself."                         │             ║
║  │ → Giúp họ declare interface TRONG app code!  │             ║
║  │ → Tạo mock struct TRONG app code!            │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
║  Q4: "Go implicit interface khác gì Java explicit?          ║
║       Ảnh hưởng API design thế nào?"                        ║
║  ┌──────────────────────────────────────────────┐             ║
║  │ Java: Producer PHẢI declare + implements!     │             ║
║  │ → Interface pollution tự nhiên!               │             ║
║  │                                              │             ║
║  │ Go: Consumer TỰ declare khi cần!             │             ║
║  │ → Convention over Configuration!              │             ║
║  │ → Package API simple → consumer linh hoạt!  │             ║
║  │ → Decoupling ở ĐÚNG NƠI CẦN!               │             ║
║  │ → Test concerns = app developer's concern!    │             ║
║  └──────────────────────────────────────────────┘             ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 7.2 Tổng kết

```
    ┌──────────────────────────────────────────────────────────┐
    │  TỔNG KẾT: APPLICATION FOCUSED API DESIGN               │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    │  ┌──────────────────────────────────────────┐            │
    │  │ 1. API phục vụ APPLICATION, không test!   │            │
    │  │    → Simple, intuitive, minimized!        │            │
    │  │                                          │            │
    │  │ 2. Không cần interface? ĐỪNG TẠO!        │            │
    │  │    → 3 Guidelines quyết định!             │            │
    │  │    → Concrete type khi đủ!               │            │
    │  │                                          │            │
    │  │ 3. Consumer TỰ declare interface!         │            │
    │  │    → Go implicit interface = sức mạnh!    │            │
    │  │    → Không cần package author!             │            │
    │  │                                          │            │
    │  │ 4. Mock = trách nhiệm app developer!      │            │
    │  │    → Tự declare interface trong app!       │            │
    │  │    → Tự tạo mock struct!                 │            │
    │  │                                          │            │
    │  │ 5. Phân chia trách nhiệm rõ ràng!        │            │
    │  │    → Package: concrete API!               │            │
    │  │    → App: interface + mock + test!         │            │
    │  └──────────────────────────────────────────┘            │
    │                                                          │
    │  William Kennedy:                                        │
    │  "I believe this will keep package APIs SIMPLER,        │
    │   MINIMIZED and more INTUITIVE for application          │
    │   developers."                                          │
    │                                                          │
    │  Nate Finch:                                             │
    │  "I think it's ok to do heinous stuff to test           │
    │   an API if it makes it more usable by others."         │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```
