# C++ Mastery Guide — Document 4 of 5
## Modern C++ Features: C++11 Through C++20

> **Scope:** Every significant language feature introduced in C++11, C++14, C++17, and C++20 — organized by version, explained with practical embedded-relevant examples. This document is your "what changed and why it matters" reference. Interviewers will ask "what C++17 features do you use?" — this document ensures you have a concrete, detailed answer for every version.

---

## Table of Contents

1. [C++11 — The Revolution](#1-c11--the-revolution)
2. [C++14 — Refinements](#2-c14--refinements)
3. [C++17 — Productivity Boost](#3-c17--productivity-boost)
4. [C++20 — The Big Leap](#4-c20--the-big-leap)
5. [constexpr — Evolution Across Versions](#5-constexpr--evolution-across-versions)
6. [Structured Bindings (C++17)](#6-structured-bindings-c17)
7. [Deduction Guides & CTAD (C++17)](#7-deduction-guides--ctad-c17)
8. [Modules (C++20)](#8-modules-c20)
9. [Coroutines (C++20)](#9-coroutines-c20)
10. [std::chrono — Time in C++](#10-stdchrono--time-in-c)
11. [Attributes](#11-attributes)
12. [Aggregates & Designated Initializers (C++20)](#12-aggregates--designated-initializers-c20)
13. [Three-Way Comparison / Spaceship (C++20)](#13-three-way-comparison--spaceship-c20)
14. [std::format (C++20)](#14-stdformat-c20)
15. [Feature Checklist: What to Know Per Version](#15-feature-checklist-what-to-know-per-version)

---

## 1. C++11 — The Revolution

C++11 was the biggest change to C++ since its creation. It effectively made C++ a new language.

### auto Type Deduction

```cpp
// auto deduces the type from the initializer:
auto i = 42;                     // int
auto d = 3.14;                   // double
auto s = std::string("hello");   // std::string
auto v = std::vector<int>{1,2,3}; // std::vector<int>

// With iterators (the original motivation):
// Before:
std::map<std::string, std::vector<int>>::iterator it = m.begin();
// After:
auto it = m.begin();             // much cleaner

// auto strips references and const by default:
const int& cr = i;
auto a1 = cr;   // a1 is int (not const int&!) — deduced as value type
auto& a2 = cr;  // a2 is const int& (reference preserved)
auto&& a3 = cr; // a3 is const int& (universal ref to lvalue → lvalue ref)

// DANGER: auto with initializer_list
auto il = {1,2,3};  // std::initializer_list<int>, NOT vector<int>!
// This is a common source of bugs

// decltype(auto): preserves reference-ness (C++14)
decltype(auto) get_ref() { return some_ref; }  // deduces as reference
```

### nullptr

```cpp
// The type-safe null pointer constant
int* p = nullptr;      // not NULL, not 0

// nullptr has type std::nullptr_t
void f(int);
void f(int*);
f(nullptr);   // calls f(int*) — unambiguous
f(NULL);      // ambiguous or calls f(int) — dangerous!
f(0);         // calls f(int) — wrong!
```

### Range-Based For Loops

```cpp
std::vector<int> v = {1,2,3,4,5};

// By value (copies elements):
for (int x : v) { /* x is a copy */ }

// By const reference (no copy, read-only):
for (const int& x : v) { /* x is a reference to element */ }

// By reference (can modify):
for (int& x : v) { x *= 2; }  // doubles all elements

// With auto (recommended):
for (const auto& x : v) { /* type-safe, no copies */ }

// Works on any type with begin()/end():
for (const auto& [key, val] : map) { /* structured binding */ }

// C array:
int arr[] = {1,2,3};
for (int x : arr) { /* works! */ }
```

### Uniform Initialization / Initializer Lists

```cpp
// Brace initialization works everywhere:
int i{42};
double d{3.14};
std::vector<int> v{1,2,3,4,5};
std::map<std::string,int> m{{"a",1},{"b",2}};

struct Point { int x, y; };
Point p{1, 2};   // aggregate initialization

// Narrowing conversions are ERRORS with brace init:
int i2 = 3.7;   // OK but truncates silently
// int i3{3.7}; // ERROR: narrowing conversion detected at compile time

// Default initialization:
int arr[5]{};    // all zeros (value-initialization)
std::vector<int> v2(10, 0);  // 10 zeros

// Prefer {} over () to avoid "Most Vexing Parse":
std::vector<int> v3(10);     // 10 default-initialized ints
// std::vector<int> v4();    // This is a FUNCTION DECLARATION, not a vector!
std::vector<int> v5{};       // empty vector (unambiguous)
```

### static_assert

```cpp
// Compile-time assertion — catches errors before runtime:
static_assert(sizeof(uint32_t) == 4, "uint32_t must be 4 bytes");
static_assert(alignof(uint64_t) >= 4, "uint64_t must have >= 4 byte alignment");

// Useful in templates:
template<typename T>
class HardwareRegister {
    static_assert(std::is_integral_v<T>, "Register type must be integral");
    static_assert(sizeof(T) <= 4, "Register must fit in 32 bits");
    volatile T* addr_;
};

// C++17: static_assert without message
static_assert(sizeof(void*) == 8);  // message is optional in C++17+
```

### `using` Type Aliases (Superior to typedef)

```cpp
// Old typedef:
typedef unsigned char byte_t;
typedef void (*callback_t)(int, void*);  // function pointer typedef: ugly syntax

// New using alias:
using byte_t     = unsigned char;
using callback_t = void(*)(int, void*);  // function pointer: same syntax as declaration

// Template alias (typedef CAN'T do this):
template<typename T>
using Vec = std::vector<T>;

Vec<int>    vi;     // std::vector<int>
Vec<double> vd;     // std::vector<double>

// Alias for complex types:
using SensorMap = std::unordered_map<std::string, std::unique_ptr<SensorBase>>;
using IrqHandler = std::function<void(uint32_t status)>;
```

### override and final

```cpp
class Base {
public:
    virtual void foo(int);
    virtual void bar() const;
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    void foo(int) override;    // compiler checks: does Base have virtual foo(int)?
    // void foo(double) override; // ERROR: typo/wrong signature caught!
    // void bar() override;       // ERROR: missing const!

    void bar() const override final;  // no further override
};

class Leaf : public Derived {
    // void bar() const override; // ERROR: bar() is final
};

class Sealed final : public Base { };
// class More : public Sealed { };  // ERROR: Sealed is final
```

### Delegating Constructors & Default/Delete

(Covered in Document 1 — key C++11 additions)

### Lambda Expressions

(Covered in Document 3 — full deep dive)

### `std::move` and Move Semantics

(Covered in Document 2 — full deep dive)

### `std::thread` and Concurrency Primitives

(Covered in Document 3)

### Variadic Templates

(Covered in Document 2)

### `constexpr` Functions (C++11)

```cpp
// Very restricted in C++11: single return statement only
constexpr int factorial_11(int n) {
    return (n <= 1) ? 1 : n * factorial_11(n - 1);  // ternary required in C++11
}
static_assert(factorial_11(5) == 120);
```

### `noexcept`

```cpp
void safe_op() noexcept { /* guaranteed not to throw */ }

// Conditional noexcept:
template<typename T>
void swap_noexcept(T& a, T& b)
    noexcept(noexcept(std::swap(a, b))) {  // noexcept if swap() is noexcept
    std::swap(a, b);
}
```

### `enum class` (Scoped Enumerations)

(Covered in Document 1)

### Right Angle Brackets

```cpp
// C++03: required space between >> to avoid >> being parsed as right-shift
std::vector<std::vector<int> > v03;  // space required

// C++11: no space needed
std::vector<std::vector<int>> v11;   // works fine
```

### Inline Namespaces (C++11)

```cpp
// Inline namespace: members are accessible from the enclosing namespace
// Used for ABI versioning:
namespace bmw {
    inline namespace v2 {  // current version
        struct Config { int new_field; };
    }
    namespace v1 {  // legacy
        struct Config { /* old layout */ };
    }
}

bmw::Config c;     // gets bmw::v2::Config (inline)
bmw::v1::Config c1; // explicit old version
```

---

## 2. C++14 — Refinements

C++14 is a small but important release — mostly fixes and conveniences.

### Generic Lambdas

```cpp
// C++11: explicit type required
auto add11 = [](int a, int b) { return a + b; };

// C++14: auto parameters make lambdas polymorphic
auto add14 = [](auto a, auto b) { return a + b; };
add14(1, 2);          // int + int
add14(1.5, 2.5);      // double + double
add14(1, 2.5);        // int + double -> double

// Generic lambda in algorithm:
std::sort(v.begin(), v.end(), [](const auto& a, const auto& b) {
    return a.priority > b.priority;  // works for any type with .priority
});
```

### Lambda Capture Initializers (Init Capture)

```cpp
// C++11: could only capture existing variables
// C++14: can initialize new variables in capture

// Move capture (crucial for unique_ptr):
auto ptr = std::make_unique<Sensor>(42);
auto lambda = [p = std::move(ptr)]() {
    p->read();  // lambda owns the Sensor
};
// ptr is now null; lambda owns the unique_ptr

// Rename capture:
int x = 42;
auto lam = [y = x * 2]() { return y; };  // y = 84, captured by value

// Capture from non-local scope:
auto make_adder(int n) {
    return [n = n](int x) { return x + n; };  // captures n by value
}
```

### `std::make_unique` (C++14)

```cpp
// C++11 had make_shared but forgot make_unique!
// C++14 adds it:
auto p = std::make_unique<Widget>(arg1, arg2);  // exception-safe, no new

// Before (C++11): had to use raw new:
std::unique_ptr<Widget> p2(new Widget(arg1, arg2));  // ok but slightly less safe
```

### Return Type Deduction for Functions

```cpp
// C++11: trailing return type required for complex deduction
auto add(int a, int b) -> decltype(a + b) { return a + b; }

// C++14: just use auto, compiler deduces from return statement
auto add14(int a, int b) { return a + b; }  // deduces int

// All return statements must deduce the same type:
auto f(bool flag) {
    if (flag) return 42;    // int
    return 42.0;            // double — ERROR! inconsistent
}

// decltype(auto): preserves reference-ness
decltype(auto) get(int& x) { return x; }  // returns int& (not int!)
```

### `[[deprecated]]` Attribute

```cpp
[[deprecated("use new_api() instead")]]
void old_api() { /* ... */ }

old_api();  // compiler warning: 'old_api' is deprecated: use new_api() instead
```

### Integer Literals Separators

```cpp
// C++14: single quotes as digit separator for readability
uint32_t mask  = 0xFF'FF'FF'FF;
uint64_t freq  = 1'000'000'000ULL;  // 1 GHz
int      big   = 1'234'567;
double   pi    = 3.141'592'653'589'793;
uint8_t  byte  = 0b1010'0101;        // binary with separator
```

### `std::integer_sequence` and `std::index_sequence`

```cpp
// Used to unpack tuples and array indices at compile time:
template<typename Tuple, size_t... Is>
void print_tuple_impl(const Tuple& t, std::index_sequence<Is...>) {
    ((std::cout << std::get<Is>(t) << " "), ...);  // C++17 fold
}

template<typename... Args>
void print_tuple(const std::tuple<Args...>& t) {
    print_tuple_impl(t, std::make_index_sequence<sizeof...(Args)>{});
}

std::tuple<int, double, std::string> tp{42, 3.14, "hello"};
print_tuple(tp);  // 42 3.14 hello
```

---

## 3. C++17 — Productivity Boost

C++17 brought significant quality-of-life improvements without as radical a change as C++11.

### Structured Bindings (Covered in Section 6)

### `if` and `switch` with Initializer

```cpp
// C++17: initialize a variable scoped to the if/switch
if (auto it = map.find("key"); it != map.end()) {
    use(it->second);
    // it only accessible inside this if block
}
// it doesn't exist here — tight scoping

// With error codes:
if (int err = do_something(); err != 0) {
    handle_error(err);
}

// switch with init:
switch (auto status = get_status(); status) {
    case OK:    break;
    case ERROR: handle(); break;
}
```

### `std::optional`, `std::variant`, `std::any`

(Covered in Document 3)

### `std::string_view`, `std::span`

(Covered in Document 3)

### Fold Expressions (Covered in Document 2)

### `if constexpr` (Covered in Document 2)

### Class Template Argument Deduction (CTAD)

```cpp
// C++17: template arguments deduced from constructor arguments
std::pair  p(42, "hello");    // pair<int, const char*>
std::vector v{1,2,3};        // vector<int>
std::tuple  t(1, 2.0, "hi"); // tuple<int, double, const char*>
std::lock_guard lk(mtx);     // lock_guard<mutex>

// Custom deduction guides:
template<typename T>
class Wrapper {
    T data;
public:
    Wrapper(T val) : data(val) {}
};
// Implicit deduction guide generated:
// template<typename T> Wrapper(T) -> Wrapper<T>;
Wrapper w(42);   // deduces Wrapper<int>
```

### Inline Variables (C++17)

```cpp
// Before C++17: defining a const in a header caused multiple definition errors
// You had to extern declare in header, define in ONE .cpp file

// C++17: inline variables can be defined in headers, ODR-safe
// header.h:
inline const int MAX_SENSORS = 32;
inline std::mutex global_mutex;  // can be in header now!

// Each TU sees the same variable (linker merges)
```

### `std::filesystem` (C++17)

```cpp
#include <filesystem>
namespace fs = std::filesystem;

// Path operations:
fs::path p = "/etc/config";
p.filename();        // "config"
p.extension();       // ""
p.parent_path();     // "/etc"
p / "subdir";        // "/etc/config/subdir"

// File operations:
fs::exists(p);
fs::is_directory(p);
fs::file_size(p);
fs::create_directories("/tmp/a/b/c");
fs::copy_file("src.txt", "dst.txt");
fs::remove("file.txt");

// Directory iteration:
for (const auto& entry : fs::directory_iterator("/proc")) {
    std::cout << entry.path() << "\n";
}
```

### `std::byte` (C++17)

```cpp
#include <cstddef>

// std::byte: a type specifically for raw memory (not integer, not character)
std::byte b{0xFF};
std::byte b2 = std::byte{42};

// Only bitwise ops defined (no arithmetic — it's not a number):
b | std::byte{0x0F};
b & std::byte{0xF0};
b ^ std::byte{0xAA};
b << 1;    // shift
b >> 1;

// Convert to/from integer:
uint8_t i = std::to_integer<uint8_t>(b);
std::byte from_int = static_cast<std::byte>(0xAB);

// Use in raw buffer:
std::vector<std::byte> buffer(1024);
std::memset(buffer.data(), 0, buffer.size());
```

### Parallel Algorithms (C++17)

```cpp
#include <execution>

std::vector<int> v(1'000'000);
std::iota(v.begin(), v.end(), 0);

// Parallel sort:
std::sort(std::execution::par, v.begin(), v.end());

// Parallel transform:
std::transform(std::execution::par_unseq,
               v.begin(), v.end(),
               v.begin(),
               [](int x){ return x * x; });

// Execution policies:
// std::execution::seq       — sequential (like normal)
// std::execution::par       — parallel (multiple threads)
// std::execution::par_unseq — parallel + vectorized (SIMD)
// std::execution::unseq     — vectorized, single thread (C++20)
```

### `std::apply` and `std::invoke` (C++17)

```cpp
// std::apply: call a function with tuple elements as arguments
auto args = std::make_tuple(1, 2.5, "hello");
auto result = std::apply([](int a, double b, const char* c) {
    std::cout << a << " " << b << " " << c;
}, args);

// std::invoke: uniform call syntax for functions, function pointers, member functions
struct Foo { int bar(int x) { return x * 2; } };
Foo foo;
std::invoke(&Foo::bar, foo, 21);  // calls foo.bar(21) → 42
std::invoke(some_lambda, args...);
```

### `__has_include` (C++17)

```cpp
// Check if a header is available:
#if __has_include(<optional>)
    #include <optional>
    using std::optional;
#elif __has_include(<experimental/optional>)
    #include <experimental/optional>
    using std::experimental::optional;
#else
    // provide fallback
#endif
```

---

## 4. C++20 — The Big Leap

C++20 is the second biggest change to C++ after C++11. It's transformative.

### Concepts

(Covered in Document 2 — full deep dive)

### Ranges

(Covered in Document 3 — iterators section)

### Coroutines

(Covered in Section 9 below)

### Modules

(Covered in Section 8 below)

### `std::span`

(Covered in Document 3)

### `std::format`

(Covered in Section 14)

### Designated Initializers

(Covered in Section 12)

### Three-Way Comparison `<=>`

(Covered in Section 13 and Document 1)

### `consteval` and `constinit`

```cpp
// consteval: function MUST be evaluated at compile time (stronger than constexpr)
consteval int square(int n) { return n * n; }
constexpr int x = square(5);   // OK: compile time
// int y = square(runtime_val); // ERROR: consteval can't run at runtime

// constinit: variable MUST be initialized at compile time (prevents static init order fiasco)
constinit int global = compute_at_compiletime();  // OK
// constinit int bad = runtime_func();            // ERROR: not constant
// Note: constinit doesn't make variable const — it can still change at runtime
constinit int counter = 0;
counter++;  // OK: constinit just requires compile-time initialization
```

### `[[likely]]` and `[[unlikely]]` (C++20)

```cpp
// Hint to compiler about branch probabilities (for branch prediction optimization):
void process(int val) {
    if (val > 0) [[likely]] {
        handle_normal(val);
    } else [[unlikely]] {
        handle_error();
    }
}

// In tight loops:
for (int i = 0; i < n; i++) {
    if (is_error(arr[i])) [[unlikely]] {
        report_error(arr[i]);
    } else {
        process(arr[i]);
    }
}
```

### `std::jthread` (Joinable Thread, C++20)

```cpp
#include <thread>

// jthread: automatically joins on destruction (no std::terminate())
// Also supports cooperative cancellation via stop_token
{
    std::jthread t([](std::stop_token st) {
        while (!st.stop_requested()) {
            do_work();
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
        // cleanup on stop
    });
    // ... work ...
    // t.request_stop() automatically called in destructor
}  // t joins here automatically

// Explicit stop:
std::jthread worker(task);
// ...
worker.request_stop();  // signal stop
worker.join();          // wait for it to notice the stop and return
```

### `std::latch` and `std::barrier` (C++20)

```cpp
#include <latch>
#include <barrier>

// latch: single-use countdown synchronization point
std::latch sync_point(3);  // 3 threads must count down

// 3 threads:
void thread_work(std::latch& latch) {
    do_initialization();
    latch.count_down();   // decrement counter
    latch.wait();         // block until counter reaches 0
    // all 3 threads initialized before any continues
    do_coordinated_work();
}

// barrier: reusable, reset after each phase
std::barrier phase_barrier(3, []{ std::cout << "phase complete\n"; });

void worker(std::barrier<>& barrier) {
    while (has_work()) {
        do_phase_work();
        barrier.arrive_and_wait();  // sync between phases
    }
}
```

### Concepts Library (`<concepts>`) Standard Concepts

```cpp
// Most useful standard concepts for embedded C++:
std::integral<T>            // int, long, etc.
std::signed_integral<T>     // signed ints
std::unsigned_integral<T>   // unsigned ints
std::floating_point<T>      // float, double
std::same_as<T, U>          // T == U
std::derived_from<T, Base>  // T : Base
std::convertible_to<From, To>
std::invocable<F, Args...>  // callable with args
std::predicate<F, Args...>  // returns bool
std::regular<T>             // copyable + equality_comparable
std::semiregular<T>         // copyable + default_constructible
std::equality_comparable<T>
std::totally_ordered<T>     // has all comparison operators
```

### Abbreviated Function Templates (C++20)

```cpp
// C++14 generic lambda (only in lambdas):
auto f = [](auto x) { return x * 2; };

// C++20 abbreviated function template (any function):
auto double_it(auto x) { return x * 2; }
// Equivalent to:
template<typename T>
auto double_it(T x) { return x * 2; }

// With concept constraint:
auto halve(std::floating_point auto x) { return x / 2.0; }
// double d = halve(4.0);   // OK
// int i = halve(4);        // ERROR: int is not floating_point
```

---

## 5. constexpr — Evolution Across Versions

### C++11: Very Restricted

```cpp
// C++11: only a single return statement, no loops, no local variables
constexpr int pow11(int base, int exp) {
    return (exp == 0) ? 1 : base * pow11(base, exp - 1);  // recursion required
}
```

### C++14: Much More Useful

```cpp
// C++14: loops, local variables, multiple statements allowed
constexpr int pow14(int base, int exp) {
    int result = 1;
    for (int i = 0; i < exp; i++) result *= base;
    return result;
}
static_assert(pow14(2, 10) == 1024);
```

### C++17: constexpr if, constexpr lambdas

```cpp
// constexpr if (covered in Document 2)
// constexpr lambdas:
constexpr auto square = [](int x) { return x * x; };
static_assert(square(5) == 25);

// constexpr in more standard library functions:
constexpr std::array<int, 5> make_squares() {
    std::array<int, 5> a{};
    for (int i = 0; i < 5; i++) a[i] = i * i;
    return a;
}
constexpr auto squares = make_squares();  // computed at compile time
```

### C++20: Almost Everything is constexpr

```cpp
// constexpr virtual functions, try/catch in constexpr, dynamic_cast, new/delete
// std::vector and std::string are constexpr (but only in constexpr context!)

constexpr int sum_to_n(int n) {
    std::vector<int> v;         // constexpr in C++20!
    for (int i = 1; i <= n; i++) v.push_back(i);
    int sum = 0;
    for (int x : v) sum += x;
    return sum;
}
static_assert(sum_to_n(10) == 55);

// consteval: must be compile-time
consteval int must_be_compiletime(int n) { return n * n; }

// constinit: must be initialized at compile time, but can change at runtime
constinit int startup_count = must_be_compiletime(5);  // 25
startup_count++;  // fine: constinit doesn't mean const
```

---

## 6. Structured Bindings (C++17)

Structured bindings decompose aggregates (structs, pairs, tuples, arrays) into named components.

```cpp
// Pair decomposition:
std::pair<int, std::string> p{42, "hello"};
auto [id, name] = p;   // id=42, name="hello"

// Map iteration (the killer use case):
std::map<std::string, int> scores;
for (const auto& [key, val] : scores) {
    std::cout << key << ": " << val << "\n";
}

// Tuple decomposition:
auto [x, y, z] = std::make_tuple(1, 2.0, "three");

// Array decomposition:
int arr[3] = {10, 20, 30};
auto [a, b, c] = arr;    // a=10, b=20, c=30

// Struct decomposition (members in declaration order):
struct Point { float x; float y; float z; };
Point pt{1.0f, 2.0f, 3.0f};
auto [px, py, pz] = pt;

// By reference (modify original):
for (auto& [key, val] : map) {
    val *= 2;  // modifies the map values
}

// From functions returning pair:
auto [it, inserted] = map.emplace("key", 42);
if (inserted) { /* new element */ }
if (it != map.end()) { /* always true here */ }

// Error-code pattern:
auto [result, error] = try_operation();
if (error) handle_error(error);
else use_result(result);
```

---

## 7. Deduction Guides & CTAD (C++17)

```cpp
// CTAD: Class Template Argument Deduction
// Compiler generates deduction guides from constructors:

template<typename T>
class Box {
    T data_;
public:
    Box(T val) : data_(val) {}
    // Implicit deduction guide: Box(T) -> Box<T>
};

Box b(42);     // deduces Box<int>
Box b2(3.14);  // deduces Box<double>

// Custom deduction guides (when implicit isn't right):
template<typename T>
class Container {
    T* data_;
    size_t size_;
public:
    Container(T* ptr, size_t n) : data_(ptr), size_(n) {}
};

// Custom guide: deduce T from pointer type
template<typename T>
Container(T*, size_t) -> Container<T>;

int arr[] = {1,2,3};
Container c(arr, 3);  // deduces Container<int>

// Standard library uses CTAD:
std::vector v = {1,2,3};           // vector<int>
std::pair p = {42, "hello"};       // pair<int, const char*>
std::array a = {1,2,3,4,5};        // array<int, 5>
std::optional o = 42;              // optional<int>
```

---

## 8. Modules (C++20)

Modules replace the #include preprocessor system. Benefits: faster compilation, no macro leakage, no include order dependencies.

```cpp
// Traditional header approach:
// sensor.h (every includer pays the parse cost, macros leak)
#pragma once
#define SENSOR_DEBUG 1  // leaks to includers!
class Sensor { /* ... */ };

// Module approach:

// sensor_module.ixx (module interface unit):
export module sensor;

export class Sensor {
public:
    float read() const;
    void calibrate(float offset);
};

// Internal: not exported, not visible to importers
class SensorImpl { /* ... */ };

// Importing:
import sensor;
// import <vector>;  // import standard library headers as modules

Sensor s;
s.read();

// Benefits:
// 1. No header guards needed
// 2. Macros don't leak between modules
// 3. Dramatically faster compilation (parsed once, cached)
// 4. No include order fragility
// 5. Better encapsulation (non-exported internals truly hidden)

// Note: toolchain support (build system + compiler) still maturing in 2024-2026
// Used with CMake 3.28+ and modern compilers
```

---

## 9. Coroutines (C++20)

Coroutines are functions that can suspend and resume execution. Foundation for async programming without callbacks.

```cpp
#include <coroutine>

// A coroutine is a function that uses co_await, co_yield, or co_return.

// Simple generator coroutine:
// (requires a generator type wrapping coroutine_handle)

template<typename T>
struct Generator {
    struct promise_type {
        T value_;
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        Generator get_return_object() {
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always yield_value(T val) {
            value_ = val;
            return {};
        }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle_;
    explicit Generator(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~Generator() { if (handle_) handle_.destroy(); }

    bool next() { handle_.resume(); return !handle_.done(); }
    T value() { return handle_.promise().value_; }
};

// Using co_yield to produce a sequence:
Generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;         // suspend and yield a value
        auto c = a + b;
        a = b;
        b = c;
    }
}

// Consume:
auto gen = fibonacci();
for (int i = 0; i < 10 && gen.next(); i++) {
    std::cout << gen.value() << " ";
}
// 0 1 1 2 3 5 8 13 21 34

// Three coroutine keywords:
// co_yield expr    — suspend, produce a value
// co_await expr    — suspend until awaitable is ready
// co_return expr   — complete the coroutine (analogous to return)

// In embedded use case: async sensor reading without blocking
```

### Coroutines for Async I/O (Conceptual)

```cpp
// With a proper async framework (e.g., Asio with C++20 coroutines):
async_task<std::vector<uint8_t>> read_sensor_async(Sensor& s) {
    // Non-blocking wait for sensor data ready
    co_await s.wait_for_data();

    // Non-blocking read
    auto data = co_await s.async_read();
    co_return data;
}

// Coroutines are the foundation of C++20 async programming
// They avoid callback hell while keeping code looking synchronous
```

---

## 10. std::chrono — Time in C++

```cpp
#include <chrono>
using namespace std::chrono;
using namespace std::chrono_literals;  // for suffix syntax

// Durations:
auto d1 = 100ms;          // 100 milliseconds
auto d2 = 5s;             // 5 seconds
auto d3 = 2min;           // 2 minutes
auto d4 = 1h;             // 1 hour
auto d5 = 500us;          // 500 microseconds
auto d6 = 250ns;          // 250 nanoseconds

// Duration arithmetic:
auto total = 1h + 30min + 45s;  // 1:30:45

// Duration conversions:
auto ms_count = duration_cast<milliseconds>(total).count();  // 5445000

// Clocks:
// steady_clock: monotonic, never goes backward — use for measuring elapsed time
// system_clock: wall clock, can jump (NTP adjustments) — use for timestamps
// high_resolution_clock: finest granularity (may alias steady or system)

// Measuring elapsed time:
auto start = steady_clock::now();
do_work();
auto end = steady_clock::now();
auto elapsed = duration_cast<microseconds>(end - start);
std::cout << "took " << elapsed.count() << " µs\n";

// Sleep:
std::this_thread::sleep_for(100ms);
std::this_thread::sleep_until(steady_clock::now() + 1s);

// Time points:
time_point<steady_clock> tp = steady_clock::now();
time_point<steady_clock> deadline = tp + 500ms;
bool expired = steady_clock::now() > deadline;

// In embedded: chrono is great for timeouts and profiling
// For hardware timers, use platform-specific APIs
```

---

## 11. Attributes

Attributes provide a standard way to express implementation hints and requirements.

```cpp
// [[nodiscard]]: warn if return value is ignored
[[nodiscard]] int read_sensor() { return adc_value_; }
// read_sensor();   // WARNING: ignoring nodiscard return value

// With message (C++20):
[[nodiscard("check for errors!")]] std::error_code write(const uint8_t* buf, size_t len);

// [[maybe_unused]]: suppress unused variable/parameter warnings
void callback([[maybe_unused]] int irrelevant_param) {
    // param required by interface but not used in this implementation
}
[[maybe_unused]] static void debug_helper() { /* ... */ }

// [[deprecated]]: mark as deprecated
[[deprecated("use new_api()")]] void old_api();

// [[likely]] / [[unlikely]] (C++20): branch prediction hints
if (error) [[unlikely]] { handle_error(); }
else [[likely]] { process_data(); }

// [[fallthrough]]: intentional switch fallthrough (no warning)
switch (mode) {
    case Mode::READ:
        do_read();
        [[fallthrough]];    // intentionally falls to WRITE
    case Mode::WRITE:
        do_write();
        break;
}

// [[no_unique_address]] (C++20): allow empty base optimization for members
template<typename Allocator>
class Container {
    [[no_unique_address]] Allocator alloc_;  // takes no space if Allocator is empty
    T* data_;
};
```

---

## 12. Aggregates & Designated Initializers (C++20)

```cpp
// Designated initializers: name the members you're initializing
struct Config {
    uint32_t baud_rate   = 115200;
    uint8_t  data_bits   = 8;
    char     parity      = 'N';
    uint8_t  stop_bits   = 1;
    bool     flow_ctrl   = false;
    uint32_t timeout_ms  = 1000;
};

// C++20: designate which fields to set (others get defaults):
Config cfg{
    .baud_rate  = 9600,
    .parity     = 'E',
    .timeout_ms = 500
    // data_bits, stop_bits, flow_ctrl use defaults
};

// Rules:
// - Must be in declaration order (unlike C99)
// - Cannot mix designated and non-designated
// - Nested: .member.submember not supported, initialize with braces

struct Network {
    std::string host;
    uint16_t    port = 80;
    bool        tls  = true;
};

Network n{.host = "192.168.1.1", .port = 8080};  // tls = true (default)

// Excellent for hardware register initialization:
struct GpioConfig {
    enum class Direction { Input, Output } dir = Direction::Input;
    enum class Pull { None, Up, Down }     pull = Pull::None;
    bool                                   open_drain = false;
    uint8_t                                speed = 0;
};

GpioConfig output_pin{
    .dir   = GpioConfig::Direction::Output,
    .pull  = GpioConfig::Pull::Up,
    .speed = 3
};
```

---

## 13. Three-Way Comparison / Spaceship (C++20)

```cpp
#include <compare>

// auto operator<=>: generate all 6 comparisons with a single operator
struct Packet {
    uint32_t sequence;
    uint8_t  priority;

    // Default: lexicographic comparison of all members in order
    auto operator<=>(const Packet&) const = default;
    // Generates: <, >, <=, >=, ==, !=
};

// Custom spaceship operator:
struct Temperature {
    float celsius;

    std::partial_ordering operator<=>(const Temperature& rhs) const {
        // partial_ordering because NaN comparisons are unordered
        return celsius <=> rhs.celsius;
    }
    bool operator==(const Temperature& rhs) const = default;
};

// Return types of <=>:
// strong_ordering:   a == b implies a and b are interchangeable (int, etc.)
// weak_ordering:     a == b means they're equivalent (case-insensitive string)
// partial_ordering:  a == b, a < b, a > b, or UNORDERED (float with NaN)

// Comparison with primitives:
struct Wrapped {
    int val;
    auto operator<=>(int rhs) const { return val <=> rhs; }
    bool operator==(int rhs) const  { return val == rhs; }
};

Wrapped w{42};
bool gt = w > 40;   // true
bool eq = w == 42;  // true
```

---

## 14. std::format (C++20)

`std::format` is Python-style string formatting, replacing `printf` and `stringstream` for most uses:

```cpp
#include <format>

// Basic formatting:
std::string s = std::format("Hello, {}!", "world");  // "Hello, world!"

// Multiple arguments:
std::string msg = std::format("Sensor {} reads {:.2f}°C", 42, 23.456);
// "Sensor 42 reads 23.46°C"

// Format specifiers:
std::format("{:d}",  42);        // decimal
std::format("{:x}",  255);       // hex lowercase: "ff"
std::format("{:X}",  255);       // hex uppercase: "FF"
std::format("{:#x}", 255);       // with prefix: "0xff"
std::format("{:08x}", 255);      // zero-padded: "000000ff"
std::format("{:b}",  42);        // binary: "101010"
std::format("{:e}",  3.14);      // scientific: "3.140000e+00"
std::format("{:.4f}", 3.14159);  // fixed, 4 decimal: "3.1416"
std::format("{:>10}", "right");  // right-aligned in 10: "     right"
std::format("{:<10}", "left");   // left-aligned: "left      "
std::format("{:^10}", "center"); // centered: "  center  "

// Named arguments (with index):
std::format("{0} + {1} = {0}", 21, 42);  // "21 + 42 = 21"

// Printing directly (C++23: std::print, std::println):
// std::println("Value: {}", 42);   // C++23

// Custom type formatting:
template<>
struct std::formatter<MyPoint> {
    constexpr auto parse(format_parse_context& ctx) { return ctx.begin(); }
    auto format(const MyPoint& p, format_context& ctx) const {
        return std::format_to(ctx.out(), "({}, {})", p.x, p.y);
    }
};
std::format("{}", MyPoint{1,2});  // "(1, 2)"

// Embedded relevance: much safer than sprintf (no buffer overflow),
// cleaner than snprintf, and type-safe
```

---

## 15. Feature Checklist: What to Know Per Version

### C++11 Must-Know List
- ✅ `auto` type deduction
- ✅ `nullptr` (not NULL)
- ✅ Range-based `for`
- ✅ Move semantics, rvalue references, `std::move`
- ✅ `unique_ptr`, `shared_ptr`, `weak_ptr`
- ✅ Lambda expressions
- ✅ `constexpr` (basic)
- ✅ `override`, `final`
- ✅ `= default`, `= delete`
- ✅ `enum class`
- ✅ `using` aliases (template-compatible)
- ✅ `static_assert`
- ✅ Variadic templates
- ✅ Initializer lists `{}`
- ✅ `std::thread`, `std::mutex`, `std::atomic`
- ✅ `std::function`
- ✅ `std::tuple`
- ✅ `thread_local`
- ✅ `noexcept`

### C++14 Must-Know List
- ✅ Generic lambdas (`auto` parameters)
- ✅ Lambda init captures (`[x = std::move(p)]`)
- ✅ `std::make_unique`
- ✅ Return type deduction (`auto f()`)
- ✅ `[[deprecated]]`
- ✅ Digit separators (`1'000'000`)
- ✅ `std::integer_sequence`

### C++17 Must-Know List
- ✅ Structured bindings (`auto [a, b] = pair`)
- ✅ `if`/`switch` with initializer (`if (auto x = f(); x > 0)`)
- ✅ `std::optional<T>`
- ✅ `std::variant<Ts...>`
- ✅ `std::string_view`
- ✅ Class Template Argument Deduction (CTAD)
- ✅ `if constexpr`
- ✅ Fold expressions
- ✅ `inline` variables in headers
- ✅ `std::filesystem`
- ✅ `std::byte`
- ✅ Parallel algorithms
- ✅ `std::scoped_lock`
- ✅ `std::apply`, `std::invoke`

### C++20 Must-Know List
- ✅ Concepts (`template<Concept T>`, `requires`)
- ✅ Ranges (`v | views::filter | views::transform`)
- ✅ Coroutines (`co_await`, `co_yield`, `co_return`)
- ✅ Modules (`export module`, `import`)
- ✅ `std::span<T>`
- ✅ `std::format`
- ✅ Designated initializers (`.field = value`)
- ✅ Three-way comparison `<=>`
- ✅ `consteval`, `constinit`
- ✅ `[[likely]]`, `[[unlikely]]`
- ✅ `std::jthread`
- ✅ `std::latch`, `std::barrier`
- ✅ Abbreviated function templates (`auto f(auto x)`)

---

## Key Interview Q&A

**Q: What is the most important new feature of C++11?**
A: Move semantics — it fundamentally changed how C++ handles resource transfer and enabled the modern STL performance model. Everything else (lambdas, auto, smart pointers) builds on top of this.

**Q: What is the difference between `constexpr` and `consteval`?**
A: `constexpr` can be evaluated at both compile time and runtime. `consteval` (C++20) must be evaluated at compile time — if called with a runtime argument, it's a compile error. Use `consteval` when you want to guarantee compile-time evaluation.

**Q: What problem do structured bindings solve?**
A: They eliminate the need to name intermediate pair/tuple variables and make map iteration dramatically cleaner. Before C++17: `auto& kv = *it; kv.first; kv.second`. After: `for (auto& [key, val] : map)`.

**Q: What is `if constexpr` and when do you need it over regular `if`?**
A: In a template function, both branches of a regular `if` are compiled even if the condition is always true/false for a given T. With `if constexpr`, the false branch is discarded and not instantiated — it doesn't even need to compile. Needed when different template instantiations would have different valid operations.

**Q: What are the advantages of `std::format` over `printf`?**
A: Type safety (no format/argument mismatch UB), no buffer overflow (returns string), extensible to custom types, positional arguments, and cleaner syntax. Same motivation as Python's f-strings.

---

*Document 4 of 5 — Continue to Document 5: Design Patterns, Error Handling & Embedded C++ Best Practices*
