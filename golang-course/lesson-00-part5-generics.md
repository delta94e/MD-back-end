# Part 5 — Generics (Go 1.18+)

> **Generics** (hay Type Parameters) là tính nang lớn nhat cua Go ke tu phien ban 1.18 (2022).
> Truoc Generics, code Go hay bi lap lai hoac phai dung `interface{}` mat di type safety.

---

## 5.1 — Van De Truoc Khi Co Generics

```go
// PROBLEM 1: Ham lap lai cho moi type
// Can viet rieng cho int, float64, string...

func SumInts(nums []int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func SumFloat64s(nums []float64) float64 {
    total := 0.0
    for _, n := range nums {
        total += n
    }
    return total
}
// Code giong het nhau, chi khac type!

// PROBLEM 2: Dung interface{} - mat type safety
func SumAny(nums []interface{}) interface{} {
    // Phai type assert, de panic, compiler khong kiem tra
    total := 0
    for _, n := range nums {
        total += n.(int) // panic neu khong phai int!
    }
    return total
}
```

---

## 5.2 — Type Parameters (Ham Generic)

```go
// SYNTAX: func FuncName[T Constraint](params) returnType
// T la "type parameter" - placeholder cho type cu the

// Ham Sum generic - hoat dong voi NHIEU type
func Sum[T int | float64 | int64](nums []T) T {
    var total T  // zero value cua T
    for _, n := range nums {
        total += n
    }
    return total
}

// Goi - compiler suy ra type tu argument:
ints := []int{1, 2, 3, 4, 5}
fmt.Println(Sum(ints))           // 15 (Go suy ra T = int)
fmt.Println(Sum[int](ints))      // 15 (explicit type argument)

floats := []float64{1.1, 2.2, 3.3}
fmt.Println(Sum(floats))         // 6.6 (Go suy ra T = float64)

// Ham map/filter generic
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

func Filter[T any](slice []T, fn func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}

// Su dung:
nums := []int{1, 2, 3, 4, 5, 6}

// Nhan doi tat ca
doubled := Map(nums, func(n int) int { return n * 2 })
fmt.Println(doubled) // [2 4 6 8 10 12]

// Chuyen int -> string
strs := Map(nums, func(n int) string { return fmt.Sprintf("item%d", n) })
fmt.Println(strs) // [item1 item2 item3 item4 item5 item6]

// Loc so chan
evens := Filter(nums, func(n int) bool { return n%2 == 0 })
fmt.Println(evens) // [2 4 6]
```

---

## 5.3 — Constraints — Gioi Han Type

```go
// Constraint = quy dinh T co the la type nao
// Duoc viet nhu interface

// Constraint 1: "any" - bao gio cung co san, nhu interface{}
func Print[T any](v T) {
    fmt.Println(v) // chi dung cac operation co tren MOI type
}

// Constraint 2: "comparable" - co san, T phai support == va !=
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

// Su dung:
fmt.Println(Contains([]int{1, 2, 3}, 2))          // true
fmt.Println(Contains([]string{"a", "b"}, "c"))   // false
// fmt.Println(Contains([][]int{{1}}, []int{1})) // COMPILE ERROR: slice khong comparable

// Constraint 3: Custom constraint voi union type (|)
type Number interface {
    int | int8 | int16 | int32 | int64 |
        float32 | float64
}

func Min[T Number](a, b T) T {
    if a < b {
        return a
    }
    return b
}

func Max[T Number](a, b T) T {
    if a > b {
        return a
    }
    return b
}

fmt.Println(Min(3, 5))       // 3
fmt.Println(Min(3.14, 2.71)) // 2.71
fmt.Println(Max("a", "z"))   // COMPILE ERROR: string khong thoa Number

// Constraint 4: ~ (underlying type) - bao gom ca type aliases
type MyInt int  // underlying type la int

type Integer interface {
    ~int | ~int32 | ~int64  // ~ = "bao gom ca type co underlying type la..."
}

func Double[T Integer](n T) T {
    return n * 2
}

var x MyInt = 5
fmt.Println(Double(x))  // 10 (hoat dong voi MyInt vi ~int)
// Neu khong co ~ : Double(x) se fail vi MyInt khac int
```

---

## 5.4 — Generic Types (Struct, Interface Generic)

```go
// Generic STRUCT - type co type parameter

// Stack generic - hoat dong voi bat ky type nao
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T  // zero value cua T
    if len(s.items) == 0 {
        return zero, false
    }
    last := len(s.items) - 1
    item := s.items[last]
    s.items = s.items[:last]
    return item, true
}

func (s *Stack[T]) Peek() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) Len() int {
    return len(s.items)
}

func (s *Stack[T]) IsEmpty() bool {
    return len(s.items) == 0
}

// Su dung Stack[int]:
intStack := &Stack[int]{}
intStack.Push(1)
intStack.Push(2)
intStack.Push(3)

if v, ok := intStack.Pop(); ok {
    fmt.Println(v) // 3
}
fmt.Println(intStack.Len()) // 2

// Su dung Stack[string]:
strStack := &Stack[string]{}
strStack.Push("hello")
strStack.Push("world")
if v, ok := strStack.Peek(); ok {
    fmt.Println(v) // "world" (khong xoa)
}
```

```go
// Generic Pair - giu 2 gia tri co the khac type
type Pair[K, V any] struct {
    Key   K
    Value V
}

func NewPair[K, V any](key K, value V) Pair[K, V] {
    return Pair[K, V]{Key: key, Value: value}
}

func (p Pair[K, V]) String() string {
    return fmt.Sprintf("(%v, %v)", p.Key, p.Value)
}

p1 := NewPair("name", "Alice")
fmt.Println(p1) // (name, Alice)

p2 := NewPair(1, true)
fmt.Println(p2) // (1, true)

p3 := NewPair("score", 99.5)
fmt.Println(p3.Key)   // "score"
fmt.Println(p3.Value) // 99.5

// Slice of pairs:
pairs := []Pair[string, int]{
    NewPair("alice", 95),
    NewPair("bob", 87),
    NewPair("carol", 92),
}
for _, p := range pairs {
    fmt.Printf("%s scored %d\n", p.Key, p.Value)
}
```

```go
// Generic Map (wrapper an toan hon map built-in):
type SafeMap[K comparable, V any] struct {
    data map[K]V
}

func NewSafeMap[K comparable, V any]() *SafeMap[K, V] {
    return &SafeMap[K, V]{data: make(map[K]V)}
}

func (m *SafeMap[K, V]) Set(key K, value V) {
    m.data[key] = value
}

func (m *SafeMap[K, V]) Get(key K) (V, bool) {
    v, ok := m.data[key]
    return v, ok
}

func (m *SafeMap[K, V]) GetOrDefault(key K, def V) V {
    if v, ok := m.data[key]; ok {
        return v
    }
    return def
}

func (m *SafeMap[K, V]) Delete(key K) {
    delete(m.data, key)
}

func (m *SafeMap[K, V]) Len() int {
    return len(m.data)
}

// Su dung:
scores := NewSafeMap[string, int]()
scores.Set("Alice", 95)
scores.Set("Bob", 87)

alice, ok := scores.Get("Alice")
fmt.Println(alice, ok) // 95 true

unknown := scores.GetOrDefault("Dave", 0)
fmt.Println(unknown) // 0 (default)
```

---

## 5.5 — Generic Interfaces va Advanced Constraints

```go
// Constraint tu interface - ket hop method va type

// Ordered: type co the so sanh thu tu
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
        ~float32 | ~float64 | ~string
}

func Sort[T Ordered](slice []T) []T {
    result := make([]T, len(slice))
    copy(result, slice)
    // sort logic (don gian - bubble sort de demo)
    for i := 0; i < len(result); i++ {
        for j := i + 1; j < len(result); j++ {
            if result[j] < result[i] {
                result[i], result[j] = result[j], result[i]
            }
        }
    }
    return result
}

fmt.Println(Sort([]int{5, 2, 8, 1, 9}))         // [1 2 5 8 9]
fmt.Println(Sort([]string{"banana", "apple"}))   // [apple banana]
fmt.Println(Sort([]float64{3.14, 1.41, 2.71}))  // [1.41 2.71 3.14]

// Constraint ket hop method + type (Go 1.18):
// Luu y: interface constraint CHI co the dung type union khi KHONG co method trong constraint do
// - Muon co method: dung interface rieng
// - Muon union type: dung interface rieng
// - Muon ca hai: constraint interface embed ca hai

type Stringer interface {
    String() string
}

// Ham chi nhan type co String() method
func PrintAll[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}
```

---

## 5.6 — Slices Package (stdlib generics)

```go
// Tu Go 1.21, stdlib co nhieu ham generic san:
import "slices"
import "maps"

// slices package:
nums := []int{5, 2, 8, 1, 9, 3}

slices.Sort(nums)           // sort in-place
fmt.Println(nums)           // [1 2 3 5 8 9]

idx := slices.Index(nums, 5)  // tim index
fmt.Println(idx)            // 3

found := slices.Contains(nums, 7)
fmt.Println(found)          // false

max := slices.Max(nums)
min := slices.Min(nums)
fmt.Println(max, min)       // 9 1

reversed := slices.Collect(slices.Backward(nums)) // reversed order

// maps package:
m := map[string]int{"a": 1, "b": 2, "c": 3}

keys := maps.Keys(m)              // lay tat ca keys (khong thu tu)
values := maps.Values(m)          // lay tat ca values

clone := maps.Clone(m)            // deep copy
maps.DeleteFunc(clone, func(k string, v int) bool {
    return v < 2                  // xoa entry co value < 2
})

// Built-in functions generic (Go 1.21+):
a, b := min(3, 5), max(3, 5)     // min/max built-in
fmt.Println(a, b)                 // 3 5

s := []int{1, 2, 3}
clear(s)                          // xoa het elements (set zero value)
fmt.Println(s)                    // [0 0 0]
```

---

## 5.7 — Ung Dung Thuc Te Trong Production

```go
// Pattern 1: Generic Repository Interface (Clean Architecture)
// Giam lap code cho tung entity

type Repository[T any, ID comparable] interface {
    FindByID(ctx context.Context, id ID) (*T, error)
    FindAll(ctx context.Context) ([]*T, error)
    Save(ctx context.Context, entity *T) error
    Update(ctx context.Context, entity *T) error
    Delete(ctx context.Context, id ID) error
}

// Implement cho User:
type userRepository struct{ db *sql.DB }

func (r *userRepository) FindByID(ctx context.Context, id string) (*User, error) {
    // SQL query...
}
// ... implement con lai

var _ Repository[User, string] = (*userRepository)(nil) // compile check

// Implement cho Product:
type productRepository struct{ db *sql.DB }
var _ Repository[Product, int64] = (*productRepository)(nil)
```

```go
// Pattern 2: Generic Result type (Option/Result pattern)
type Result[T any] struct {
    value T
    err   error
}

func Success[T any](v T) Result[T] {
    return Result[T]{value: v}
}

func Failure[T any](err error) Result[T] {
    return Result[T]{err: err}
}

func (r Result[T]) IsOK() bool      { return r.err == nil }
func (r Result[T]) Unwrap() T       { return r.value }
func (r Result[T]) Error() error    { return r.err }

func (r Result[T]) UnwrapOr(def T) T {
    if r.err != nil {
        return def
    }
    return r.value
}

// Su dung:
func divide(a, b float64) Result[float64] {
    if b == 0 {
        return Failure[float64](errors.New("division by zero"))
    }
    return Success(a / b)
}

result := divide(10, 3)
if result.IsOK() {
    fmt.Printf("%.2f\n", result.Unwrap()) // 3.33
}

safe := divide(10, 0).UnwrapOr(-1)
fmt.Println(safe) // -1
```

```go
// Pattern 3: Generic Cache
type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]cacheItem[V]
    ttl   time.Duration
}

type cacheItem[V any] struct {
    value     V
    expiresAt time.Time
}

func NewCache[K comparable, V any](ttl time.Duration) *Cache[K, V] {
    return &Cache[K, V]{
        items: make(map[K]cacheItem[V]),
        ttl:   ttl,
    }
}

func (c *Cache[K, V]) Set(key K, value V) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = cacheItem[V]{
        value:     value,
        expiresAt: time.Now().Add(c.ttl),
    }
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    item, ok := c.items[key]
    if !ok || time.Now().After(item.expiresAt) {
        var zero V
        return zero, false
    }
    return item.value, true
}

// Su dung:
userCache := NewCache[string, *User](5 * time.Minute)
userCache.Set("usr_123", &User{ID: "123", Name: "Alice"})

if user, ok := userCache.Get("usr_123"); ok {
    fmt.Println(user.Name) // Alice
}
```

---

## 5.8 — Gioi Han Cua Generics Trong Go

```go
// Go generics CO MOT SO HAN CHE (khac Java/Rust):

// 1. Khong the dung type parameter cho receiver cua method
type MySlice[T any] []T

// OK: method tren generic type
func (s MySlice[T]) Len() int { return len(s) }

// KHONG OK: them type parameter moi vao method
// func (s MySlice[T]) Map[U any](fn func(T) U) MySlice[U] { ... }
// → Phai dung ham, khong phai method

func MapSlice[T, U any](s []T, fn func(T) U) []U { ... } // ham OK

// 2. Khong the type assert tren type parameter
func bad[T any](v T) {
    // s := v.(string) // COMPILE ERROR
    // Phai dung reflect hoac any
}

// 3. Khong co specialization (khac C++ templates)
// Tat ca types dung cung 1 implementation
// Khong the viet logic khac nhau cho int vs string

// 4. Struct co type parameter phai specify khi dung
// var s Stack // COMPILE ERROR - thieu [T]
var s Stack[int] // OK

// KHI NAO DUNG GENERICS:
// + Ham/type ap dung giong het nhau cho nhieu types
// + Muon type safety hon interface{}
// + Thu vien co tat ca: containers, algorithms, utilities

// KHI NAO KHONG DUNG GENERICS:
// - Behavior khac nhau theo type -> dung interface
// - Chi co 1-2 types -> code rieng ho n gian hon
// - Team chua quen generics -> doc kho hon
```

---

## Tong Ket Generics

```
╔══════════════════════════════════════════════════════════════════════╗
║  GENERICS TRONG GO - KEY POINTS                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Truoc Go 1.18:     interface{} + type assertion -> mat safety       ║
║  Tu Go 1.18:        Type parameters -> giu type safety              ║
║                                                                      ║
║  Syntax:  func Fn[T Constraint](v T) T                              ║
║  Struct:  type Box[T any] struct { value T }                        ║
║                                                                      ║
║  Constraints co san:                                                ║
║  any        = interface{} - chap nhan moi type                      ║
║  comparable = support == va != (int, string, struct, ...)           ║
║                                                                      ║
║  Custom Constraint = interface voi union type:                      ║
║  type Number interface { int | float64 | ... }                      ║
║  ~int = int va moi type co underlying type la int                   ║
║                                                                      ║
║  stdlib generics: slices, maps, cmp (Go 1.21+)                     ║
║                                                                      ║
║  Han che:                                                           ║
║  - Khong co method type parameter                                   ║
║  - Khong co specialization                                          ║
║  - Compile cham hon mot chut                                        ║
╚══════════════════════════════════════════════════════════════════════╝
```

**Cau hoi tu kiem tra:**

1. Tai sao `Sum[T any]` khong compile duoc nhung `Sum[T Number]` thi duoc?
2. `comparable` constraint bao gom nhung type nao? Slice va map co O khong?
3. `~int` khac `int` o diem nao? Khi nao can dung `~`?
4. Tai sao khong the them type parameter vao method trong Go?
5. `slices.Sort` yeu cau constraint gi? Tai sao khong the sort `[][]int`?
6. Generic Repository `Repository[T, ID]` co loi ich gi so voi interface binh thuong?

---

*Sau khi nam vung Generics -> Ban da hoan thanh toan bo Go Fundamentals!*
*Tiep theo: [Lesson 01 - Clean Architecture](./lesson-01-project-setup-and-architecture.md)*
