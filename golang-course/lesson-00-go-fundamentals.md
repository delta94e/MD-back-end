# Lesson 00 — Go Fundamentals: Nền Tảng Bắt Buộc

> **Mục tiêu:** Nắm vững cú pháp và tư duy của Go trước khi bước vào kiến trúc production.
> Nội dung dựa trên **A Tour of Go** — tài liệu chính thức của Go team, được làm sâu thêm với
> giải thích tại sao Go thiết kế theo hướng đó.
>
> **Yêu cầu:** Cài Go ≥ 1.21. Chạy thử tại [go.dev/tour](https://go.dev/tour) hoặc [play.golang.org](https://play.golang.org)

---

## Bản Đồ Tổng Quan

```
╔══════════════════════════════════════════════════════════════════════╗
║           LESSON 00 — GO FUNDAMENTALS                               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Part 1: Packages, Variables & Functions                            ║
║  Part 2: Flow Control (for, if, switch, defer)                      ║
║  Part 3: More Types (Pointers, Structs, Slices, Maps, Closures)     ║
║  Part 4: Methods & Interfaces (★ co ban cho Clean Architecture)    ║
║  Part 5: Generics — Type Parameters & Constraints (Go 1.18+)       ║
║  Part 6: Concurrency — Goroutines, Channels, Mutex                 ║
║                                                                      ║
║  Deep Dive Files:                                                   ║
║  → lesson-00-part4-deep-dive.md  (Pointer / Interface internals)   ║
║  → lesson-00-part5-generics.md   (Generics day du)                 ║
║  → lesson-00-part6-concurrency.md (Concurrency day du)             ║
║                                                                      ║
║  → Sau lesson nay: Lesson 01 (Clean Architecture & Production API)  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Part 1: Packages, Variables & Functions

### 1.1 — Hello, 世界

```go
// Mọi Go program bắt đầu bằng package declaration
// package main = executable program (khác với library)
package main

import "fmt" // fmt = format, stdlib package cho I/O

func main() {
    fmt.Println("Hello, 世界") // Go native support Unicode
}
```

**Tại sao Go in được "世界"?** Go source files luôn là UTF-8. Đây là design decision từ đầu — creator của Go (Rob Pike) cũng là co-creator của UTF-8.

### 1.2 — Packages & Imports

```go
package main

import (
    "fmt"
    "math"
    "math/rand" // sub-package: chỉ dùng rand.Xxx(), không phải math/rand.Xxx()
)

func main() {
    fmt.Printf("Pi = %.2f\n", math.Pi)      // math.Pi = 3.14
    fmt.Println("Random:", rand.Intn(100))  // 0-99
}
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  EXPORTED NAMES — Quy tắc Capital Letter                           ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  math.Pi   → exported (Public)  — chữ HOA đầu tiên                ║
║  math.pi   → unexported (private) — chữ thường đầu tiên           ║
║                                                                      ║
║  Rule: Chữ hoa = public API. Chữ thường = internal only.           ║
║  Đây là cơ chế encapsulation DUY NHẤT trong Go (không có private,  ║
║  protected keyword như Java/C++)                                     ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 1.3 — Functions

```go
// Go: type đứng SAU tên biến (khác C/Java)
// Lý do: dễ đọc hơn khi type phức tạp (chan []func(int) string)
func add(x int, y int) int {
    return x + y
}

// Shorthand: cùng type → viết 1 lần
func add(x, y int) int {
    return x + y
}

// MULTIPLE RETURN VALUES — đặc trưng của Go
// Không cần boxing vào struct/tuple như Python hay Java
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

// Gọi:
result, err := divide(10, 3)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("%.2f\n", result) // 3.33
```

**Tại sao multiple return values quan trọng?** Go không có exceptions. Mọi lỗi đều được trả về tường minh. Đây là lý do code Go rất explicit và dễ trace error path.

```go
// NAMED RETURN VALUES — có thể đặt tên cho return values
// Hữu ích khi function phức tạp, cần document rõ ý nghĩa từng giá trị
func minMax(arr []int) (min, max int) {
    min, max = arr[0], arr[0]
    for _, v := range arr {
        if v < min { min = v }
        if v > max { max = v }
    }
    return // "naked return" — trả về min, max đã được set
}
```

> ⚠️ **Naked return** nên tránh trong function dài — khó đọc. Chỉ dùng cho function ngắn (< 5 dòng).

### 1.4 — Variables

```go
// var declaration — package level hoặc function level
var x int        // zero value = 0
var s string     // zero value = ""
var b bool       // zero value = false

// Multiple variables
var (
    age  int    = 25
    name string = "Go"
)

// SHORT VARIABLE DECLARATION (:=) — chỉ dùng TRONG function
// Go tự infer type từ value bên phải
count := 0          // int
message := "hello"  // string
pi := 3.14          // float64
isValid := true     // bool

// := vs var:
// := chỉ trong function body
// var ở package level hoặc khi cần khai báo mà chưa có value
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  ZERO VALUES — Go không có "undefined" hay "null pointer" mặc định  ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  int, int8, int64...    → 0                                         ║
║  float32, float64       → 0.0                                       ║
║  string                 → "" (empty string)                         ║
║  bool                   → false                                     ║
║  pointer, slice, map    → nil                                       ║
║  struct                 → mỗi field zero value của type tương ứng   ║
║                                                                      ║
║  Lợi ích: không bao giờ đọc garbage memory như C/C++               ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 1.5 — Basic Types & Type Conversions

```go
// Go có type system nghiêm ngặt — KHÔNG có implicit conversion
var i int = 42
var f float64 = float64(i) // phải explicit cast
var u uint = uint(f)

// Sai — compile error:
// var f float64 = i // cannot use i (type int) as type float64

// Basic types:
bool
string
int  int8  int16  int32  int64    // signed integers
uint uint8 uint16 uint32 uint64   // unsigned integers
byte    // alias cho uint8 (thường dùng cho raw bytes)
rune    // alias cho int32 (Unicode code point)
float32 float64
complex64 complex128
```

```go
// TYPE INFERENCE — Go tự suy ra type
i := 42           // int (platform-dependent: 32 hoặc 64 bit)
f := 3.14         // float64 (mặc định cho decimal literals)
c := 0.867 + 0.5i // complex128

// Kiểm tra type tại runtime:
fmt.Printf("Type: %T, Value: %v\n", i, i) // Type: int, Value: 42
```

### 1.6 — Constants

```go
const Pi = 3.14159 // untyped constant — có thể dùng với bất kỳ numeric type

// Typed constant
const MaxRetries int = 3

// iota — auto-increment trong const block
type Weekday int
const (
    Sunday Weekday = iota // 0
    Monday                // 1
    Tuesday               // 2
    Wednesday             // 3
    Thursday              // 4
    Friday                // 5
    Saturday              // 6
)

// iota patterns phức tạp hơn:
type ByteSize float64
const (
    _           = iota // bỏ qua 0
    KB ByteSize = 1 << (10 * iota) // 1024
    MB                              // 1048576
    GB                              // 1073741824
)
```

---

## Part 2: Flow Control

### 2.1 — For Loop

```go
// Go CHỈ CÓ 1 loại loop: for
// (không có while, do-while)

// C-style for
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// "while" trong Go:
n := 1
for n < 100 {
    n *= 2
}

// Infinite loop:
for {
    // chạy mãi, cần break hoặc return để thoát
}

// Loop với index và value (range):
nums := []int{10, 20, 30}
for i, v := range nums {
    fmt.Printf("index=%d value=%d\n", i, v)
}

// Chỉ muốn value (bỏ index với _):
for _, v := range nums {
    fmt.Println(v)
}

// Chỉ muốn index:
for i := range nums {
    fmt.Println(i)
}
```

### 2.2 — If / Else

```go
// If với short statement — scope của x chỉ trong if/else block
// Pattern này RẤT phổ biến trong Go
if x := computeValue(); x > 0 {
    fmt.Println("positive:", x)
} else {
    fmt.Println("non-positive:", x)
}
// x không tồn tại ở đây

// Pattern cực kỳ phổ biến: if err != nil
user, err := getUserFromDB(id)
if err != nil {
    return nil, fmt.Errorf("getUser: %w", err)
}
// tiếp tục dùng user...
```

### 2.3 — Switch

```go
// Go switch KHÔNG fall-through mặc định (khác C/Java)
// Không cần break ở cuối mỗi case
os := runtime.GOOS
switch os {
case "darwin":
    fmt.Println("macOS")
case "linux":
    fmt.Println("Linux")
default:
    fmt.Println("Other:", os)
}

// Switch không điều kiện = if-else chain
t := time.Now()
switch {
case t.Hour() < 12:
    fmt.Println("Morning")
case t.Hour() < 17:
    fmt.Println("Afternoon")
default:
    fmt.Println("Evening")
}

// Type switch — xác định type của interface{}
func describe(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case string:
        return fmt.Sprintf("string: %q", v)
    default:
        return fmt.Sprintf("unknown type: %T", v)
    }
}
```

### 2.4 — Defer

```go
// defer: trì hoãn execution cho đến khi function return
// RẤT phổ biến cho cleanup (close file, unlock mutex, close DB connection)

func readFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close() // đảm bảo file LUÔN được đóng dù return ở đâu

    // ... đọc file
    return nil
}

// STACKING DEFERS — LIFO (Last In, First Out)
func main() {
    defer fmt.Println("world") // chạy thứ 3
    defer fmt.Println("!")     // chạy thứ 2
    fmt.Println("hello")       // chạy thứ 1
    // Output: hello, !, world
}

// Lưu ý: defer capture VALUE tại thời điểm gọi, không phải reference
// Ngoại lệ: function literal capture biến theo reference
i := 0
defer fmt.Println(i) // in 0, không phải 1
i = 1
```

---

## Part 3: More Types

### 3.1 — Pointers

```go
// Go có pointers nhưng KHÔNG có pointer arithmetic (khác C)
// & = address-of operator (lấy địa chỉ)
// * = dereference operator (lấy giá trị tại địa chỉ)

i := 42
p := &i         // p là *int, trỏ đến i
fmt.Println(*p) // 42 — dereference
*p = 100        // thay đổi i thông qua pointer
fmt.Println(i)  // 100

// Tại sao cần pointer trong Go?
// 1. Pass large struct mà không copy toàn bộ
// 2. Cho phép function modify caller's variable
// 3. Optional value (nil pointer = không có giá trị)

func increment(n *int) { // nhận pointer, có thể modify caller's variable
    *n++
}

x := 10
increment(&x) // truyền địa chỉ của x
fmt.Println(x) // 11
```

### 3.2 — Structs

```go
// Struct = collection of fields (như class nhưng không có inheritance)
type Point struct {
    X int
    Y int
}

// Khởi tạo
p1 := Point{1, 2}          // positional (không khuyến khích)
p2 := Point{X: 1, Y: 2}    // named fields (KHUYẾN KHÍCH — rõ ràng hơn)
p3 := Point{}               // zero value: {X:0, Y:0}

// Truy cập field
fmt.Println(p2.X) // 1

// Pointer to struct — Go cho phép dot notation trực tiếp (không cần ->)
pp := &p2
pp.X = 10   // tương đương (*pp).X = 10, Go tự dereference
fmt.Println(p2.X) // 10

// Struct embedding — Go's alternative to inheritance
type Animal struct {
    Name string
}
func (a Animal) Speak() string { return a.Name + " speaks" }

type Dog struct {
    Animal        // Embed Animal — Dog có tất cả fields và methods của Animal
    Breed string
}

d := Dog{Animal: Animal{Name: "Buddy"}, Breed: "Labrador"}
fmt.Println(d.Name)   // Buddy — promoted field
fmt.Println(d.Speak()) // Buddy speaks — promoted method
fmt.Println(d.Breed)  // Labrador
```

### 3.3 — Slices (Quan Trọng!)

```go
// Array trong Go có fixed size — ít khi dùng trực tiếp
var a [2]string
a[0] = "Hello"
a[1] = "World"

// SLICE — dynamic, flexible view vào array — đây mới là cái dùng hàng ngày
s := []int{1, 2, 3, 4, 5}

// Slice là reference — khi slice sang nhau, cùng share underlying array!
a := []int{1, 2, 3}
b := a[1:3]  // b = [2, 3], nhưng share memory với a
b[0] = 99
fmt.Println(a) // [1, 99, 3] — a cũng bị thay đổi!
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  SLICE INTERNALS — 3 fields                                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  slice = { pointer, length, capacity }                              ║
║                                                                      ║
║  Underlying Array:  [1] [2] [3] [4] [5]                            ║
║                      ↑                                               ║
║  slice s:          ptr=&arr[0], len=5, cap=5                        ║
║                                                                      ║
║  s[1:3]:           ptr=&arr[1], len=2, cap=4                        ║
║                          ↑                                           ║
║  → len = số phần tử hiện có                                         ║
║  → cap = số phần tử có thể có (không cần alloc mới)                ║
╚══════════════════════════════════════════════════════════════════════╝
```

```go
// make — tạo slice với specific len và cap
s := make([]int, 5)       // len=5, cap=5, tất cả = 0
s := make([]int, 3, 10)   // len=3, cap=10

// append — thêm phần tử
s := []int{1, 2, 3}
s = append(s, 4, 5)       // s = [1, 2, 3, 4, 5]
s = append(s, []int{6, 7}...) // spread operator

// Khi append vượt cap → Go tự allocate array mới (thường gấp đôi cap)
// Đây là lý do nên init với cap đủ nếu biết trước size

// NIL SLICE vs EMPTY SLICE
var nilSlice []int     // nil, len=0, cap=0
emptySlice := []int{} // không nil, len=0, cap=0

// Cả hai đều usable với append và range
// Khác nhau: nilSlice == nil là true, emptySlice == nil là false
```

### 3.4 — Maps

```go
// Map = hash map / dictionary
// Syntax: map[KeyType]ValueType

// Khởi tạo phải dùng make hoặc literal — nil map panic khi write!
m := make(map[string]int)
m["one"] = 1
m["two"] = 2

// Map literal
scores := map[string]int{
    "Alice": 95,
    "Bob":   87,
    "Carol": 92,
}

// Đọc — luôn trả về zero value nếu key không tồn tại
score := scores["David"] // 0, không panic

// Kiểm tra key có tồn tại không — "comma ok" idiom
score, ok := scores["Alice"]
if ok {
    fmt.Println("Alice:", score)
} else {
    fmt.Println("Alice not found")
}

// Xóa key
delete(scores, "Bob")

// Iterate — thứ tự KHÔNG đảm bảo!
for name, score := range scores {
    fmt.Printf("%s: %d\n", name, score)
}

// Nếu cần thứ tự: sort keys trước
keys := make([]string, 0, len(scores))
for k := range scores {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Printf("%s: %d\n", k, scores[k])
}
```

### 3.5 — Function Values & Closures

```go
// Functions là first-class citizens trong Go
// Có thể assign vào variable, pass làm argument, return từ function

// Function value
add := func(a, b int) int { return a + b }
fmt.Println(add(3, 4)) // 7

// Higher-order function
func apply(fn func(int, int) int, a, b int) int {
    return fn(a, b)
}
fmt.Println(apply(add, 3, 4)) // 7

// CLOSURE — function capture variables từ enclosing scope
func makeCounter() func() int {
    count := 0
    return func() int { // closure capture count
        count++
        return count
    }
}

counter := makeCounter()
fmt.Println(counter()) // 1
fmt.Println(counter()) // 2
fmt.Println(counter()) // 3

counter2 := makeCounter() // counter2 có state riêng
fmt.Println(counter2())   // 1
```

**Ứng dụng Closure trong production Go:**

```go
// 1. Middleware pattern (rất phổ biến trong web framework)
func withLogging(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Printf("START %s %s", r.Method, r.URL.Path)
        next(w, r)
        log.Printf("END %s %s", r.Method, r.URL.Path)
    }
}

// 2. Functional options pattern
type Server struct { timeout time.Duration }
type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

// 3. Goroutine capture (common gotcha!)
// SAI — tất cả goroutines share biến i cuối cùng
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i) // có thể in 3,3,3
    }()
}

// ĐÚNG — capture giá trị i tại thời điểm tạo goroutine
for i := 0; i < 3; i++ {
    i := i // shadow variable
    go func() {
        fmt.Println(i) // 0, 1, 2 (thứ tự không đảm bảo)
    }()
}
```

---

## Part 4: Methods & Interfaces

> ★ **Quan trọng nhất** — Đây là nền tảng của Clean Architecture trong Go.
> Repository interfaces, UseCase interfaces, error handling — tất cả đều dựa từ đây.

### 4.1 — Methods

```go
// Go không có class. Thay vào đó: gắn function vào type bằng RECEIVER
// Syntax: func (receiverName ReceiverType) MethodName(params) returnType
// receiverName thường là chữ viết tắt của type (1-2 chữ — không dùng 'self', 'this')

type Rectangle struct {
    Width, Height float64
}

// VALUE RECEIVER — nhận BẢN SAO của struct (như pass-by-value)
// r là copy của Rectangle — thay đổi r không ảnh hưởng original
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// Scale thay đổi r nhưng chỉ bản sao — caller không thấy thay đổi!
func (r Rectangle) ScaleWrong(factor float64) {
    r.Width *= factor  // chỉ thay bản sao
    r.Height *= factor // chỉ thay bản sao
}

rect := Rectangle{Width: 3, Height: 4}
rect.ScaleWrong(2)
fmt.Println(rect.Width) // 3 — không đổi!
```

```go
// METHOD LÀ FUNCTION CÓ RECEIVER — hoàn toàn tương đương, chỉ khác cú pháp
func AreaFn(r Rectangle) float64    { return r.Width * r.Height } // function
func (r Rectangle) Area() float64  { return r.Width * r.Height } // method

// CẦ hai có thể gọi như METHOD EXPRESSION:
rect := Rectangle{Width: 3, Height: 4}

// Gọi bình thường
fmt.Println(rect.Area()) // 12

// Method expression — lấy method ra thành function
areaFn := Rectangle.Area          // areaFn có type func(Rectangle) float64
fmt.Println(areaFn(rect))         // 12 — pass receiver explicitly

// Method value — bound vào instance cụ thể
boundArea := rect.Area            // bind vào rect
fmt.Println(boundArea())          // 12 — không cần pass receiver
```

```go
// METHOD TRÊN NON-STRUCT TYPES — Go cho phép define method trên bất kỳ type nào
// trong cùng package (không phải chỉ struct)

// Type alias cho primitive
type Celsius float64
type Fahrenheit float64

func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (f Fahrenheit) ToCelsius() Celsius {
    return Celsius((f - 32) * 5 / 9)
}

fmt.Println(Celsius(100).ToFahrenheit())   // 212
fmt.Println(Fahrenheit(32).ToCelsius())    // 0

// Type alias cho slice — thêm behavior vào slice
type StringSlice []string

func (ss StringSlice) Join(sep string) string {
    return strings.Join(ss, sep)
}

func (ss StringSlice) Filter(fn func(string) bool) StringSlice {
    result := make(StringSlice, 0)
    for _, s := range ss {
        if fn(s) {
            result = append(result, s)
        }
    }
    return result
}

ss := StringSlice{"go", "python", "rust", "java"}
filtered := ss.Filter(func(s string) bool { return len(s) <= 4 })
fmt.Println(filtered.Join(", ")) // "go, rust, java"
```

> **Tại sao Go không có class?**
> Class truyền thống gói data + behavior + inheritance + visibility vào 1 khái niệm. Go tách rời: data (struct), behavior (methods), reuse (embedding), visibility (exported names). Kết quả: đơn giản hơn và dễ compose hơn.

### 4.2 — Pointer Receivers (★ Critical)

```go
// POINTER RECEIVER — nhận ĐỊA CHỈ của struct (pass-by-reference)
// *Counter = pointer to Counter

type Counter struct {
    count int
    name  string
}

// Value receiver — modify bản sao, KHÔNG ảnh hưởng original
func (c Counter) BadIncrement() {
    c.count++ // chỉ modify bản sao!
    // Như func BadIncrement(c Counter) — Go copy toàn bộ struct
}

// Pointer receiver — modify original
func (c *Counter) Increment() {
    c.count++ // equivalent: (*c).count++
}

func (c *Counter) Reset()         { c.count = 0 }
func (c *Counter) SetName(n string) { c.name = n }
func (c Counter) Value() int      { return c.count }  // read-only OK
func (c Counter) String() string  { return fmt.Sprintf("%s: %d", c.name, c.count) }

c := Counter{}
c.BadIncrement()
fmt.Println(c.Value()) // 0 — không đổi!

c.Increment()
c.Increment()
fmt.Println(c.Value()) // 2
```

**Pointer Indirection — Go tự động convert:**

```go
// Go tự lưu ý việc lấy address khi cần
c := Counter{}   // local variable on stack

// Gọi pointer receiver trên value var — Go tự lấy (&c)
c.Increment()             // Go rewrite thành: (&c).Increment()

// Gọi value receiver trên pointer var — Go tự dereference
pc := &Counter{}          // pointer
pc.Value()                // Go rewrite thành: (*pc).Value()

// Nhưng với interface, không có tự động! (xem §4.3)
// Nếu người ta chỉ có Counter (value), không thể gán vào interface cần *Counter
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  POINTER INDIRECTION TABLE                                          ║
╠═══════════════════╦════════════════╦════════════════════════════════╣
║  Biến            ║  Method        ║  Gọi OK?                     ║
╠═══════════════════╬════════════════╬════════════════════════════════╣
║  v Counter        ║  Value recv    ║  ✅ trực tiếp                 ║
║  v Counter        ║  Pointer recv  ║  ✅ Go tự (&v).Method()       ║
║  p *Counter       ║  Value recv    ║  ✅ Go tự (*p).Method()       ║
║  p *Counter       ║  Pointer recv  ║  ✅ trực tiếp                 ║
║─────────────────────────────────────────────────────────────────────║
║  interface (value Counter)   Pointer recv  ❌ KHÔNG satisfy       ║
║  interface (value *Counter)  Pointer recv  ✅ satisfy              ║
║  interface (value *Counter)  Value recv    ✅ satisfy              ║
╚══════════════════════════════════════════════════════════════════════╝
```

```go
// VÍ DỤ THỰC TẾ: tại sao interface không tự động convert?
type Incrementer interface {
    Increment()
    Value() int
}

// *Counter implement Incrementer (vì Increment dùng pointer receiver)
var inc Incrementer = &Counter{} // ✅ OK

// Counter (value) KHÔNG implement Incrementer
var inc2 Incrementer = Counter{} // ❌ COMPILE ERROR!
// "Counter does not implement Incrementer
//  (Increment method has pointer receiver)"

// Tại sao? Vì interface store bản sao của value.
// Nếu Go cho phép, Increment() sẽ modify bản sao, không phải original —
// → silent bug nguy hiểm hơn lỗi compile!
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  RULE: Chọn Value hay Pointer Receiver?                            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Dùng POINTER receiver (*T) khi:                                    ║
║  → Method cần modify struct (Increment, Reset, Set...)             ║
║  → Struct lớn (> ~64 bytes) — tránh copy tốn kém               ║
║  → Struct chứa sync.Mutex, chan (không được copy)               ║
║                                                                      ║
║  Dùng VALUE receiver (T) khi:                                       ║
║  → Type là nhỏ, read-only (Point, Color, Celsius, bool-like)      ║
║  → Type là built-in hoặc primitive (int, string, time.Time)        ║
║  → Method không cần modify và struct nhỏ                          ║
║                                                                      ║
║  LƯU Ý: Nếu bất kỳ method nào dùng *T,                             ║
║  toàn bộ method set nên dùng *T (nhất quán, dễ đoán)            ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.3 — Interfaces (★★ Cốt Lõi)

```
╔══════════════════════════════════════════════════════════════════════╗
║  INTERFACE = HỢP ĐỒNG (CONTRACT)                                   ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Interface Định Nghĩa (Phía Domain)                               ║
║  +---------------------------+                                       ║
║  |  Shape                    |                                       ║
║  |---------------------------|                                       ║
║  |  Area() float64           |  ← Contract (Port)                   ║
║  |  Perimeter() float64      |                                       ║
║  +---------------------------+                                       ║
║         ▲               ▲                                            ║
║         ┌───────┤       ┌───────┤                                    ║
║  Rectangle             Circle                                        ║
║  (Adapter)             (Adapter)                                     ║
║                                                                      ║
║  "Implement implicitly" — không cần ký hợp                        ║
║  Chỉ cần có method signatures khớp                               ║
╚══════════════════════════════════════════════════════════════════════╝
```

```go
// Định nghĩa interface
type Shape interface {
    Area() float64
    Perimeter() float64
}

// Infinite types implement Shape implicitly —
// bất kỳ struct nào có 2 method này đều là Shape
type Rectangle struct{ Width, Height float64 }
func (r Rectangle) Area() float64      { return r.Width * r.Height }
func (r Rectangle) Perimeter() float64 { return 2 * (r.Width + r.Height) }

type Circle struct{ Radius float64 }
func (c Circle) Area() float64      { return math.Pi * c.Radius * c.Radius }
func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.Radius }

type Triangle struct{ A, B, C float64 } // 3 cạnh
func (t Triangle) Area() float64 {
    s := (t.A + t.B + t.C) / 2 // Heron's formula
    return math.Sqrt(s * (s - t.A) * (s - t.B) * (s - t.C))
}
func (t Triangle) Perimeter() float64 { return t.A + t.B + t.C }

// Hàm này hoạt động với Rectangle, Circle, Triangle — và mọi shape tương lai!
func printShapeInfo(s Shape) {
    fmt.Printf("Type: %T, Area: %.2f, Perimeter: %.2f\n",
        s, s.Area(), s.Perimeter())
}

// Slice của interface — có thể mix các types khác nhau!
shapes := []Shape{
    Rectangle{Width: 3, Height: 4},
    Circle{Radius: 5},
    Triangle{A: 3, B: 4, C: 5},
}
for _, s := range shapes {
    printShapeInfo(s)
}
// Type: main.Rectangle, Area: 12.00, Perimeter: 14.00
// Type: main.Circle, Area: 78.54, Perimeter: 31.42
// Type: main.Triangle, Area: 6.00, Perimeter: 12.00
```

```go
// INTERFACE COMPOSITION — ghép nhiều interfaces thành 1 interface lớn hơn
// Này là cách stdlib xây dựng io.ReadWriter, io.ReadWriteCloser...

type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Closer interface { Close() error }

// Compose
type ReadWriter interface {
    Reader  // embed Reader
    Writer  // embed Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// Áp dụng trong Clean Arch — interface segregation principle (ISP):
// Không tạo 1 interface khổng lồ, tách thành nhỏ và compose lại

type UserReader interface {
    FindByID(ctx context.Context, id string) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
    List(ctx context.Context, filter ListFilter) ([]*User, error)
}

type UserWriter interface {
    Save(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}

// Full repository = Read + Write
type UserRepository interface {
    UserReader
    UserWriter
}

// UseCase chỉ cần đọc — inject interface nhỏ hơn → dễ test hơn!
type getUserUseCase struct{ repo UserReader }
```

**Liên hệ với Clean Architecture:**

```go
// ============================================================
// Domain layer: Định nghĩa CONTRACT (không biết implementation)
// ============================================================

// internal/domain/repository/user_repository.go
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
    Save(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}

// ============================================================
// Infrastructure layer: IMPLEMENT contract (biết Postgres)
// ============================================================

// internal/repository/postgres/user_repository.go
type postgresUserRepo struct {
    db *sql.DB
}

// Gói này CONCRẪTELY implement domain interface
// KHÔNG cần declare 'implements UserRepository'
func (r *postgresUserRepo) FindByID(ctx context.Context, id string) (*User, error) {
    // ... SQL query
}
func (r *postgresUserRepo) Save(ctx context.Context, user *User) error {
    // ... INSERT query
}

// Compile-time check — fail nếu thiếu method!
var _ UserRepository = (*postgresUserRepo)(nil)

// ============================================================
// UseCase layer: CHỈ biết interface, không cần biết Postgres hay MySQL
// ============================================================

type userUseCase struct {
    repo UserRepository // inject interface, không phải concrete type!
}

func (uc *userUseCase) GetUser(ctx context.Context, id string) (*User, error) {
    return uc.repo.FindByID(ctx, id) // gọi qua interface
}

// Để test: inject mock thay vì Postgres
type mockUserRepo struct{}
func (m *mockUserRepo) FindByID(_ context.Context, id string) (*User, error) {
    return &User{ID: id, Name: "Mock"}, nil
}
var _ UserRepository = (*mockUserRepo)(nil)

func TestGetUser(t *testing.T) {
    uc := &userUseCase{repo: &mockUserRepo{}} // inject mock!
    user, err := uc.GetUser(context.Background(), "123")
    // ... assert
}
```

### 4.4 — Interface Values & nil

```go
// Interface value = (type, value) — một cặp gồm concrete type + concrete value

var s Shape // nil interface: (nil, nil)
fmt.Println(s == nil) // true

s = Rectangle{Width: 3, Height: 4} // (Rectangle, Rectangle{3,4})
s = &Circle{Radius: 5}             // (*Circle, 0xc000...) — pointer!

// Interface value với nil UNDERLYING VALUE — gật khọn!
type Writer interface{ Write() }
type MyWriter struct{}
func (w *MyWriter) Write() { fmt.Println("writing") }

var w *MyWriter = nil // nil pointer, nhưng có type
var iw Writer = w    // iw = (*MyWriter, nil) — không phải nil interface!

fmt.Println(w == nil)  // true
fmt.Println(iw == nil) // FALSE! — iw có type info
// iw.Write() — sẽ gọi method, nhưng receiver là nil pointer → panic!
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  INTERFACE NIL GOTCHA                                                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Nil interface:  (nil, nil)       → == nil TRUE                     ║
║  Non-nil interface: (*T, nil)     → == nil FALSE (GỘI BỬP!)        ║
║                                                                      ║
║  Rule: function trả về interface, KHÔNG return typed nil pointer!   ║
║                                                                      ║
║  SAI:   func get() Shape { var r *Rectangle; return r } → non-nil!  ║
║  ĐÚNG:  func get() Shape { return nil }                             ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.5 — Empty Interface & Type Assertions

```go
// EMPTY INTERFACE (interface{} hoặc any trong Go 1.18+)
// Giữ bất kỳ value nào — như Object trong Java
var anything interface{}
anything = 42
anything = "hello"
anything = []int{1, 2, 3}

// Phổ biến trong func nhận nhiều loại input:
func log(v any) {
    fmt.Printf("[LOG] %T: %v\n", v, v)
}

// TYPE ASSERTION — lấy concrete value từ interface
var i interface{} = "hello"

// Dạng 1: panic nếu sai type
s := i.(string)         // OK: s = "hello"
n := i.(int)            // PANIC: interface holds string, not int

// Dạng 2: "comma ok" — an toàn hơn
s, ok := i.(string)     // ok=true, s="hello"
n, ok := i.(int)        // ok=false, n=0 (zero value)

// TYPE SWITCH — kiểm tra nhiều type một lần
func describe(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("int(%d)", v)
    case string:
        return fmt.Sprintf("string(%q)", v)
    case bool:
        return fmt.Sprintf("bool(%v)", v)
    case []int:
        return fmt.Sprintf("[]int(len=%d)", len(v))
    default:
        return fmt.Sprintf("unknown(%T)", v)
    }
}
```

### 4.6 — Stringer Interface

```go
// fmt.Stringer — interface quan trọng trong stdlib
// Nếu type implement String() string, fmt.Println sẽ dùng nó

type Stringer interface {
    String() string
}

// Áp dụng cho domain entity:
type User struct {
    ID    string
    Name  string
    Email string
}

func (u User) String() string {
    return fmt.Sprintf("User{ID:%s, Name:%s}", u.ID, u.Name)
}

u := User{ID: "123", Name: "John", Email: "john@example.com"}
fmt.Println(u) // User{ID:123, Name:John} — gọi String() tự động

// Tương tự với Weekday được biết đến hơn:
type Direction int
const (North Direction = iota; East; South; West)
func (d Direction) String() string {
    return [...]string{"North", "East", "South", "West"}[d]
}
fmt.Println(North) // "North" thay vì "0"
```

### 4.7 — Error Interface (★★ Production Critical)

```go
// error là built-in interface trong Go:
// type error interface {
//     Error() string
// }

// Tạo custom error bằng implement interface:
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on field '%s': %s", e.Field, e.Message)
}

// Dùng:
func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "must be non-negative"}
    }
    if age > 150 {
        return &ValidationError{Field: "age", Message: "unrealistic value"}
    }
    return nil // thành công — nil error
}

err := validateAge(-5)
if err != nil {
    // errors.As — unwrap sang concrete type để inspect fields
    var ve *ValidationError
    if errors.As(err, &ve) {
        fmt.Println("Field:", ve.Field) // Field: age
    }
}

// SENTINEL ERRORS — pre-defined error values cho comparison
var (
    ErrNotFound   = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
)

// errors.Is — kiểm tra kể cả wrapped errors (%w)
func getUser(id string) error {
    return fmt.Errorf("getUser %s: %w", id, ErrNotFound) // wrap
}

err = getUser("123")
fmt.Println(errors.Is(err, ErrNotFound)) // true (dù đã wrap)
```

```
╔══════════════════════════════════════════════════════════════════════╗
║  SO SÁNH ERROR HANDLING                                              ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  errors.New("msg") → simple sentinel, không có context thêm        ║
║  fmt.Errorf("%w", err) → wrap, giữ original error chain             ║
║  custom struct       → có fields (Field, Code, Detail...)           ║
║                                                                      ║
║  errors.Is(err, target) → kiểm tra sentinel (kể cả wrapped)       ║
║  errors.As(err, &target) → unwrap ra concrete type                  ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 4.8 — io.Reader & io.Writer

```go
// Hai interface quan trọng nhất trong stdlib:
//
// type Reader interface {
//     Read(p []byte) (n int, err error)
// }
// type Writer interface {
//     Write(p []byte) (n int, err error)
// }

// io.Reader được implement bởi: os.File, http.Request.Body, strings.Reader,
//     bytes.Buffer, net.Conn, gzip.Reader...
// Cùng một interface → có thể viết function xử lý dữ liệu từ bất kỳ nguồn nào!

// Ví dụ: đọc từ bất kỳ Reader nào — file, HTTP body, in-memory...
func processData(r io.Reader) error {
    data, err := io.ReadAll(r)
    if err != nil {
        return fmt.Errorf("processData: %w", err)
    }
    fmt.Printf("Read %d bytes\n", len(data))
    return nil
}

// Test với strings.Reader (không cần file thật!):
processData(strings.NewReader("test data"))

// Production: đọc từ HTTP request body:
http.HandleFunc("/upload", func(w http.ResponseWriter, r *http.Request) {
    processData(r.Body) // r.Body implement io.Reader!
})

// Decorator pattern với io.Reader/Writer:
// io.TeeReader — đọc và ghi đồng thời
// io.LimitReader — giới hạn số byte đọc
// gzip.NewReader(r) — decompress từ bất kỳ io.Reader
```

### 4.9 — Một Số Interface Quan Trọng Trong Stdlib

| Interface | Package | Mục đích | Ví dụ impl |
|-----------|---------|---------|------------|
| `error` | builtin | Error handling | `*os.PathError`, custom errors |
| `fmt.Stringer` | fmt | Human-readable string | `time.Time`, `net.IP` |
| `io.Reader` | io | Đọc dữ liệu | `os.File`, `http.Response.Body` |
| `io.Writer` | io | Ghi dữ liệu | `os.File`, `bytes.Buffer` |
| `io.Closer` | io | Close resource | `os.File`, `net.Conn` |
| `http.Handler` | net/http | Xử lý HTTP request | Gin, Chi, stdlib mux |
| `sort.Interface` | sort | Sắp xếp | Bất kỳ slice có len+swap+less |
| `context.Context` | context | Deadline, cancel, values | `context.Background()` |

---

## Bài Tập Tổng Hợp

### Exercise 1 — FizzBuzz với Go idioms

```go
// Viết FizzBuzz (1-100) theo chuẩn Go:
// - Chia hết 15 → "FizzBuzz"
// - Chia hết 3  → "Fizz"
// - Chia hết 5  → "Buzz"
// - Còn lại     → số đó

// Gợi ý: dùng switch không điều kiện
func fizzBuzz(n int) string {
    switch {
    case n%15 == 0:
        return "FizzBuzz"
    case n%3 == 0:
        return "Fizz"
    case n%5 == 0:
        return "Buzz"
    default:
        return strconv.Itoa(n)
    }
}
```

### Exercise 2 — Fibonacci Closure

```go
// Implement fibonacci() trả về closure
// Mỗi lần gọi closure → giá trị Fibonacci tiếp theo
// f := fibonacci()  →  f() = 0, f() = 1, f() = 1, f() = 2, f() = 3...

func fibonacci() func() int {
    a, b := 0, 1
    return func() int {
        result := a
        a, b = b, a+b
        return result
    }
}
```

### Exercise 3 — WordCount với Map

```go
// Đếm số lần xuất hiện của mỗi từ trong string
// "I am learning Go and I love Go" → map[I:2 am:1 and:1 learning:1 love:1 Go:2]

func wordCount(s string) map[string]int {
    counts := make(map[string]int)
    for _, word := range strings.Fields(s) {
        counts[word]++
    }
    return counts
}
```

### Exercise 4 — Slice của Slices (Matrix)

```go
// Tạo matrix n×n, fill giá trị theo index
// Gợi ý: make([][]int, n) để tạo slice of slices

func makeMatrix(n int) [][]int {
    matrix := make([][]int, n)
    for i := range matrix {
        matrix[i] = make([]int, n)
        for j := range matrix[i] {
            matrix[i][j] = i*n + j
        }
    }
    return matrix
}
```

---

## Tổng Kết — Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║  SAU LESSON 00, BẠN NẮM ĐƯỢC:                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ✅ Exported names: Chữ Hoa = public, chữ thường = private         ║
║                                                                      ║
║  ✅ Functions: multiple return values, named returns                ║
║     → Pattern: (result, error)                                      ║
║                                                                      ║
║  ✅ Variables: var vs := , zero values, type inference              ║
║                                                                      ║
║  ✅ Flow control: for (1 loop), if/else, switch (no fall-through)   ║
║     → defer: LIFO, cleanup pattern                                  ║
║                                                                      ║
║  ✅ Pointers: & (address-of), * (dereference), dot notation         ║
║                                                                      ║
║  ✅ Structs: embedding thay inheritance, named field init           ║
║                                                                      ║
║  ✅ Slices: pointer + len + cap, reference semantics, append        ║
║     → SLICE GOTCHA: sharing underlying array                        ║
║                                                                      ║
║  ✅ Maps: make, comma-ok, không đảm bảo thứ tự                     ║
║                                                                      ║
║  ✅ Closures: capture by reference, middleware pattern              ║
║     → GOROUTINE GOTCHA: closure + loop variable                     ║
║                                                                      ║
║  ✅ Methods: value vs pointer receiver, khi nào dùng cái nào       ║
║                                                                      ║
║  ★ Interfaces: implicit impl, (type, value) pair                    ║
║     → INTERFACE NIL GOTCHA: (*T, nil) != nil                        ║
║     → Liên hệ trực tiếp với Clean Architecture Ports             ║
║                                                                      ║
║  ★ Error interface: sentinel errors, wrapping (%w), errors.Is/As    ║
║                                                                      ║
║  ★ io.Reader/Writer: decorator pattern, testability                 ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

> **Câu hỏi tự kiểm tra:**
>
> 1. Tại sao Go không có `while`? (Hint: simplicity)
> 2. `var s []int` và `s := []int{}` khác nhau ở điểm nào quan trọng?
> 3. Khi nào `append` tạo new underlying array? Ảnh hưởng gì đến performance?
> 4. Tại sao iterating map không có thứ tự? (Hint: security — hash randomization)
> 5. Closure `return func() int { count++ }` — `count` được lưu ở đâu (stack hay heap)?
> 6. Tại sao `(*T, nil)` không phải nil interface? Ảnh hưởng gì trong thực tế?
> 7. Khi nào dùng value receiver, khi nào dùng pointer receiver?
> 8. `errors.Is` vs `errors.As` — cái nào dùng khi nào?

---

*📖 Next: [Lesson 01 — Project Setup & Clean Architecture](./lesson-01-project-setup-and-architecture.md)*
