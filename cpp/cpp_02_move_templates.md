# C++ Mastery Guide — Document 2 of 5
## Move Semantics, Templates & Generic Programming

> **Scope:** C++11 move semantics from first principles, template metaprogramming, variadic templates, SFINAE, concepts (C++20), and the full generics toolbox. These are the topics that separate a solid C++ programmer from an expert one — interviewers at BMW (embedded C++) will probe these deeply.

---

## Table of Contents

1. [Move Semantics From First Principles](#1-move-semantics-from-first-principles)
2. [std::move and std::forward](#2-stdmove-and-stdforward)
3. [Universal References & Reference Collapsing](#3-universal-references--reference-collapsing)
4. [Perfect Forwarding](#4-perfect-forwarding)
5. [Function Templates](#5-function-templates)
6. [Class Templates](#6-class-templates)
7. [Template Specialization](#7-template-specialization)
8. [Variadic Templates](#8-variadic-templates)
9. [SFINAE & enable_if](#9-sfinae--enable_if)
10. [if constexpr (C++17)](#10-if-constexpr-c17)
11. [Concepts (C++20)](#11-concepts-c20)
12. [Template Metaprogramming (TMP)](#12-template-metaprogramming-tmp)
13. [Policy-Based Design](#13-policy-based-design)
14. [CRTP — Curiously Recurring Template Pattern](#14-crtp--curiously-recurring-template-pattern)
15. [Type Traits & compile-time Introspection](#15-type-traits--compile-time-introspection)

---

## 1. Move Semantics From First Principles

### The Problem Move Semantics Solves

Before C++11, every object transfer involved copying — even when the source was about to be destroyed:

```cpp
// Pre-C++11: adding element to vector
std::vector<std::string> vec;
std::string s = "a very long string that allocates heap memory";
vec.push_back(s);   // COPIES s into the vector
                    // s still exists after this — two heap allocations!

// Also pre-C++11: returning from function always copies:
std::vector<int> make_vector() {
    std::vector<int> v(10000);
    return v;  // copies all 10000 ints!  (unless NRVO kicks in)
}
```

### What Move Semantics Provides

Move: transfer ownership of a resource without copying it. The "moved-from" object is left in a valid but unspecified state (typically empty/null).

```cpp
// The idea:
class DmaBuffer {
    uint8_t* data_;
    size_t   size_;
public:
    // Move constructor: steal the pointer, leave source empty
    DmaBuffer(DmaBuffer&& other) noexcept
        : data_(other.data_)
        , size_(other.size_) {
        other.data_ = nullptr;  // source is now "empty" — valid but unspecified
        other.size_ = 0;
    }
};

DmaBuffer a(4096);  // allocates 4KB
DmaBuffer b(std::move(a));  // b gets a's buffer, a is now empty
// a.data_ == nullptr, b.data_ points to the 4KB
// Cost: 3 pointer assignments. No heap allocation.
```

### Rvalue References Enable Move

```cpp
void sink(DmaBuffer&& buf) {
    // buf is an rvalue reference — we can move from it
    DmaBuffer local = std::move(buf);  // move into local
}

DmaBuffer make() {
    DmaBuffer buf(1024);
    return buf;  // NRVO or implicit move
}

DmaBuffer d = make();  // no copy — either NRVO (elision) or move
```

### Move vs Copy: When Each Happens

```cpp
DmaBuffer a(4096);
DmaBuffer b = a;              // COPY: a is lvalue, has a name, still needed
DmaBuffer c = std::move(a);  // MOVE: explicitly said "a is done"
DmaBuffer d = make();         // MOVE (or RVO/NRVO — even better: no call at all)

// In function calls:
void f(DmaBuffer);
f(a);              // COPY (a still needed after)
f(std::move(a));   // MOVE (a's value transferred, a now empty)
f(DmaBuffer(512)); // MOVE from temporary (compiler might elide this entirely)
```

### The Moved-From State

After a move, the object must remain in a **valid but unspecified** state — destructible and assignable, but with no guaranteed content:

```cpp
std::string s = "hello";
std::string t = std::move(s);

// s is now valid but unspecified:
// s.size() might be 0 (most implementations)
// But the standard only guarantees: s is destructible and s = "new value" works
s = "world";  // OK: reassign the moved-from string
```

### Copy Elision and RVO/NRVO

C++17 guarantees copy elision in certain cases — no move, no copy, just construction in-place:

```cpp
// RVO (Return Value Optimization): guaranteed since C++17 for prvalues
std::string make_string() {
    return std::string("hello");  // no copy, no move: constructed directly in caller's space
}

// NRVO (Named RVO): not guaranteed, but most compilers do it
std::string make_named() {
    std::string s = "hello";
    return s;  // NRVO: typically optimized, but add std::move here = prevents NRVO!
               // Never: return std::move(s);  -- prevents NRVO!
}
```

**Interview gotcha:** Never write `return std::move(local_var)` — it prevents NRVO and actually makes things worse!

---

## 2. std::move and std::forward

### What `std::move` Actually Is

`std::move` does NOT move anything. It's a cast to rvalue reference that enables the compiler to use the move constructor/assignment:

```cpp
// std::move is just this (simplified):
template<typename T>
typename std::remove_reference<T>::type&&
move(T&& t) noexcept {
    return static_cast<typename std::remove_reference<T>::type&&>(t);
}

// All it does: unconditionally cast to rvalue reference
// The ACTUAL move happens when a move constructor/assignment uses the result
```

```cpp
std::string s = "hello";
std::string t = std::move(s);  // std::move(s) casts s to string&&
                                // Then string's move ctor is called: t steals s's buffer
```

### When NOT to Use `std::move`

```cpp
// 1. Don't move a const object (move ctor takes non-const &&, falls back to copy)
const std::string cs = "hello";
std::string t = std::move(cs);  // Silently copies! const T&& can't be stolen from

// 2. Don't move when returning a local variable (prevents NRVO)
std::string bad() {
    std::string s = "hello";
    return std::move(s);  // BAD: disables NRVO, may actually be slower
}
std::string good() {
    std::string s = "hello";
    return s;  // GOOD: compiler applies NRVO
}

// 3. Don't use moved-from object (unless you reassign it first)
std::vector<int> v = {1, 2, 3};
auto w = std::move(v);
// v is empty now — don't call v.size() expecting 3
v.push_back(4);  // OK: assigning to moved-from is fine
```

---

## 3. Universal References & Reference Collapsing

### Universal References (Forwarding References)

`T&&` where `T` is a deduced template parameter is NOT always an rvalue reference — it's a **universal reference** (Scott Meyers' term) that can bind to either lvalues or rvalues:

```cpp
template<typename T>
void process(T&& val) {  // T&& here is a UNIVERSAL REFERENCE, not rvalue reference
    // val might be an lvalue ref or rvalue ref depending on what's passed
}

int x = 42;
process(x);        // T deduced as int&,  val is int& (lvalue reference)
process(42);       // T deduced as int,   val is int&& (rvalue reference)
process(std::move(x));  // T deduced as int, val is int&&
```

### Reference Collapsing Rules

When references to references form (in template instantiation), they collapse:

```
T& &   → T&      // lvalue ref to lvalue ref → lvalue ref
T& &&  → T&      // rvalue ref to lvalue ref → lvalue ref
T&& &  → T&      // lvalue ref to rvalue ref → lvalue ref
T&& && → T&&     // rvalue ref to rvalue ref → rvalue ref
```

Simple rule: **if either reference is an lvalue reference, the result is an lvalue reference.**

```cpp
template<typename T>
void f(T&& x) {
    // If T = int&:  T&& = int& && = int&   (lvalue ref)
    // If T = int:   T&& = int&&            (rvalue ref)
    // If T = int&&: T&& = int&& && = int&& (rvalue ref)
}
```

---

## 4. Perfect Forwarding

### The Problem

Without perfect forwarding, a wrapper function changes the value category of its argument:

```cpp
void sink(std::string&& s) {
    std::cout << "received: " << s;
}

// BAD wrapper: always passes as lvalue!
template<typename T>
void bad_wrapper(T&& arg) {
    sink(arg);  // arg has a name -> it's an lvalue here -> error if sink takes rvalue!
}

std::string s = "hello";
bad_wrapper(std::move(s));  // We pass rvalue, but bad_wrapper passes it as lvalue
```

### `std::forward` Is The Solution

```cpp
// GOOD wrapper: preserves value category
template<typename T>
void perfect_wrapper(T&& arg) {
    sink(std::forward<T>(arg));  // forwards as rvalue if T=string, lvalue if T=string&
}

// What std::forward does (simplified):
template<typename T>
T&& forward(typename std::remove_reference<T>::type& arg) noexcept {
    return static_cast<T&&>(arg);
}
// When T = string:  returns string&&  (casts to rvalue)
// When T = string&: returns string&   (casts to lvalue, collapse: & && → &)
```

### Perfect Forwarding in Practice

```cpp
// Factory function: construct any type, forwarding all arguments perfectly
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// Emplace: construct directly in container without copies
std::vector<std::string> v;
v.emplace_back("hello");       // constructs string in-place, no temporary
v.push_back(std::string("hi")); // creates temporary string, then moves into vector

// Wrapper that perfectly forwards to any callable:
template<typename F, typename... Args>
auto invoke_with_log(F&& f, Args&&... args) {
    std::cout << "calling function\n";
    return std::forward<F>(f)(std::forward<Args>(args)...);
}
```

---

## 5. Function Templates

### Basic Template Syntax

```cpp
// Function template: parameterized by type T
template<typename T>
T max_val(T a, T b) {
    return a > b ? a : b;
}

// Called with explicit template argument:
int m1 = max_val<int>(3, 5);

// Called with template argument deduction (TAD):
int m2 = max_val(3, 5);       // T deduced as int
double m3 = max_val(3.0, 5.0); // T deduced as double
// max_val(3, 5.0)              // ERROR: T can't be both int and double

// Multiple template parameters:
template<typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {   // C++11 trailing return type
    return a + b;
}
// C++14: just use auto return type (return type deduction)
template<typename T, typename U>
auto add14(T a, U b) {
    return a + b;  // return type deduced from expression
}
```

### Non-Type Template Parameters

```cpp
// Template parameterized on a value, not a type
template<typename T, size_t N>
struct Array {
    T data[N];
    size_t size() const { return N; }
    T& operator[](size_t i) { return data[i]; }
};

Array<int, 5> arr;   // N=5 baked in at compile time: sizeof(arr) = 20 bytes
// No heap allocation, size known at compile time

// Useful for fixed-size buffers in embedded:
template<size_t CAPACITY>
class RingBuffer {
    uint8_t buf_[CAPACITY];
    size_t head_ = 0, tail_ = 0, count_ = 0;
public:
    bool push(uint8_t byte) {
        if (count_ == CAPACITY) return false;  // full
        buf_[tail_] = byte;
        tail_ = (tail_ + 1) % CAPACITY;
        ++count_;
        return true;
    }
    // ... pop, size, etc.
};

RingBuffer<256> uart_rx_buf;   // 256 bytes on stack/BSS, no heap
```

### Template Argument Deduction (TAD)

```cpp
// The compiler deduces T from the argument types
template<typename T>
void f(T x) {}     // T deduced from argument

template<typename T>
void g(T& x) {}    // T deduced, reference not part of T

template<typename T>
void h(const T& x) {}  // T deduced without const or ref

int x = 42;
f(x);         // T = int
g(x);         // T = int (g takes int&)
h(x);         // T = int (h takes const int&)
h(42);        // T = int (h takes const int&, binds rvalue to const lvalue ref)

// Decay in template deduction:
int arr[5];
f(arr);       // T = int* (arrays decay to pointer in by-value deduction)
g(arr);       // T = int[5] (by-reference: no decay! g takes int(&)[5])
```

---

## 6. Class Templates

### Full Class Template

```cpp
template<typename T>
class Stack {
    std::vector<T> data_;
public:
    void push(const T& val) { data_.push_back(val); }
    void push(T&& val)      { data_.push_back(std::move(val)); }

    // Single push with perfect forwarding:
    template<typename U>
    void push(U&& val) { data_.push_back(std::forward<U>(val)); }

    void pop()              { data_.pop_back(); }
    T&   top()              { return data_.back(); }
    const T& top()   const  { return data_.back(); }
    bool empty()     const  { return data_.empty(); }
    size_t size()    const  { return data_.size(); }
};

Stack<int> si;
Stack<std::string> ss;
```

### Class Template with Default Arguments

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class MyVector {
    // ...
};

MyVector<int>                    v1;  // uses default allocator
MyVector<int, CustomAllocator>   v2;  // uses custom allocator
```

### Class Template Argument Deduction (CTAD — C++17)

```cpp
// Before C++17: had to specify template arguments
std::pair<int, std::string> p1(42, "hello");
auto p2 = std::make_pair(42, "hello");  // workaround

// C++17 CTAD: compiler deduces from constructor arguments
std::pair p3(42, "hello");     // deduces pair<int, const char*>
std::vector v = {1, 2, 3};    // deduces vector<int>
std::lock_guard lk(mtx);       // deduces lock_guard<std::mutex>

// Custom deduction guides:
template<typename T>
class Box {
public:
    explicit Box(T val) : val_(val) {}
    T val_;
};
Box b(42);   // deduces Box<int> via implicit deduction guide
```

### Member Templates

```cpp
template<typename T>
class Container {
    T value_;
public:
    Container(T val) : value_(val) {}

    // Member function template: can convert between Container types
    template<typename U>
    Container(const Container<U>& other)
        : value_(static_cast<T>(other.value_)) {}
};

Container<double> cd(3.14);
Container<int>    ci(cd);    // converts: calls member template ctor with U=double
```

---

## 7. Template Specialization

### Full Specialization

```cpp
// Primary template
template<typename T>
struct TypeName {
    static const char* name() { return "unknown"; }
};

// Full specialization for int
template<>
struct TypeName<int> {
    static const char* name() { return "int"; }
};

// Full specialization for bool
template<>
struct TypeName<bool> {
    static const char* name() { return "bool"; }
};

TypeName<double>::name();  // "unknown"
TypeName<int>::name();     // "int"
TypeName<bool>::name();    // "bool"
```

### Partial Specialization

```cpp
// Primary template
template<typename T, typename U>
struct Pair {};

// Partial specialization: when both types are the same
template<typename T>
struct Pair<T, T> {
    static const char* desc() { return "same types"; }
};

// Partial specialization: when T is a pointer
template<typename T>
struct TypeName<T*> {
    static const char* name() { return "pointer"; }
};

TypeName<int*>::name();    // "pointer" (matches T*)
TypeName<int>::name();     // "int" (matches full specialization)
TypeName<double*>::name(); // "pointer" (matches T*)
```

### Function Template Specialization vs Overloading

Prefer **overloading** over specializing function templates — specializations don't participate in overload resolution:

```cpp
// Primary template
template<typename T>
void process(T val) { std::cout << "generic\n"; }

// Overload (preferred):
void process(int val) { std::cout << "int overload\n"; }  // found first in overload resolution

// Specialization (pitfall):
template<>
void process<int>(int val) { std::cout << "int spec\n"; } // might not be called when expected

// Herb Sutter's rule: "Don't specialize function templates"
// Use overloads or class template specializations instead
```

---

## 8. Variadic Templates

### Pack Expansion

```cpp
// Variadic template: accepts any number of type arguments
template<typename... Ts>
void print_all(Ts&&... args) {
    // sizeof...(args): number of arguments in the pack
    std::cout << "count: " << sizeof...(args) << "\n";

    // Pack expansion with fold expression (C++17):
    ((std::cout << args << " "), ...);   // expands: (cout<<a0 << " "), (cout<<a1<<" "), ...
    std::cout << "\n";
}

print_all(1, 2.5, "hello", true);  // count: 4 \n 1 2.5 hello 1
```

### Recursive Variadic Templates (Pre-C++17 Fold Expressions)

```cpp
// Base case: no arguments
void print_recursive() { std::cout << "\n"; }

// Recursive case: peel off first argument
template<typename First, typename... Rest>
void print_recursive(First&& first, Rest&&... rest) {
    std::cout << first << " ";
    print_recursive(std::forward<Rest>(rest)...);  // recurse with remaining
}

print_recursive(1, 2.5, "hello");  // 1 2.5 hello
```

### Fold Expressions (C++17)

```cpp
// Unary right fold: (pack op ...)
template<typename... Ts>
auto sum(Ts... vals) {
    return (... + vals);  // unary left fold: ((v0 + v1) + v2) + ...
}
sum(1, 2, 3, 4);  // 10

// Binary fold with initial value:
template<typename... Ts>
auto sum_with_init(Ts... vals) {
    return (0 + ... + vals);  // binary left fold: 0 + v0 + v1 + ...
}
sum_with_init();  // 0 (works with empty pack!)

// AND all boolean conditions:
template<typename... Ts>
bool all_positive(Ts... vals) {
    return (... && (vals > 0));  // true if all vals > 0
}

// OR: any positive
template<typename... Ts>
bool any_positive(Ts... vals) {
    return (... || (vals > 0));
}
```

### Parameter Pack in Structs

```cpp
// Tuple-like structure using variadic templates:
template<typename... Ts>
struct Tuple {};

template<typename Head, typename... Tail>
struct Tuple<Head, Tail...> : Tuple<Tail...> {
    Head value;
    Tuple(Head h, Tail... t) : Tuple<Tail...>(t...), value(h) {}
};

// Emplace pattern: forwarding all constructor args
template<typename T, typename... Args>
T* construct_in_buffer(void* buf, Args&&... args) {
    return new(buf) T(std::forward<Args>(args)...);
}
```

---

## 9. SFINAE & enable_if

### What SFINAE Is

**Substitution Failure Is Not An Error:** When the compiler tries to instantiate a template and fails during substitution of template arguments, it doesn't give an error — it simply removes that candidate from the overload set.

```cpp
// These two overloads compete:
template<typename T>
typename T::value_type  // fails for types without ::value_type member
get_value(T container) { return container.front(); }

template<typename T>
T get_value(T* ptr) { return *ptr; }

int arr[] = {1,2,3};
get_value(arr);         // first overload fails (int* has no ::value_type) -> SFINAE -> second chosen
std::vector<int> v{1};
get_value(v);           // second overload fails (vector is not a pointer) -> first chosen
```

### `std::enable_if`

```cpp
#include <type_traits>

// Enable a function only for integral types:
template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
double_it(T val) { return val * 2; }

// C++14 shorthand:
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
double_it_14(T val) { return val * 2; }

// Via default template parameter (cleaner, doesn't change return type):
template<typename T,
         typename = std::enable_if_t<std::is_integral_v<T>>>
T double_it_v2(T val) { return val * 2; }

double_it(5);       // OK: int is integral
double_it(3.14);    // ERROR: double is not integral -> substitution failure -> no candidate

// Conditional overloads:
template<typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>
void serialize(T val) { /* write as integer */ }

template<typename T, std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
void serialize(T val) { /* write as float */ }

template<typename T, std::enable_if_t<std::is_class_v<T>, int> = 0>
void serialize(T val) { /* write as struct */ }
```

---

## 10. if constexpr (C++17)

`if constexpr` evaluates the condition at compile time and discards the branch that isn't taken — the discarded branch doesn't even need to be syntactically valid for the given types:

```cpp
// Before C++17: needed template specialization or enable_if
// After C++17: clean and readable

template<typename T>
auto stringify(T val) {
    if constexpr (std::is_same_v<T, std::string>) {
        return val;              // val is already a string
    } else if constexpr (std::is_arithmetic_v<T>) {
        return std::to_string(val);   // numeric to string
    } else {
        return std::string("[unknown]");
    }
    // Each branch is only compiled for the T where the condition is true
    // So val.c_str() (valid for string) won't be compiled for T=int
}

// Extremely useful in embedded generic code:
template<typename T>
void write_register(T val) {
    if constexpr (sizeof(T) == 1) {
        write_byte(static_cast<uint8_t>(val));
    } else if constexpr (sizeof(T) == 2) {
        write_halfword(static_cast<uint16_t>(val));
    } else if constexpr (sizeof(T) == 4) {
        write_word(static_cast<uint32_t>(val));
    } else {
        static_assert(sizeof(T) <= 4, "register value too large");
    }
}
```

---

## 11. Concepts (C++20)

Concepts are compile-time predicates on template arguments — they make SFINAE readable and give better error messages.

### Defining Concepts

```cpp
#include <concepts>

// Concept: T must support operator+ and operator<
template<typename T>
concept Arithmetic = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
    { a - b } -> std::convertible_to<T>;
    { a * b } -> std::convertible_to<T>;
    { a < b } -> std::convertible_to<bool>;
};

// Concept: T must have .read() that returns something numeric
template<typename T>
concept Sensor = requires(T s) {
    { s.read() } -> std::convertible_to<float>;
    { s.is_valid() } -> std::same_as<bool>;
};

// Concept: T is a trivially copyable type (useful for DMA/memcpy safety)
template<typename T>
concept DmaTransferable = std::is_trivially_copyable_v<T>;
```

### Using Concepts

```cpp
// Method 1: requires clause after template parameter
template<typename T> requires Arithmetic<T>
T add(T a, T b) { return a + b; }

// Method 2: concept as type constraint (most concise)
template<Arithmetic T>
T multiply(T a, T b) { return a * b; }

// Method 3: abbreviated function template (C++20)
auto subtract(Arithmetic auto a, Arithmetic auto b) {
    return a - b;
}

// Method 4: requires clause inline
template<typename T>
T divide(T a, T b) requires Arithmetic<T> && (sizeof(T) >= 4) {
    return a / b;
}

// Standard library concepts (in <concepts>):
std::integral<T>           // integral types
std::floating_point<T>     // float, double, long double
std::signed_integral<T>    // signed integers
std::unsigned_integral<T>  // unsigned integers
std::same_as<T, U>         // T and U are the same type
std::derived_from<T, Base> // T is derived from Base
std::convertible_to<T, U>  // T is convertible to U
std::invocable<F, Args...> // F is callable with Args
std::regular<T>            // copyable, moveable, equality comparable
```

### Concept Constraints in Real Code

```cpp
// Sensor reading interface that works with any Sensor type:
template<Sensor S>
float read_filtered(S& sensor, int samples) {
    float sum = 0;
    for (int i = 0; i < samples; i++) {
        if (sensor.is_valid()) sum += sensor.read();
    }
    return sum / samples;
}

// DMA-safe serialization:
template<DmaTransferable T>
void dma_transfer(const T& src, volatile T* dst) {
    std::memcpy(const_cast<T*>(dst), &src, sizeof(T));
}

// Concept-constrained class template:
template<std::integral T>
class RegisterField {
    volatile T* addr_;
    T mask_;
public:
    T read()   const { return *addr_ & mask_; }
    void write(T val) { *addr_ = (*addr_ & ~mask_) | (val & mask_); }
};
```

---

## 12. Template Metaprogramming (TMP)

TMP uses templates to perform computation at compile time, producing values, types, or decisions.

### Compile-Time Values

```cpp
// Factorial at compile time:
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N-1>::value;
};
template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

static_assert(Factorial<5>::value == 120);

// C++14 and later: use constexpr functions instead (cleaner):
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
static_assert(factorial(5) == 120);
```

### Type Manipulation at Compile Time

```cpp
// Remove pointer from type:
template<typename T> struct RemovePointer      { using type = T; };
template<typename T> struct RemovePointer<T*>  { using type = T; };
// RemovePointer<int*>::type == int
// RemovePointer<int>::type  == int (no change)

// Conditional type selection:
template<bool Cond, typename IfTrue, typename IfFalse>
struct Conditional              { using type = IfTrue; };
template<typename IfTrue, typename IfFalse>
struct Conditional<false, IfTrue, IfFalse> { using type = IfFalse; };

// Choose uint16_t or uint32_t based on size needed:
template<int Bits>
using SmallestUint = typename Conditional<
    (Bits <= 16), uint16_t, uint32_t
>::type;

SmallestUint<8>  u8;   // uint16_t
SmallestUint<24> u24;  // uint32_t
```

### Type Lists

```cpp
// A list of types (fundamental TMP data structure):
template<typename... Ts>
struct TypeList {};

// Get Nth type from a list:
template<size_t N, typename... Ts>
struct TypeAt;

template<size_t N, typename Head, typename... Tail>
struct TypeAt<N, Head, Tail...> {
    using type = typename TypeAt<N-1, Tail...>::type;
};

template<typename Head, typename... Tail>
struct TypeAt<0, Head, Tail...> {
    using type = Head;
};

// TypeAt<2, int, float, double, char>::type == double

// Count types in list:
template<typename... Ts>
constexpr size_t type_count = sizeof...(Ts);
```

### constexpr if (TMP in C++17 and later)

```cpp
// Modern TMP prefers constexpr over recursive specialization:

// Compute power of 2 at compile time:
constexpr uint64_t pow2(int n) {
    return (n == 0) ? 1 : 2 * pow2(n - 1);
}
static_assert(pow2(10) == 1024);

// Select type based on platform:
template<int PtrSize = sizeof(void*)>
struct PtrSizedInt {
    using type = std::conditional_t<PtrSize == 8, uint64_t, uint32_t>;
};
using NativeWord = PtrSizedInt<>::type;  // uint64_t on 64-bit, uint32_t on 32-bit
```

---

## 13. Policy-Based Design

Policy-based design (Alexandrescu's idiom) uses template parameters to inject behavior at compile time — zero-overhead abstraction.

```cpp
// Policies as template parameters:

// Logger policies:
struct StdoutLogger {
    static void log(const char* msg) { std::puts(msg); }
};
struct NoLogger {
    static void log(const char* msg) {}  // noop: compiled away
};
struct SyslogLogger {
    static void log(const char* msg) { syslog(LOG_INFO, "%s", msg); }
};

// Error handling policies:
struct ThrowOnError {
    static void handle(const char* msg) { throw std::runtime_error(msg); }
};
struct AssertOnError {
    static void handle(const char* msg) { assert(false); }
};
struct IgnoreError {
    static void handle(const char* msg) {}
};

// Component parameterized by policies:
template<
    typename LogPolicy   = StdoutLogger,
    typename ErrorPolicy = ThrowOnError
>
class UartDriver {
public:
    bool init(uint32_t baud) {
        LogPolicy::log("UART init");
        if (!configure_hw(baud)) {
            ErrorPolicy::handle("UART init failed");
            return false;
        }
        return true;
    }

private:
    bool configure_hw(uint32_t baud) { /* ... */ return true; }
};

// Production: real logger, throw on error
UartDriver<SyslogLogger, ThrowOnError> prod_uart;

// Embedded: no logging overhead, assert on error
UartDriver<NoLogger, AssertOnError>    embedded_uart;

// Test: capture logs, ignore errors
UartDriver<StdoutLogger, IgnoreError>  test_uart;

// The compiler generates a completely different class for each combination.
// The policy code is fully inlined — zero runtime overhead.
```

### Combining Multiple Policies

```cpp
template<
    typename ValueType,
    typename Allocator   = std::allocator<ValueType>,
    typename SortPolicy  = std::less<ValueType>,
    typename CheckPolicy = NoRangeCheck
>
class SortedList {
    std::vector<ValueType, Allocator> data_;
    SortPolicy cmp_;
public:
    void insert(ValueType val) {
        auto pos = std::lower_bound(data_.begin(), data_.end(), val, cmp_);
        data_.insert(pos, std::move(val));
    }
    ValueType get(size_t i) {
        CheckPolicy::check(i, data_.size());  // compiled away for NoRangeCheck
        return data_[i];
    }
};
```

---

## 14. CRTP — Curiously Recurring Template Pattern

CRTP achieves static (compile-time) polymorphism — like virtual dispatch, but zero runtime cost.

### The Basic Pattern

```cpp
// Base class template: takes Derived as template parameter
template<typename Derived>
class SensorBase {
public:
    // Uses Derived's implementation via static_cast:
    float read() {
        return static_cast<Derived*>(this)->read_impl();
    }

    float read_average(int n) {
        float sum = 0;
        for (int i = 0; i < n; i++) sum += read();  // calls through CRTP
        return sum / n;
    }

    bool is_valid() {
        return static_cast<Derived*>(this)->is_valid_impl();
    }
};

// Concrete sensors:
class TemperatureSensor : public SensorBase<TemperatureSensor> {
    uint32_t* adc_reg_;
public:
    float read_impl()    { return *adc_reg_ * 0.1f - 40.0f; }  // ADC to Celsius
    bool  is_valid_impl(){ return (*adc_reg_ & 0x8000) == 0; }  // check error bit
};

class PressureSensor : public SensorBase<PressureSensor> {
public:
    float read_impl()    { return read_i2c_pressure(); }
    bool  is_valid_impl(){ return true; }
};

// Using them:
TemperatureSensor temp;
float avg_temp = temp.read_average(10);  // no virtual dispatch! fully inlined.

// Note: can't store mixed types in a container (they're different types)
// That's the trade-off: compile-time polymorphism vs runtime polymorphism
```

### CRTP for Mixin Functionality

```cpp
// Add comparison operators to any class:
template<typename Derived>
class Comparable {
public:
    bool operator!=(const Derived& rhs) const {
        return !(static_cast<const Derived&>(*this) == rhs);
    }
    bool operator<=(const Derived& rhs) const {
        return !(rhs < static_cast<const Derived&>(*this));
    }
    bool operator>(const Derived& rhs) const {
        return rhs < static_cast<const Derived&>(*this);
    }
    bool operator>=(const Derived& rhs) const {
        return !(static_cast<const Derived&>(*this) < rhs);
    }
};

// User just defines == and <, gets all 6 comparisons:
class Timestamp : public Comparable<Timestamp> {
    uint64_t ns_;
public:
    Timestamp(uint64_t ns) : ns_(ns) {}
    bool operator==(const Timestamp& rhs) const { return ns_ == rhs.ns_; }
    bool operator< (const Timestamp& rhs) const { return ns_ <  rhs.ns_; }
};

Timestamp t1(100), t2(200);
bool ge = t1 >= t2;   // works via CRTP Comparable
bool ne = t1 != t2;   // works via CRTP Comparable
```

### CRTP vs Virtual: When to Choose

| | CRTP (Static) | Virtual (Dynamic) |
|---|---|---|
| Performance | Zero overhead, inlined | vtable lookup, can't inline |
| Container of mixed types | ❌ (different types) | ✅ (polymorphic base) |
| Runtime type selection | ❌ | ✅ |
| Compile-time errors | ✅ clear | sometimes obscure |
| Code bloat | More instances | One vtable |
| Use case | Known types at compile time | Plugin systems, unknown types |

---

## 15. Type Traits & compile-time Introspection

### Standard Type Traits

```cpp
#include <type_traits>

// Query type properties:
std::is_integral_v<int>           // true
std::is_floating_point_v<double>  // true
std::is_pointer_v<int*>           // true
std::is_reference_v<int&>         // true
std::is_const_v<const int>        // true
std::is_same_v<int, int32_t>      // true on most platforms
std::is_trivially_copyable_v<int> // true
std::is_abstract_v<Shape>         // true (has pure virtual)

// Query relationships:
std::is_base_of_v<Base, Derived>  // true
std::is_convertible_v<int, double>// true
std::is_constructible_v<Foo, int> // true if Foo(int) exists

// Type transformations:
std::remove_reference_t<int&>     // int
std::remove_const_t<const int>    // int
std::remove_pointer_t<int*>       // int
std::decay_t<int[5]>              // int* (array decay)
std::decay_t<int&>                // int  (reference stripped)
std::add_const_t<int>             // const int
std::add_pointer_t<int>           // int*
std::underlying_type_t<MyEnum>    // the underlying integer type of enum

// Conditionals:
std::conditional_t<true, int, double>   // int
std::conditional_t<false, int, double>  // double
std::enable_if_t<condition, T>          // T if condition, else substitution failure
```

### Writing Custom Type Traits

```cpp
// Check if a type has a serialize() method:
template<typename T, typename = void>
struct has_serialize : std::false_type {};

template<typename T>
struct has_serialize<T, std::void_t<decltype(std::declval<T>().serialize())>>
    : std::true_type {};

// Usage:
class Foo { public: void serialize(); };
class Bar {};
static_assert(has_serialize<Foo>::value == true);
static_assert(has_serialize<Bar>::value == false);

// Shorthand:
template<typename T>
constexpr bool has_serialize_v = has_serialize<T>::value;

// Use in constexpr if:
template<typename T>
void save(const T& obj) {
    if constexpr (has_serialize_v<T>) {
        obj.serialize();
    } else {
        // fallback: raw memory dump
        std::fwrite(&obj, sizeof(T), 1, file);
    }
}
```

### `std::void_t` — The Detection Idiom

```cpp
// void_t maps any valid type expression to void
// If the expression is invalid, SFINAE kicks in

template<typename T, typename = void>
struct is_iterable : std::false_type {};

template<typename T>
struct is_iterable<T,
    std::void_t<
        decltype(std::begin(std::declval<T>())),  // has begin()
        decltype(std::end(std::declval<T>()))     // has end()
    >> : std::true_type {};

static_assert(is_iterable<std::vector<int>>::value == true);
static_assert(is_iterable<int>::value == false);
```

### `decltype` and `declval`

```cpp
// decltype: query the type of an expression without evaluating it
int x = 42;
decltype(x) y = x;           // y has type int
decltype(x + 3.14) z = 0.0;  // z has type double

// declval: create a value of type T in unevaluated context
// Useful in type traits where T might not be default-constructible:
struct NoDefault {
    NoDefault(int);
};

// This would fail: NoDefault nd; nd.serialize();
// But this works (unevaluated!):
using RetType = decltype(std::declval<NoDefault>().serialize());
```

---

## Key Interview Q&A Summary

**Q: What is the difference between `T&&` in a template parameter and in a concrete type?**
A: In a concrete context (`void f(int&&)`), `int&&` is an rvalue reference. In a template (`template<typename T> void f(T&&)`), `T&&` is a universal (forwarding) reference that can bind to both lvalues and rvalues, with T being deduced as either `Type&` or `Type` accordingly.

**Q: When would CRTP be preferred over virtual functions?**
A: When the set of types is known at compile time, performance is critical (no vtable overhead), and you don't need to store mixed types in a collection. Classic use cases: mixins, static interface contracts, expression templates.

**Q: What's the difference between `if constexpr` and regular `if`?**
A: `if constexpr` branches are evaluated at compile time — the rejected branch is discarded and not instantiated. A regular `if` compiles both branches regardless. `if constexpr` allows branches to use type-dependent operations that would fail to compile for other template instantiations.

**Q: What does `std::forward` actually do?**
A: It's a conditional cast. If `T` is deduced as `X&` (lvalue), `forward<T>(arg)` casts to `X&`. If `T` is deduced as `X` (rvalue), it casts to `X&&`. This preserves the value category through a function call — hence "perfect forwarding."

**Q: Why should move constructors be `noexcept`?**
A: `std::vector` and other standard containers use the move constructor during reallocation only if it's `noexcept`. If it can throw, they fall back to copy (to maintain the strong exception guarantee). Marking it `noexcept` unlocks performance — containers will choose move over copy.

**Q: What is SFINAE and when would you use Concepts instead?**
A: SFINAE is the mechanism (substitution failure is not an error) and `enable_if` is the traditional tool. It works but produces terrible error messages and hard-to-read code. C++20 Concepts are the clean replacement — same power, readable syntax, excellent error messages ("constraint not satisfied" instead of 20 lines of template substitution failures).

---

*Document 2 of 5 — Continue to Document 3: Standard Library, Smart Pointers & Concurrency*
