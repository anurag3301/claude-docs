# C++ Mastery Guide — Document 1 of 5
## Core Language Fundamentals: Types, Memory, Pointers, References & Classes

> **Scope:** Everything from fundamental types through classes, RAII, and the object model. This is the bedrock — every other C++ concept builds on this. Aimed at demonstrating senior-level C++ knowledge in a technical interview for an embedded/automotive role.

---

## Table of Contents

1. [Type System & Fundamental Types](#1-type-system--fundamental-types)
2. [Pointers, References & Value Categories](#2-pointers-references--value-categories)
3. [Memory Layout & Storage Duration](#3-memory-layout--storage-duration)
4. [The Object Model & Lifetime](#4-the-object-model--lifetime)
5. [Classes: Construction & Destruction](#5-classes-construction--destruction)
6. [The Rule of Zero, Three, Five](#6-the-rule-of-zero-three-five)
7. [RAII — Resource Acquisition Is Initialization](#7-raii--resource-acquisition-is-initialization)
8. [Operator Overloading](#8-operator-overloading)
9. [Inheritance & Polymorphism](#9-inheritance--polymorphism)
10. [const Correctness](#10-const-correctness)
11. [Type Qualifiers: volatile, mutable, restrict](#11-type-qualifiers-volatile-mutable-restrict)
12. [Casting in C++](#12-casting-in-c)
13. [Namespaces & Linkage](#13-namespaces--linkage)
14. [Enumerations](#14-enumerations)
15. [Unions & Bit Fields](#15-unions--bit-fields)

---

## 1. Type System & Fundamental Types

### The Type Hierarchy

C++ types are categorized as:
- **Fundamental types:** `bool`, `char`, `int`, `float`, `double`, `void`
- **Compound types:** pointers, references, arrays, functions, classes, unions, enums

### Exact-Width Integer Types (Critical for Embedded)

Always prefer `<cstdint>` types in embedded code:

```cpp
#include <cstdint>

int8_t   a;   // exactly 8 bits, signed
uint8_t  b;   // exactly 8 bits, unsigned
int16_t  c;   // exactly 16 bits, signed
uint16_t d;   // exactly 16 bits, unsigned
int32_t  e;   // exactly 32 bits, signed
uint32_t f;   // exactly 32 bits, unsigned
int64_t  g;   // exactly 64 bits, signed
uint64_t h;   // exactly 64 bits, unsigned

// Fastest types of at least N bits (compiler chooses the fastest native type)
int_fast32_t  fast;
uint_fast32_t ufast;

// Pointer-sized integers
intptr_t  ptr_val;   // signed integer that can hold a pointer
uintptr_t uptr_val;  // unsigned integer that can hold a pointer
ptrdiff_t diff;      // result of pointer subtraction
size_t    sz;        // result of sizeof, unsigned
ssize_t   ssz;       // POSIX: signed size (not in C++ standard!)
```

**Why this matters in interviews:** Using `int` for a register value is wrong — it's implementation-defined width. Using `uint32_t` is unambiguous and shows embedded discipline.

### Integer Promotion & Usual Arithmetic Conversions

```cpp
uint8_t  a = 200;
uint8_t  b = 100;
uint8_t  c = a + b;  // WRONG: a+b is int (300), then truncated to 44!

// What actually happens:
// 1. a and b are promoted to int (integer promotion rule)
// 2. addition happens in int: 300
// 3. result is converted back to uint8_t: 300 % 256 = 44

// Correct approach: be explicit
uint8_t c = static_cast<uint8_t>(a + b);
```

The **usual arithmetic conversions** (UAC) apply when two operands have different types:
1. If either is `long double`, convert the other to `long double`
2. Else if either is `double`, convert to `double`
3. Else if either is `float`, convert to `float`
4. Otherwise, apply integer promotions, then if types still differ, convert the "smaller" type to the "larger"

### Signed vs Unsigned: A Critical Pitfall

```cpp
int i = -1;
unsigned u = 1;
if (i < u)          // FALSE! i is converted to unsigned: becomes UINT_MAX
    std::cout << "i < u";  // never prints

// This is undefined behavior territory:
int x = INT_MAX;
x = x + 1;          // UB: signed overflow. Compiler assumes this never happens.
                     // May be optimized away, loop unrolled incorrectly, etc.

unsigned y = UINT_MAX;
y = y + 1;          // Well-defined: wraps to 0 (unsigned arithmetic is modular)
```

### `sizeof` and Alignment

```cpp
struct Padded {
    char  a;    // 1 byte
    // 3 bytes padding (on 32-bit aligned systems)
    int   b;    // 4 bytes
    char  c;    // 1 byte
    // 3 bytes padding
};              // sizeof = 12, not 6!

struct Packed {
    char  a;    // 1 byte
    char  c;    // 1 byte — reordered
    // 2 bytes padding
    int   b;    // 4 bytes
};              // sizeof = 8

// Forcing specific alignment (C++11):
struct alignas(16) SIMDVec {
    float data[4];
};

// Checking alignment:
static_assert(alignof(SIMDVec) == 16, "alignment not met");
static_assert(sizeof(SIMDVec) == 16, "size not as expected");
```

**Interview insight:** Struct layout is critical in embedded when you're mapping structs to hardware registers or DMA buffers. Always be aware of padding and use `__attribute__((packed))` or `#pragma pack` with care (packed structs can cause alignment faults on ARM).

---

## 2. Pointers, References & Value Categories

### Pointers: The Full Picture

```cpp
int  x = 42;
int* p = &x;         // pointer to int
*p = 100;            // dereference: x is now 100

int** pp = &p;       // pointer to pointer to int

const int* cp = &x;  // pointer to const int: can't modify *cp
int* const pc = &x;  // const pointer to int: can't change where pc points
const int* const cpc = &x;  // const pointer to const int: neither

// Pointer arithmetic
int arr[5] = {1,2,3,4,5};
int* ap = arr;       // arr decays to pointer to first element
ap++;                // points to arr[1]
*(ap + 2) = 99;      // same as arr[3] = 99

// Pointer comparison
int a, b;
int* pa = &a;
int* pb = &b;
bool same = (pa == pb);  // false (different objects)
// pa < pb is UNDEFINED for pointers to different objects (only defined within array)
```

### Null Pointer: Always Use `nullptr`

```cpp
int* p1 = nullptr;   // C++11: type-safe null pointer constant
int* p2 = NULL;      // C: macro, expands to 0 — ambiguous in overloads
int* p3 = 0;         // also works but confusing

// Why nullptr matters:
void f(int);
void f(int*);
f(NULL);    // ambiguous or calls f(int)!
f(nullptr); // unambiguously calls f(int*)
```

### References: Aliases, Not Pointers

```cpp
int x = 42;
int& ref = x;        // ref is an alias for x
ref = 100;           // x is now 100 — ref IS x

// Rules:
// 1. References must be initialized at declaration
// 2. References cannot be rebound (always refer to the same object)
// 3. No null references (a reference is always valid — if it exists)
// 4. No pointer arithmetic on references
// 5. sizeof(ref) == sizeof(the_referred_object)

// Reference to const: can bind to temporaries
const int& cr = 42;    // lifetime of temporary extended to lifetime of cr
// int& r = 42;        // ERROR: non-const reference to temporary

// Passing large objects — always pass by const reference when not modifying:
void process(const std::vector<uint8_t>& data);  // no copy
```

### Value Categories: lvalue, rvalue, xvalue (C++11)

This is one of the most important and misunderstood C++ concepts:

```cpp
// lvalue: has identity (a name, a memory address), can take its address
int x = 42;
x;          // lvalue: x has a name
&x;         // taking address of lvalue: OK

// rvalue (prvalue): no identity, temporary, about to be destroyed
42;          // rvalue: literal
x + 1;       // rvalue: result of expression
std::string("hello");  // rvalue: temporary object

// xvalue (expiring value): has identity but is moveable
std::move(x);  // xvalue: x has identity but we've said "you can move from me"

// Why this matters:
int& lref = x;           // lvalue reference: binds to lvalue
// int& lref = 42;       // ERROR: can't bind non-const lref to rvalue
const int& clref = 42;   // OK: const lvalue ref can bind to rvalue (extends lifetime)
int&& rref = 42;         // rvalue reference: binds to rvalue (C++11)
int&& rref2 = std::move(x);  // binds to xvalue
```

### Pointer vs Reference: When to Use Which

| Situation | Use |
|---|---|
| Must always refer to an object | Reference |
| May be null (optional parameter) | Pointer |
| May be rebound to different objects | Pointer |
| Return value that may be "nothing" | `std::optional<T>` (C++17) or pointer |
| Function parameter, never null | Reference |
| Observing ownership | Raw pointer or `const T*` |
| Owning a heap object | `std::unique_ptr<T>` |

---

## 3. Memory Layout & Storage Duration

### The Four Storage Durations

```cpp
// 1. STATIC storage duration: exists for entire program lifetime
static int global_counter = 0;     // file-scope: static storage
static int fn_counter = 0;         // function-local static: initialized once, persists

// 2. THREAD storage duration (C++11): one instance per thread
thread_local int tls_val = 0;      // each thread has its own copy

// 3. AUTOMATIC storage duration: scope-bound (stack)
void foo() {
    int local = 42;                // on the stack, destroyed when foo() returns
}                                  // local destroyed here

// 4. DYNAMIC storage duration: heap, manual or smart-pointer managed
int* p = new int(42);              // heap allocation
delete p;                          // must free manually
// Better:
auto sp = std::make_unique<int>(42); // C++14: auto-freed
```

### Stack vs Heap: Deep Understanding

```cpp
// Stack:
// - Fast allocation (just decrement stack pointer)
// - Automatically freed (increment stack pointer on return)
// - Limited size (typically 1-8 MB, configurable)
// - Contiguous, cache-friendly
// - No fragmentation

// Heap:
// - Arbitrary size (limited by virtual memory)
// - Manual management (or smart pointers)
// - Slower (allocator must find a free block)
// - Can fragment over time
// - Dynamic size known at runtime

// In embedded: avoid heap allocation after initialization
// Reason: fragmentation can cause allocation failure in long-running systems

// Stack overflow example:
void recursive(int n) {
    char big[65536];  // 64KB on stack each call
    recursive(n + 1); // will overflow stack
}
```

### Memory Segments in a Binary

```
High address
+-------------------+
|      Stack        |  <- grows downward, automatic storage
+-------------------+
|        |          |
|        v          |
|                   |
|        ^          |
|        |          |
+-------------------+
|      Heap         |  <- grows upward, dynamic storage
+-------------------+
|   BSS segment     |  <- zero-initialized globals/statics (not in binary file)
+-------------------+
|   Data segment    |  <- initialized globals/statics (in binary file)
+-------------------+
|   Text segment    |  <- executable code (read-only)
Low address
```

```cpp
int global_init = 42;           // .data segment
int global_zero;                 // .bss segment (zero-initialized)
const int global_const = 10;    // .rodata (read-only data, usually merged with .text)

void foo() {
    static int s = 99;          // .data segment (despite being inside a function)
    int local = 5;              // stack
    int* heap_p = new int(7);   // heap (pointer itself is on stack)
}
```

---

## 4. The Object Model & Lifetime

### Object Lifetime Rules

An object's lifetime begins when its initialization is complete and ends when its storage is released or reused.

```cpp
struct Sensor {
    Sensor()  { std::cout << "born\n"; }
    ~Sensor() { std::cout << "died\n"; }
};

void demo() {
    Sensor s1;              // lifetime starts here
    {
        Sensor s2;          // lifetime starts
    }                       // s2 dies here (reverse order of construction)
    // s1 still alive
}                           // s1 dies here

// Destruction order: reverse of construction order (for objects at same scope)
// This is guaranteed by the standard — critical for RAII correctness
```

### Placement New: Constructing in Pre-Allocated Memory

```cpp
// Used in embedded for memory pools, custom allocators
alignas(Sensor) char buf[sizeof(Sensor)];   // raw storage

Sensor* s = new(buf) Sensor();              // placement new: construct in buf
s->~Sensor();                               // must explicitly call destructor!
// do NOT: delete s; (would try to free buf, which isn't heap memory)

// Common use: fixed-size object pool in embedded systems
template<typename T, size_t N>
class ObjectPool {
    alignas(T) char storage_[sizeof(T) * N];
    bool used_[N] = {};
public:
    T* allocate() {
        for (size_t i = 0; i < N; i++) {
            if (!used_[i]) {
                used_[i] = true;
                return new(storage_ + i * sizeof(T)) T();
            }
        }
        return nullptr; // pool exhausted
    }
    void deallocate(T* p) {
        p->~T();
        size_t i = (reinterpret_cast<char*>(p) - storage_) / sizeof(T);
        used_[i] = false;
    }
};
```

### Trivial Types, Standard Layout & POD

```cpp
// Trivially copyable: can be memcpy'd safely
struct TrivialPoint { int x; int y; };   // trivially copyable

// Standard layout: compatible with C struct layout
struct StdLayout {
    int a;
    double b;
    // no virtual functions, no mixed access specifiers, no base classes
};

// POD (Plain Old Data) = trivial + standard layout
// C++20 deprecates the term POD, but the concept matters

// Check at compile time:
static_assert(std::is_trivially_copyable_v<TrivialPoint>);
static_assert(std::is_standard_layout_v<StdLayout>);

// Why this matters in embedded:
// - DMA transfers expect C-compatible layout
// - memcpy between firmware buffers requires trivial copyability
// - Hardware register structs must be standard layout
```

---

## 5. Classes: Construction & Destruction

### Member Initializer Lists

```cpp
class Register {
    uint32_t address_;
    uint32_t value_;
    const std::string name_;  // const member: MUST be in initializer list

public:
    // Correct: member initializer list (runs before body)
    Register(uint32_t addr, std::string name)
        : address_(addr)       // direct initialization
        , value_(0)            // explicitly zero
        , name_(std::move(name))  // move, not copy
    {
        // Body: address_, value_, name_ are already constructed here
    }

    // WRONG way (assignment, not initialization):
    Register(uint32_t addr) {
        address_ = addr;  // This is ASSIGNMENT, not initialization!
        // For const or reference members, this won't even compile
    }
};
```

**Initialization order:** Members are always initialized in the order they are declared in the class, regardless of the order in the initializer list. If your initializer list order differs from declaration order, compilers warn — heed those warnings.

### Delegating Constructors (C++11)

```cpp
class Uart {
    int baud_;
    int bits_;
    char parity_;

public:
    Uart(int baud, int bits, char parity)
        : baud_(baud), bits_(bits), parity_(parity) {}

    // Delegate to the main constructor
    Uart() : Uart(115200, 8, 'N') {}
    Uart(int baud) : Uart(baud, 8, 'N') {}
};
```

### Default Member Initialization (C++11)

```cpp
class Config {
    uint32_t timeout_ms_ = 1000;     // default value
    bool     enabled_    = true;     // default value
    std::string name_    = "default"; // default value

public:
    Config() = default;  // uses all the defaults above
    Config(uint32_t t) : timeout_ms_(t) {}  // other members use defaults
};
```

### Constructor Explicit Keyword

```cpp
class Buffer {
    size_t size_;
public:
    explicit Buffer(size_t sz) : size_(sz) {}
};

// Without explicit:
// Buffer b = 42;  // would compile — implicit conversion from int to Buffer
// With explicit:
Buffer b1(42);          // OK: direct initialization
Buffer b2 = Buffer(42); // OK: explicit construction then copy
// Buffer b3 = 42;      // ERROR: no implicit conversion (good!)

// Always mark single-argument constructors as explicit
// unless you specifically want implicit conversion (rare)
```

### Destructor Details

```cpp
class FileHandle {
    int fd_;
public:
    explicit FileHandle(const char* path) : fd_(open(path, O_RDONLY)) {
        if (fd_ < 0) throw std::runtime_error("open failed");
    }

    ~FileHandle() {
        if (fd_ >= 0) close(fd_);  // always cleanup
        // Destructors should NEVER throw
        // If close() fails, log it but don't throw — exception in destructor
        // during stack unwinding = std::terminate()
    }
};

// Virtual destructors: REQUIRED if base class is used polymorphically
class Base {
public:
    virtual ~Base() = default;  // virtual! Without this, deleting via Base* is UB
};

class Derived : public Base {
    // members that need cleanup
};

Base* b = new Derived();
delete b;  // Without virtual destructor: only ~Base() called, ~Derived() skipped -> leak
           // With virtual destructor: ~Derived() then ~Base() called -> correct
```

---

## 6. The Rule of Zero, Three, Five

### Why These Rules Exist

When you manage a resource (heap memory, file handle, socket, mutex), the compiler-generated special member functions (copy constructor, copy assignment, destructor) won't know how to correctly handle that resource. You must either define them or prevent their use.

### Rule of Three (Pre-C++11)

If you define any of these, define all three:
1. Destructor
2. Copy constructor
3. Copy assignment operator

```cpp
class DmaBuffer {
    uint8_t* data_;
    size_t   size_;

public:
    explicit DmaBuffer(size_t size)
        : data_(new uint8_t[size]), size_(size) {}

    // 1. Destructor
    ~DmaBuffer() { delete[] data_; }

    // 2. Copy constructor (deep copy)
    DmaBuffer(const DmaBuffer& other)
        : data_(new uint8_t[other.size_]), size_(other.size_) {
        std::memcpy(data_, other.data_, size_);
    }

    // 3. Copy assignment operator (deep copy)
    DmaBuffer& operator=(const DmaBuffer& other) {
        if (this != &other) {           // self-assignment check
            delete[] data_;             // release old resource
            size_ = other.size_;
            data_ = new uint8_t[size_]; // acquire new
            std::memcpy(data_, other.data_, size_);
        }
        return *this;
    }
};
```

### Rule of Five (C++11)

Add move constructor and move assignment to the Rule of Three:

```cpp
class DmaBuffer {
    uint8_t* data_;
    size_t   size_;

public:
    explicit DmaBuffer(size_t size)
        : data_(new uint8_t[size]), size_(size) {}

    ~DmaBuffer() { delete[] data_; }

    // Copy constructor (expensive: allocates and copies)
    DmaBuffer(const DmaBuffer& other)
        : data_(new uint8_t[other.size_]), size_(other.size_) {
        std::memcpy(data_, other.data_, size_);
    }

    // Copy assignment (expensive)
    DmaBuffer& operator=(const DmaBuffer& other) {
        if (this != &other) {
            DmaBuffer tmp(other);           // copy-and-swap idiom
            std::swap(data_, tmp.data_);
            std::swap(size_, tmp.size_);
        }
        return *this;
    }

    // Move constructor (cheap: steal the pointer)
    DmaBuffer(DmaBuffer&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;   // leave source in valid-but-empty state
        other.size_ = 0;
    }

    // Move assignment (cheap)
    DmaBuffer& operator=(DmaBuffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;          // free our current resource
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;   // leave source empty
            other.size_ = 0;
        }
        return *this;
    }
};
```

### Rule of Zero (Preferred)

Design your class so it needs **none** of the special member functions — use members that manage their own resources:

```cpp
class DmaBuffer {
    std::unique_ptr<uint8_t[]> data_;
    size_t size_;

public:
    explicit DmaBuffer(size_t size)
        : data_(std::make_unique<uint8_t[]>(size)), size_(size) {}

    // No destructor needed: unique_ptr cleans up automatically
    // No copy: unique_ptr is non-copyable -> DmaBuffer is non-copyable (good!)
    // Move works automatically: unique_ptr is moveable -> DmaBuffer is moveable

    // That's it. Rule of Zero achieved.
};
```

**The hierarchy of preference:** Rule of Zero > Rule of Five > Rule of Three.

### `= default` and `= delete`

```cpp
class NonCopyable {
public:
    NonCopyable() = default;                              // compiler-generated default ctor
    ~NonCopyable() = default;                             // compiler-generated dtor
    NonCopyable(const NonCopyable&) = delete;             // explicitly deleted
    NonCopyable& operator=(const NonCopyable&) = delete;  // explicitly deleted
    NonCopyable(NonCopyable&&) = default;                 // compiler-generated move
    NonCopyable& operator=(NonCopyable&&) = default;      // compiler-generated move assign
};

// This is the pattern for anything that owns a unique resource
// (mutex, file handle, hardware device handle)
```

---

## 7. RAII — Resource Acquisition Is Initialization

RAII is the single most important C++ idiom. Every resource must be owned by an object whose destructor releases it. There must be no "naked" resource acquisition without a corresponding owner.

### The Pattern

```cpp
// BAD: naked resource management
void bad_example(const char* path) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) return;

    char* buf = new char[4096];
    ssize_t n = read(fd, buf, 4096);
    if (n < 0) {
        delete[] buf;    // must remember to free
        close(fd);       // must remember to close
        return;          // easy to miss one of these
    }
    process(buf, n);
    delete[] buf;
    close(fd);
}

// GOOD: RAII ownership
class FileDescriptor {
    int fd_;
public:
    explicit FileDescriptor(const char* path, int flags)
        : fd_(open(path, flags)) {
        if (fd_ < 0) throw std::system_error(errno, std::generic_category());
    }
    ~FileDescriptor() noexcept { if (fd_ >= 0) close(fd_); }
    int get() const noexcept { return fd_; }

    FileDescriptor(const FileDescriptor&) = delete;
    FileDescriptor& operator=(const FileDescriptor&) = delete;
    FileDescriptor(FileDescriptor&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }
};

void good_example(const char* path) {
    FileDescriptor fd(path, O_RDONLY);        // acquired
    auto buf = std::make_unique<char[]>(4096); // acquired

    ssize_t n = read(fd.get(), buf.get(), 4096);
    if (n < 0) return;   // fd and buf both cleaned up automatically

    process(buf.get(), n);
}   // fd and buf both cleaned up here too
```

### RAII for Mutexes

```cpp
#include <mutex>

std::mutex mtx;

// BAD:
void bad_lock() {
    mtx.lock();
    // ... if exception thrown here, mutex never unlocked -> deadlock
    mtx.unlock();
}

// GOOD:
void good_lock() {
    std::lock_guard<std::mutex> guard(mtx);  // locks in constructor
    // ... even if exception thrown, guard's destructor unlocks
}   // guard destroyed here, mutex unlocked

// C++17: Class Template Argument Deduction (CTAD)
void modern_lock() {
    std::lock_guard guard(mtx);  // no need to specify <std::mutex>
}

// scoped_lock (C++17): locks multiple mutexes deadlock-free
std::mutex m1, m2;
void two_locks() {
    std::scoped_lock lock(m1, m2);  // acquires both, deadlock-free algorithm
}
```

### `noexcept` in RAII

Destructors and move operations **must** be `noexcept`:

```cpp
class Resource {
public:
    // Move constructor: noexcept allows containers to move instead of copy
    Resource(Resource&&) noexcept;

    // Destructor: noexcept is implicit, but be explicit
    ~Resource() noexcept;

    // If move ctor can throw, std::vector won't use it (uses copy instead)
    // This kills performance — always make moves noexcept
};

// Check:
static_assert(std::is_nothrow_move_constructible_v<Resource>);
```

---

## 8. Operator Overloading

### Which Operators to Overload (and How)

```cpp
class Vec2 {
public:
    float x, y;
    Vec2(float x, float y) : x(x), y(y) {}

    // Arithmetic: return by value, take by const ref
    Vec2 operator+(const Vec2& rhs) const { return {x + rhs.x, y + rhs.y}; }
    Vec2 operator-(const Vec2& rhs) const { return {x - rhs.x, y - rhs.y}; }
    Vec2 operator*(float s)         const { return {x * s, y * s}; }

    // Compound assignment: return *this by reference
    Vec2& operator+=(const Vec2& rhs) { x += rhs.x; y += rhs.y; return *this; }
    Vec2& operator-=(const Vec2& rhs) { x -= rhs.x; y -= rhs.y; return *this; }
    Vec2& operator*=(float s)         { x *= s; y *= s; return *this; }

    // Unary minus
    Vec2 operator-() const { return {-x, -y}; }

    // Comparison (C++20: spaceship operator)
    bool operator==(const Vec2& rhs) const { return x == rhs.x && y == rhs.y; }

    // Subscript
    float& operator[](int i)       { return i == 0 ? x : y; }
    float  operator[](int i) const { return i == 0 ? x : y; }  // const version

    // Stream output (non-member, friend)
    friend std::ostream& operator<<(std::ostream& os, const Vec2& v) {
        return os << "(" << v.x << ", " << v.y << ")";
    }
};

// Scalar * Vec2 (can't be member since left operand is float, not Vec2)
Vec2 operator*(float s, const Vec2& v) { return v * s; }
```

### Conversion Operators

```cpp
class Voltage {
    float millivolts_;
public:
    explicit Voltage(float mv) : millivolts_(mv) {}

    // Implicit conversion to float (use with care!)
    operator float() const { return millivolts_ / 1000.0f; }

    // Explicit conversion (preferred — prevents accidental conversions)
    explicit operator int() const { return static_cast<int>(millivolts_); }

    // Conversion to bool (used in if(voltage) checks)
    explicit operator bool() const { return millivolts_ > 0.0f; }
};

Voltage v(3300.0f);  // 3.3V in millivolts
float volts = v;     // implicit: 3.3
// int mv = v;       // ERROR: explicit conversion requires cast
int mv = static_cast<int>(v);  // OK: 3300
if (v) { /* voltage is positive */ }  // explicit bool conversion works in conditions
```

### The Spaceship Operator `<=>` (C++20)

```cpp
#include <compare>

struct Timestamp {
    uint32_t seconds;
    uint32_t nanoseconds;

    // Single operator defines all six comparisons
    auto operator<=>(const Timestamp& rhs) const = default;
    // Default: lexicographic comparison of members in declaration order

    // Custom:
    std::strong_ordering operator<=>(const Timestamp& rhs) const {
        if (auto cmp = seconds <=> rhs.seconds; cmp != 0) return cmp;
        return nanoseconds <=> rhs.nanoseconds;
    }
    bool operator==(const Timestamp&) const = default;
};

// Now all of these work:
Timestamp t1{100, 0}, t2{100, 500};
bool lt = t1 < t2;    // true
bool gt = t1 > t2;    // false
bool le = t1 <= t2;   // true
// etc.
```

---

## 9. Inheritance & Polymorphism

### Single Inheritance & The vtable

```cpp
class Shape {
public:
    virtual double area() const = 0;    // pure virtual: makes Shape abstract
    virtual double perimeter() const = 0;
    virtual void draw() const { /* default implementation */ }
    virtual ~Shape() = default;          // MUST be virtual
    std::string name;
};

class Circle : public Shape {
    double radius_;
public:
    explicit Circle(double r) : radius_(r) {}
    double area()      const override { return 3.14159 * radius_ * radius_; }
    double perimeter() const override { return 2 * 3.14159 * radius_; }
    // draw() not overridden: uses Shape's default
};

// Polymorphism in action
void print_area(const Shape& s) {
    std::cout << s.area();  // dispatched via vtable at runtime
}

Circle c(5.0);
print_area(c);  // calls Circle::area()
```

### How the vtable Works Internally

```cpp
// What the compiler generates (conceptually):
struct Shape_vtable {
    double (*area)(const Shape*);
    double (*perimeter)(const Shape*);
    void (*draw)(const Shape*);
    void (*destructor)(Shape*);
};

struct Shape {
    Shape_vtable* vptr;   // hidden pointer, added by compiler (typically first member)
    std::string name;
};
// sizeof(Shape) includes the vptr!
```

**Interview insight:** Every class with virtual functions has a vtable. Every instance has a hidden vptr (pointer to the vtable). This costs:
- 1 pointer per object (8 bytes on 64-bit)
- 1 indirect call per virtual dispatch (potential cache miss)
- Can't be inlined (compiler doesn't know the concrete type)

### Multiple Inheritance & Virtual Inheritance

```cpp
class Flyable  { public: virtual void fly()  = 0; };
class Swimmable{ public: virtual void swim() = 0; };

class Duck : public Flyable, public Swimmable {
public:
    void fly()  override { /* quack and fly */ }
    void swim() override { /* paddle */ }
};

// Diamond problem:
class A { public: int x; };
class B : public A {};
class C : public A {};
class D : public B, public C {};  // D has TWO copies of A::x!

D d;
// d.x;         // ambiguous: which A::x?
d.B::x = 1;    // explicit disambiguation
d.C::x = 2;

// Solution: virtual inheritance
class B2 : virtual public A {};
class C2 : virtual public A {};
class D2 : public B2, public C2 {};  // only ONE copy of A
D2 d2;
d2.x = 42;  // unambiguous now
```

### `override` and `final` (C++11)

```cpp
class Base {
    virtual void foo(int x);
    virtual void bar() const;
};

class Derived : public Base {
    void foo(int x) override;        // OK: overrides Base::foo(int)
    // void foo(double x) override;  // ERROR: no matching base function
    // void bar() override;          // ERROR: missing const
    void bar() const override final; // final: no further overriding
};

class MoreDerived : public Derived {
    // void bar() const override;    // ERROR: bar is final
};

// Final class: cannot be inherited from
class Singleton final {
    // ...
};
```

### Abstract Interfaces and Pure Virtual

```cpp
// Interface pattern (no data members, all pure virtual)
class IUartDriver {
public:
    virtual ~IUartDriver() = default;
    virtual bool     init(uint32_t baud_rate)                   = 0;
    virtual ssize_t  write(const uint8_t* data, size_t len)     = 0;
    virtual ssize_t  read(uint8_t* buf, size_t len, int timeout) = 0;
    virtual void     flush()                                     = 0;
};

// Concrete implementations
class LinuxUartDriver : public IUartDriver { /* ... */ };
class SimulatedUartDriver : public IUartDriver { /* for testing */ };

// Code depends on the interface, not the implementation
class Protocol {
    IUartDriver& uart_;
public:
    explicit Protocol(IUartDriver& uart) : uart_(uart) {}
    void send_command(uint8_t cmd) {
        uart_.write(&cmd, 1);  // calls through vtable
    }
};
```

---

## 10. const Correctness

### const on Variables, Pointers, Members

```cpp
const int MAX = 100;         // constant variable
int* const cp = &x;          // const pointer (can't redirect)
const int* pc = &x;          // pointer to const (can't modify value)
const int* const cpc = &x;   // both

// Reading pointer declarations: right-to-left
// "int* const": const pointer to int
// "const int*": pointer to const int
// Mnemonic: "const applies to what's immediately to its left"
//            (if nothing to left, applies to what's to right)
```

### const Member Functions

```cpp
class Sensor {
    float value_;
    mutable int read_count_;  // mutable: can be modified in const functions

public:
    // const member function: doesn't modify observable state
    float read() const {
        ++read_count_;        // OK: read_count_ is mutable
        return value_;
    }

    // Non-const: modifies state
    void calibrate(float offset) {
        value_ -= offset;
    }

    // const overloads: provide both versions
    float&       get()       { return value_; }  // non-const: writable access
    const float& get() const { return value_; }  // const: read-only access
};

const Sensor s;
// s.calibrate(1.0f);  // ERROR: cannot call non-const member on const object
float v = s.read();    // OK: read() is const
```

### Propagating const Through Containers

```cpp
class RegisterMap {
    std::vector<uint32_t> regs_;
public:
    // Returns writable reference for non-const RegisterMap
    uint32_t& operator[](size_t i) { return regs_[i]; }

    // Returns read-only reference for const RegisterMap
    const uint32_t& operator[](size_t i) const { return regs_[i]; }
};

RegisterMap rm;
rm[0] = 42;              // calls non-const operator[]

const RegisterMap crm = rm;
// crm[0] = 42;          // ERROR: calls const operator[], returns const ref
uint32_t v = crm[0];    // OK: reading is fine
```

---

## 11. Type Qualifiers: volatile, mutable, restrict

### `volatile` in C++ (Embedded Critical)

```cpp
// Hardware register mapped to memory
volatile uint32_t* const UART_STATUS = reinterpret_cast<volatile uint32_t*>(0x40013800);

// WITHOUT volatile: compiler may cache register value in CPU register:
// if (UART_STATUS & TX_READY) { ... }  // compiler caches the read
// while (!(UART_STATUS & TX_READY)) {} // might loop forever using cached value

// WITH volatile: every access is a real memory read
while (!(*UART_STATUS & TX_READY)) {}  // fresh read every iteration

// volatile IS NOT a synchronization primitive!
// It prevents compiler optimization but does NOT prevent CPU reordering
// For multi-threaded synchronization, use std::atomic
volatile bool flag = false;  // WRONG for thread synchronization

std::atomic<bool> aflag = false;  // CORRECT: atomic + memory ordering
```

### `mutable`

```cpp
class Cache {
    mutable std::unordered_map<int, int> cache_;  // mutable: modifiable in const
    mutable std::mutex cache_mutex_;              // mutable: lockable in const

public:
    // Logically const: caller sees the same value, even though we update cache
    int expensive_compute(int key) const {
        std::lock_guard lock(cache_mutex_);
        auto it = cache_.find(key);
        if (it != cache_.end()) return it->second;

        int result = /* heavy computation */ 42;
        cache_[key] = result;   // OK: cache_ is mutable
        return result;
    }
};
```

---

## 12. Casting in C++

**Never use C-style casts `(Type)value` in C++ code.** They're too powerful and hide errors.

### `static_cast` — Compile-Time Checked Conversions

```cpp
// Numeric conversions
double d = 3.14;
int i = static_cast<int>(d);          // 3: explicit truncation

// Pointer up/down cast (no runtime check)
Derived* dp = static_cast<Derived*>(base_ptr);  // dangerous if not Derived

// Enum to int and back
uint8_t val = static_cast<uint8_t>(MyEnum::VALUE);
MyEnum e = static_cast<MyEnum>(42);

// void* to typed pointer (common in C interop)
void* raw = malloc(sizeof(int));
int* ip = static_cast<int*>(raw);
```

### `dynamic_cast` — Runtime-Checked Polymorphic Cast

```cpp
Base* b = get_shape();  // might be Circle*, Square*, etc.

// Pointer: returns nullptr on failure
Circle* c = dynamic_cast<Circle*>(b);
if (c) { /* b is actually a Circle */ }

// Reference: throws std::bad_cast on failure
try {
    Circle& cr = dynamic_cast<Circle&>(*b);
} catch (const std::bad_cast&) {
    // b is not a Circle
}

// Requires at least one virtual function (RTTI must be enabled)
// Avoid in tight loops: it has runtime cost
```

### `const_cast` — Remove const (Use Rarely)

```cpp
// Only valid use: calling a non-const function on a const object
// ONLY safe if the original object was non-const
const int* cp = get_const_ptr();
int* p = const_cast<int*>(cp);  // remove const
// Writing through p is UB if the original object was actually const!

// Legitimate use in C interop:
void legacy_c_api(char* s);        // old C API that should take const char*
const char* msg = "hello";
legacy_c_api(const_cast<char*>(msg));  // safe: legacy_c_api won't write
```

### `reinterpret_cast` — Bit-Level Reinterpretation

```cpp
// Access hardware registers
volatile uint32_t* GPIOA = reinterpret_cast<volatile uint32_t*>(0x40020000);

// Convert between pointer types (undefined behavior if misused)
uint8_t bytes[4] = {0x01, 0x02, 0x03, 0x04};
uint32_t* word = reinterpret_cast<uint32_t*>(bytes);  // careful: alignment!

// Type punning (inspect float as bits) — use memcpy for correctness
float f = 3.14f;
uint32_t bits;
std::memcpy(&bits, &f, sizeof(bits));  // correct type punning
// uint32_t bits = *reinterpret_cast<uint32_t*>(&f);  // UB: strict aliasing violation
```

---

## 13. Namespaces & Linkage

### Namespaces

```cpp
namespace bmw::platform {   // C++17 nested namespace shorthand

    class EcuManager { /* ... */ };

    namespace detail {
        void internal_helper();  // implementation detail, hidden from users
    }

}  // namespace bmw::platform

// Usage:
bmw::platform::EcuManager mgr;

// Using declaration (brings one name into scope):
using bmw::platform::EcuManager;
EcuManager mgr2;  // OK now

// Using directive (brings all names — use sparingly, never in headers):
// using namespace bmw::platform;  // pollutes scope

// Anonymous namespace: internal linkage (replaces C's static at file scope)
namespace {
    int file_local_counter = 0;  // only visible in this translation unit
}
```

### Linkage

```cpp
// External linkage: visible across translation units (default for non-static globals)
int global_var = 42;         // external linkage
void global_func() {}        // external linkage

// Internal linkage: visible only in this translation unit
static int local_var = 0;   // internal linkage (C-style, still valid in C++)
namespace { int anon_var; }  // internal linkage (preferred C++ style)

// No linkage: local variables (not accessible outside their scope)
void foo() { int local = 0; }  // no linkage

// inline variables (C++17): defined in multiple TUs, one definition
inline int shared_counter = 0;  // can be in a header without ODR violation

// extern: declare without defining
extern int defined_elsewhere;  // declaration only
```

### The One Definition Rule (ODR)

```cpp
// ODR: every entity must have exactly ONE definition across all translation units
// Violations are undefined behavior (and often silent — linker may pick one arbitrarily)

// Header guards / #pragma once prevent multiple inclusion:
#pragma once
// or:
#ifndef MY_HEADER_H
#define MY_HEADER_H
// ...
#endif

// Things allowed to have multiple identical definitions (inline, templates):
inline int square(int x) { return x * x; }  // OK in header
template<typename T> T max_val(T a, T b) { return a > b ? a : b; }  // OK in header
```

---

## 14. Enumerations

### Plain Enum vs Enum Class

```cpp
// Plain enum: leaks names into enclosing scope
enum Color { RED, GREEN, BLUE };  // RED, GREEN, BLUE pollute the namespace
int red = RED;                     // implicit conversion to int

// enum class (C++11): scoped, no implicit conversion
enum class Opcode : uint8_t {    // underlying type specified
    ADV = 0,
    BXL = 1,
    BST = 2,
    JNZ = 3,
    BXC = 4,
    OUT = 5,
    BDV = 6,
    CDV = 7
};

Opcode op = Opcode::ADV;         // must qualify
// int i = op;                   // ERROR: no implicit conversion
uint8_t i = static_cast<uint8_t>(op);  // explicit cast required

// Switch on enum class (compiler warns on unhandled values):
switch (op) {
    case Opcode::ADV: /* ... */ break;
    case Opcode::BXL: /* ... */ break;
    // ... compiler warns if a case is missing (with -Wswitch)
}
```

### Using Enum in Flags/Bitmasks

```cpp
enum class Permission : uint8_t {
    NONE    = 0,
    READ    = 1 << 0,
    WRITE   = 1 << 1,
    EXECUTE = 1 << 2
};

// enum class doesn't support bitwise ops out of the box — define them:
constexpr Permission operator|(Permission a, Permission b) {
    return static_cast<Permission>(
        static_cast<uint8_t>(a) | static_cast<uint8_t>(b)
    );
}
constexpr bool operator&(Permission a, Permission b) {
    return static_cast<uint8_t>(a) & static_cast<uint8_t>(b);
}

Permission p = Permission::READ | Permission::WRITE;
bool can_read = p & Permission::READ;  // true
```

---

## 15. Unions & Bit Fields

### Unions

```cpp
// Union: all members share the same memory
union FloatBits {
    float    f;
    uint32_t bits;
    uint8_t  bytes[4];
};

// Type punning via union (implementation-defined in C++, well-defined in C99)
// Use memcpy instead for guaranteed correctness:
float f = 3.14f;
uint32_t bits;
std::memcpy(&bits, &f, sizeof(bits));

// Tagged union pattern (use std::variant C++17 instead):
struct Variant {
    enum class Tag { INT, FLOAT, STRING } tag;
    union {
        int    i;
        float  f;
        char   s[32];
    };
};
```

### Bit Fields

```cpp
// Compact storage for hardware register fields
struct UartControlReg {
    uint32_t enable      : 1;   // 1 bit
    uint32_t mode        : 2;   // 2 bits
    uint32_t baud_index  : 4;   // 4 bits (values 0-15)
    uint32_t reserved    : 25;  // 25 bits, pads to 32 total
};
static_assert(sizeof(UartControlReg) == 4);

UartControlReg reg = {};
reg.enable = 1;
reg.baud_index = 3;

// CAUTION: bit field layout is implementation-defined!
// - Bit ordering (MSB or LSB first) depends on compiler/architecture
// - For hardware registers, use bit shifts and masks instead:
volatile uint32_t* UART_CR = reinterpret_cast<volatile uint32_t*>(0x40011000);
*UART_CR |=  (1u << 13);   // Set UE bit (bit 13)
*UART_CR &= ~(1u << 12);   // Clear M bit (bit 12)
// This is unambiguous regardless of compiler
```

---

## Summary: Key Concepts for Interview

| Topic | What to Demonstrate |
|---|---|
| Types | Always use `uint32_t` etc., know promotions, UAC |
| Pointers | Pointer to const vs const pointer, nullptr not NULL |
| References | lvalue vs rvalue refs, const ref extends temporaries |
| Value categories | lvalue/rvalue/xvalue — why it matters for move semantics |
| Memory | Stack vs heap tradeoffs, BSS/data/text sections |
| Object lifetime | Construction/destruction order, placement new |
| Classes | MIL, delegating ctors, `explicit`, virtual dtors |
| Rule of 3/5/0 | Know all three, prefer Rule of Zero with smart pointers |
| RAII | The fundamental C++ idiom — tie every resource to an object |
| Operator overloading | Return types, member vs non-member, spaceship |
| Inheritance | vtable cost, override/final, virtual destructor |
| const | `const` member functions, mutable, const propagation |
| volatile | For MMIO only, NOT for threading (use atomic) |
| Casting | Never C-style cast; know all four C++ casts |
| Namespaces | Avoid `using namespace` in headers, anonymous for TU-local |
| enum class | Always prefer over plain enum; specify underlying type |

---

*Document 1 of 5 — Continue to Document 2: Move Semantics, Templates & Generic Programming*
