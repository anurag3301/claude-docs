# C++ Mastery Guide — Document 3 of 5
## Standard Library, Smart Pointers & Concurrency

> **Scope:** The full standard library toolkit — containers, iterators, algorithms, smart pointers, and the C++11/14/17/20 concurrency model with atomics, mutexes, futures, and memory ordering. In an embedded/automotive interview, smart pointers and concurrency are tested heavily.

---

## Table of Contents

1. [Smart Pointers — The Full Picture](#1-smart-pointers--the-full-picture)
2. [Standard Containers](#2-standard-containers)
3. [Iterators & Ranges (C++20)](#3-iterators--ranges-c20)
4. [Standard Algorithms](#4-standard-algorithms)
5. [std::optional, std::variant, std::any (C++17)](#5-stdoptional-stdvariant-stdany-c17)
6. [std::string_view and std::span (C++17/20)](#6-stdstring_view-and-stdspan-c1720)
7. [Lambdas — Deep Dive](#7-lambdas--deep-dive)
8. [std::function and Callable Types](#8-stdfunction-and-callable-types)
9. [Threading Model — std::thread](#9-threading-model--stdthread)
10. [Mutexes, Locks & Condition Variables](#10-mutexes-locks--condition-variables)
11. [std::atomic — The Complete Model](#11-stdatomic--the-complete-model)
12. [Memory Ordering](#12-memory-ordering)
13. [std::future, std::promise & async](#13-stdfuture-stdpromise--async)
14. [Lock-Free Data Structures](#14-lock-free-data-structures)
15. [Exception Handling](#15-exception-handling)

---

## 1. Smart Pointers — The Full Picture

Smart pointers implement RAII for heap-allocated objects. **Never use raw `new`/`delete` in modern C++.**

### `std::unique_ptr<T>` — Exclusive Ownership

```cpp
#include <memory>

// Creation: always use make_unique (C++14)
auto p = std::make_unique<Sensor>(42);   // allocates Sensor(42), p owns it
// Never: std::unique_ptr<Sensor> p(new Sensor(42));
// Reason: make_unique is exception-safe; new+constructor can leak if other args throw

// Unique ownership: non-copyable
// auto p2 = p;  // ERROR: unique_ptr is not copyable

// Transfer ownership with move:
auto p2 = std::move(p);   // p is now null, p2 owns the Sensor
// p.get() == nullptr

// Access the object:
p2->calibrate();           // arrow operator
(*p2).calibrate();         // dereference
Sensor* raw = p2.get();    // get raw pointer (ownership NOT transferred)

// Release ownership:
Sensor* released = p2.release();  // p2 is now null, caller owns raw pointer
delete released;                   // caller must manually delete

// Reset (delete current, optionally take new):
p2.reset(new Sensor(99));   // delete old sensor, own new one
p2.reset();                  // delete sensor, p2 is now null

// Boolean conversion:
if (p2) { /* p2 is not null */ }

// Arrays:
auto arr = std::make_unique<uint8_t[]>(1024);  // unique_ptr for arrays
arr[0] = 0xFF;  // subscript operator available
// Automatically calls delete[] (not delete) on destruction
```

### Custom Deleters with `unique_ptr`

```cpp
// For resources that need a custom cleanup function
struct FdDeleter {
    void operator()(int* fd) const {
        if (*fd >= 0) close(*fd);
        delete fd;
    }
};
std::unique_ptr<int, FdDeleter> safe_fd(new int(open("file.txt", O_RDONLY)));

// Shorter: lambda deleter
auto file_deleter = [](FILE* f) { if(f) fclose(f); };
std::unique_ptr<FILE, decltype(file_deleter)> safe_file(fopen("a.txt", "r"), file_deleter);

// For malloc'd memory:
auto malloc_deleter = [](void* p) { free(p); };
std::unique_ptr<void, decltype(malloc_deleter)> safe_mem(malloc(1024), malloc_deleter);
```

### `std::shared_ptr<T>` — Shared Ownership

```cpp
// Multiple shared_ptr instances can own the same object
// Object destroyed when the LAST shared_ptr is destroyed (ref count drops to 0)

auto sp1 = std::make_shared<Sensor>(42);  // ref count = 1
{
    auto sp2 = sp1;    // copy: ref count = 2
    auto sp3 = sp1;    // copy: ref count = 3
    sp2->read();
}  // sp2 and sp3 destroyed: ref count = 1
// sp1 still alive: ref count = 1
// Sensor destroyed when sp1 goes out of scope: ref count = 0

// Control block: shared_ptr has two heap allocations if you use new
// make_shared: ONE allocation (control block + object together) - preferred!

// Inspection:
sp1.use_count();   // number of shared_ptrs owning this object
sp1.unique();      // true if use_count() == 1

// Shared ownership across threads:
// Incrementing/decrementing the ref count is thread-safe
// BUT accessing the pointed-to object is NOT thread-safe!
```

### `std::weak_ptr<T>` — Non-Owning Observer

```cpp
// Does not extend lifetime. Must check if still valid before use.
auto sp = std::make_shared<Sensor>(42);
std::weak_ptr<Sensor> wp = sp;  // observes sp but doesn't own

// To use: lock() converts to shared_ptr (or null if object expired)
if (auto sp2 = wp.lock()) {
    sp2->read();  // safe: object still alive
} else {
    // object was destroyed
}

// Use case: breaking circular references
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node>   prev;  // back-pointer: weak to avoid cycle
};
// If prev were shared_ptr: cycle -> ref count never reaches 0 -> memory leak
```

### When to Use Which

```cpp
// Default: unique_ptr
// - Single owner, explicit transfer of ownership
// - Zero overhead over raw pointer
auto device = std::make_unique<UartDevice>("/dev/ttyS0");

// shared_ptr when:
// - Multiple owners need the object
// - Object lifetime is unclear at design time
// - Callback/observer pattern across threads
auto config = std::make_shared<Config>(load_config());
register_callback([config]() { config->reload(); });  // config shared with callback

// weak_ptr when:
// - You need to observe but not own
// - Breaking cycles in graph structures
// - Cache implementations (entry can be evicted)

// In embedded: prefer unique_ptr (shared_ptr has atomic ref count overhead)
// In safety-critical: avoid both — prefer static allocation or object pools
```

---

## 2. Standard Containers

### Choosing the Right Container

| Container | Access | Insert/Delete | Memory | Use When |
|---|---|---|---|---|
| `vector<T>` | O(1) random | O(n) middle, O(1) back | Contiguous | Default sequence |
| `deque<T>` | O(1) random | O(1) front and back | Chunks | Queue with index access |
| `list<T>` | O(n) | O(1) anywhere | Scattered | Frequent middle insert/delete |
| `array<T,N>` | O(1) random | Fixed size | Stack/static | Fixed-size, no heap |
| `map<K,V>` | O(log n) | O(log n) | Tree nodes | Sorted key-value |
| `unordered_map<K,V>` | O(1) avg | O(1) avg | Hash buckets | Fast lookup, unsorted |
| `set<T>` | O(log n) | O(log n) | Tree nodes | Sorted unique values |
| `unordered_set<T>` | O(1) avg | O(1) avg | Hash buckets | Fast membership test |
| `stack<T>` | O(1) top | O(1) | Adaptor | LIFO |
| `queue<T>` | O(1) front/back | O(1) | Adaptor | FIFO |
| `priority_queue<T>` | O(1) max | O(log n) | Heap | Always get max |

### `std::vector` — The Workhorse

```cpp
std::vector<int> v;

// Reserve: avoid reallocations (critical for performance)
v.reserve(100);   // allocates space for 100 ints, size stays 0

// Size vs Capacity:
v.push_back(1);
v.size();       // number of elements: 1
v.capacity();   // allocated space: >= 1 (probably 100 from reserve)

// Growth strategy: doubles capacity on overflow (amortized O(1) push_back)
// Without reserve: causes O(log n) reallocations for n elements

// Emplace vs push: emplace constructs in-place (no temporary)
v.emplace_back(42);            // constructs int(42) in-place
v.push_back(42);               // creates int(42) temporary, then moves/copies

// For complex types:
std::vector<std::string> sv;
sv.emplace_back("hello");      // constructs string in-place from const char*
sv.push_back("hello");         // creates string temp, then moves

// Access:
v[0];           // no bounds check (UB if out of range)
v.at(0);        // bounds checked (throws std::out_of_range)
v.front();      // first element
v.back();       // last element
v.data();       // pointer to underlying array (useful for C interop)

// Removal:
v.pop_back();   // remove last O(1)
v.erase(v.begin() + 2);  // remove element at index 2, O(n)
// Erase-remove idiom:
v.erase(std::remove(v.begin(), v.end(), 42), v.end());  // remove all 42s
// C++20:
std::erase(v, 42);                                        // equivalent, cleaner
std::erase_if(v, [](int x){ return x % 2 == 0; });       // remove all evens

// Shrink to fit:
v.shrink_to_fit();  // release excess capacity (non-binding hint)

// Vector of booleans - SPECIAL CASE: packed bits, not array of bool
std::vector<bool> bv;  // this is weird! use vector<uint8_t> or bitset instead
```

### `std::array` — Stack-Allocated Fixed Size

```cpp
#include <array>

// Fixed size, no heap, full STL interface:
std::array<int, 5> arr = {1, 2, 3, 4, 5};

arr[0];             // no bounds check
arr.at(0);          // bounds checked
arr.size();         // 5 (compile-time constant)
arr.data();         // pointer to raw array

// Works with algorithms:
std::sort(arr.begin(), arr.end());

// Perfect for embedded: register sets, DMA descriptor arrays, etc.
std::array<volatile uint32_t*, 16> gpio_ports;
```

### `std::map` vs `std::unordered_map`

```cpp
#include <map>
#include <unordered_map>

// map: sorted, O(log n) ops, tree-based (many small allocations)
std::map<std::string, int> sorted_map;
sorted_map["uart"] = 115200;
sorted_map["spi"]  = 10000000;
// Iteration is in sorted key order

// unordered_map: O(1) avg, hash-based, more memory
std::unordered_map<std::string, int> hash_map;
hash_map["uart"] = 115200;
// Iteration order is undefined

// Operations common to both:
auto it = hash_map.find("uart");
if (it != hash_map.end()) {
    std::cout << it->second;  // 115200
}

// Insert:
hash_map.insert({"uart", 115200});      // insert pair
hash_map.emplace("uart", 115200);       // construct in-place
hash_map["new_key"] = 0;               // insert default then assign (creates entry!)
hash_map.insert_or_assign("k", 42);    // C++17: insert or update

// try_emplace (C++17): only construct if key doesn't exist
hash_map.try_emplace("key", 42);       // no construction if "key" already present

// In embedded: prefer std::array + linear search for small maps
// unordered_map has nondeterministic behavior (hash collisions)
```

---

## 3. Iterators & Ranges (C++20)

### Iterator Categories

```cpp
// From weakest to strongest:
// InputIterator    → single-pass read (e.g., istream_iterator)
// OutputIterator   → single-pass write (e.g., back_insert_iterator)
// ForwardIterator  → multi-pass read/write (e.g., list::iterator)
// BidirectionalIt  → forward + backward (e.g., map::iterator)
// RandomAccessIt   → bidirectional + O(1) jump (e.g., vector::iterator)
// ContiguousIt     → random access + contiguous memory (C++17, e.g., vector::iterator)

// Why categories matter:
// std::sort requires RandomAccess -> works on vector, not list
// std::advance is O(n) for bidirectional, O(1) for random access

std::vector<int> v = {5,3,1,4,2};
std::sort(v.begin(), v.end());          // OK: vector has random access

std::list<int> l = {5,3,1,4,2};
// std::sort(l.begin(), l.end());       // ERROR: list has bidirectional only
l.sort();                               // use list's own sort member
```

### Range-Based For and Iterators

```cpp
// Range-based for loop expands to:
for (auto& x : container) { /* ... */ }
// becomes:
auto __begin = container.begin();
auto __end   = container.end();
for (; __begin != __end; ++__begin) {
    auto& x = *__begin;
    /* ... */
}

// Your type works with range-for if it has begin()/end() members
// or free begin()/end() functions

// Iterator adapters:
#include <iterator>
std::vector<int> v;
std::copy(std::istream_iterator<int>(std::cin),
          std::istream_iterator<int>(),
          std::back_inserter(v));   // back_inserter: push_back into v

// Reverse iteration:
for (auto it = v.rbegin(); it != v.rend(); ++it) { /* reverse */ }
// or: range-for with std::views::reverse (C++20)
```

### C++20 Ranges

```cpp
#include <ranges>

std::vector<int> v = {1,2,3,4,5,6,7,8,9,10};

// Lazy views — no allocation, compose pipeline
auto even_squares =
    v | std::views::filter([](int x){ return x % 2 == 0; })
      | std::views::transform([](int x){ return x * x; });

for (int x : even_squares) {
    std::cout << x << " ";  // 4 16 36 64 100
}
// Nothing computed until iteration — lazy evaluation

// Take, drop:
auto first3 = v | std::views::take(3);   // {1,2,3}
auto skip2  = v | std::views::drop(2);   // {3,4,...,10}

// Algorithms with ranges (no begin/end boilerplate):
std::ranges::sort(v);                    // sort whole vector
std::ranges::sort(v, std::greater<>{}); // sort descending
auto it = std::ranges::find(v, 5);       // find 5

// Keys/values views for maps:
std::map<std::string, int> m;
for (auto& key : m | std::views::keys) { /* ... */ }
for (auto& val : m | std::views::values) { /* ... */ }
```

---

## 4. Standard Algorithms

```cpp
#include <algorithm>
#include <numeric>

std::vector<int> v = {3,1,4,1,5,9,2,6,5,3};

// Sorting:
std::sort(v.begin(), v.end());                      // ascending
std::sort(v.begin(), v.end(), std::greater<int>{});  // descending
std::stable_sort(v.begin(), v.end());               // preserves equal-element order
std::partial_sort(v.begin(), v.begin()+3, v.end()); // only smallest 3 sorted

// Searching:
bool found = std::binary_search(v.begin(), v.end(), 5);  // requires sorted
auto it = std::lower_bound(v.begin(), v.end(), 5);        // first element >= 5
auto it2 = std::upper_bound(v.begin(), v.end(), 5);       // first element > 5
auto it3 = std::find(v.begin(), v.end(), 5);              // linear search

// Counting/checking:
int count = std::count(v.begin(), v.end(), 5);            // how many 5s
bool any  = std::any_of(v.begin(), v.end(), [](int x){ return x > 8; });
bool all  = std::all_of(v.begin(), v.end(), [](int x){ return x > 0; });
bool none = std::none_of(v.begin(), v.end(), [](int x){ return x < 0; });

// Transformation:
std::transform(v.begin(), v.end(), v.begin(), [](int x){ return x*2; }); // in-place
std::vector<int> out(v.size());
std::transform(v.begin(), v.end(), out.begin(), [](int x){ return x*x; }); // to new

// Numeric:
int sum  = std::accumulate(v.begin(), v.end(), 0);        // sum
int prod = std::accumulate(v.begin(), v.end(), 1, std::multiplies<int>{});
std::iota(v.begin(), v.end(), 0);  // fill with 0,1,2,3,...
auto [min_it, max_it] = std::minmax_element(v.begin(), v.end());

// Copying/removing:
std::copy(src.begin(), src.end(), dst.begin());
std::fill(v.begin(), v.end(), 0);
std::remove(v.begin(), v.end(), 5);  // doesn't resize! moves non-5s to front

// Partitioning:
std::partition(v.begin(), v.end(), [](int x){ return x % 2 == 0; });
// All evens before all odds, relative order not preserved

// Permutations:
std::reverse(v.begin(), v.end());
std::rotate(v.begin(), v.begin()+2, v.end());  // rotate left by 2
std::shuffle(v.begin(), v.end(), std::mt19937{std::random_device{}()});
```

---

## 5. std::optional, std::variant, std::any (C++17)

### `std::optional<T>` — May or May Not Have a Value

```cpp
#include <optional>

// Functions that might fail without exceptions:
std::optional<int> parse_int(const std::string& s) {
    try { return std::stoi(s); }
    catch (...) { return std::nullopt; }  // no value
}

auto result = parse_int("42");
if (result) {                     // or: result.has_value()
    std::cout << *result;         // or: result.value()
}

// With default:
int val = result.value_or(0);     // 0 if no value

// In embedded — representing an optional sensor reading:
std::optional<float> read_temperature() {
    if (!sensor_initialized_) return std::nullopt;
    return raw_to_celsius(adc_read());
}

// Monadic operations (C++23, but pattern useful now):
auto doubled = result.transform([](int x){ return x * 2; });   // C++23
// Manual C++17:
auto doubled17 = result ? std::optional<int>(*result * 2) : std::nullopt;
```

### `std::variant<Ts...>` — Type-Safe Union

```cpp
#include <variant>

// Can hold exactly one of the listed types at a time
using Result = std::variant<int, float, std::string, std::error_code>;

Result r1 = 42;
Result r2 = 3.14f;
Result r3 = std::string("error");

// Check which type is active:
std::holds_alternative<int>(r1);   // true
r1.index();                         // 0 (index of int in the type list)

// Get the value:
int i = std::get<int>(r1);          // throws std::bad_variant_access if wrong type
int* ip = std::get_if<int>(&r1);    // returns pointer, null if wrong type (no throw)

// Visit: call a function based on active type
std::visit([](auto& val) {
    std::cout << val;
}, r1);

// Exhaustive visitor with overloaded lambdas (C++17 pattern):
template<typename... Ts>
struct Overload : Ts... { using Ts::operator()...; };
template<typename... Ts>
Overload(Ts...) -> Overload<Ts...>;  // deduction guide

std::visit(Overload{
    [](int i)               { std::cout << "int: " << i; },
    [](float f)             { std::cout << "float: " << f; },
    [](const std::string& s){ std::cout << "string: " << s; },
    [](std::error_code& ec) { std::cout << "error: " << ec.message(); }
}, r1);

// Variant for state machine (no heap, type-safe):
using UartState = std::variant<
    struct Idle{},
    struct Transmitting{ size_t bytes_left; },
    struct Receiving{ uint8_t* buf; size_t len; },
    struct Error{ int code; }
>;

UartState state = Idle{};
state = Transmitting{1024};
```

### `std::any` — Type-Erased Container

```cpp
#include <any>

std::any a = 42;
a = std::string("hello");  // can change type
a = 3.14;

// Access (throws std::bad_any_cast if wrong type):
double d = std::any_cast<double>(a);

// Check type:
a.type() == typeid(double);  // true

// Avoid std::any in embedded — uses heap, no compile-time type safety
// Prefer std::variant when you know the set of types
```

---

## 6. std::string_view and std::span (C++17/20)

### `std::string_view` — Non-Owning String Reference

```cpp
#include <string_view>

// Does NOT own the string. Just a pointer + length.
// Zero allocation, zero copy.

void process(std::string_view sv) {    // accepts any string-like thing
    sv.size();
    sv.substr(0, 5);                   // returns another view (no allocation!)
    sv.find("hello");
    sv[0];
    sv.starts_with("hello");           // C++20
}

std::string s = "hello world";
process(s);               // from std::string
process("hello world");   // from string literal
process(s.substr(0, 5));  // from substring (but temp! be careful)

// DANGER: string_view can dangle!
std::string_view dangerous() {
    std::string local = "hello";
    return local;  // UB! string_view points to destroyed local
}

// SAFE uses:
// - Function parameters
// - Returning a view of data the caller owns
// - Avoid storing string_view as class member unless lifetime is clear
```

### `std::span<T>` — Non-Owning Contiguous View (C++20)

```cpp
#include <span>

// Like string_view but for any contiguous data of type T
// No allocation, just pointer + size

void transmit(std::span<const uint8_t> data) {
    // works with any contiguous container!
    for (uint8_t byte : data) {
        uart_write(byte);
    }
}

uint8_t buf[64];
std::vector<uint8_t> vec = {1,2,3};
std::array<uint8_t, 4> arr = {0xAA, 0xBB, 0xCC, 0xDD};

transmit(buf);    // from C array
transmit(vec);    // from vector
transmit(arr);    // from array

// Subspans:
std::span<uint8_t> sp(buf, 64);
auto first_half = sp.first(32);    // span of first 32 bytes
auto last_half  = sp.last(32);     // span of last 32 bytes
auto middle     = sp.subspan(16, 32);  // 32 bytes starting at offset 16

// Static extent (size known at compile time):
std::span<uint8_t, 4> fixed_sp(arr);  // exactly 4 bytes, size_t is compile-time constant
```

---

## 7. Lambdas — Deep Dive

### Lambda Syntax and Captures

```cpp
// Syntax: [capture](params) specifiers -> return_type { body }

// No capture:
auto square = [](int x) { return x * x; };
square(5);  // 25

// Capture by value (copies at lambda creation):
int offset = 10;
auto add_offset = [offset](int x) { return x + offset; };
offset = 99;          // doesn't affect add_offset: offset was copied
add_offset(5);        // 15 (uses the copied value 10)

// Capture by reference (dangerous if lambda outlives variable):
auto add_offset_ref = [&offset](int x) { return x + offset; };
offset = 99;
add_offset_ref(5);    // 104 (uses current value of offset)

// Capture all by value:  [=]
// Capture all by ref:    [&]
// Mix:                   [=, &specific_var]
// Named captures:        [x=std::move(unique_ptr)] (C++14 init capture)

// Mutable lambda (modify captured-by-value variables):
int count = 0;
auto counter = [count]() mutable { return ++count; };
counter();  // 1
counter();  // 2
count;      // still 0! (lambda has its own copy)

// C++14 generic lambda (auto parameters):
auto print_any = [](const auto& val) { std::cout << val; };
print_any(42);
print_any("hello");
print_any(3.14);
```

### Lambdas as Callbacks and State Machines

```cpp
// Storing state in a lambda — useful for one-off event handlers:
auto make_counter(int start) {
    return [n = start]() mutable { return n++; };
}
auto c = make_counter(5);
c();  // 5
c();  // 6
c();  // 7

// Lambda for comparators:
std::sort(v.begin(), v.end(), [](const Sensor& a, const Sensor& b) {
    return a.id() < b.id();
});

// Immediately invoked lambda (IIFE: for complex const initialization):
const auto config = [&]() {
    Config c;
    if (use_uart) c.set_uart(115200);
    if (use_spi)  c.set_spi(10'000'000);
    return c;
}();  // immediately invoked

// Recursive lambda (C++14+):
auto fib = [](auto self, int n) -> int {
    if (n <= 1) return n;
    return self(self, n-1) + self(self, n-2);
};
fib(fib, 10);  // 55
```

### Lambda Internals

A lambda is syntactic sugar for a compiler-generated struct with `operator()`:

```cpp
int multiplier = 3;
auto times_n = [multiplier](int x) { return x * multiplier; };

// Equivalent compiler-generated class:
struct __lambda_1 {
    int multiplier;  // captured member
    __lambda_1(int m) : multiplier(m) {}
    int operator()(int x) const { return x * multiplier; }
};
auto times_n_manual = __lambda_1(multiplier);
```

---

## 8. std::function and Callable Types

```cpp
#include <functional>

// std::function: type-erased callable (stores any callable with matching signature)
std::function<int(int, int)> op;

op = [](int a, int b) { return a + b; };  // lambda
op(3, 4);  // 7

// Free function:
int multiply(int a, int b) { return a * b; }
op = multiply;

// Member function via bind or lambda:
struct Calculator { int add(int a, int b) { return a+b; } };
Calculator calc;
op = [&calc](int a, int b) { return calc.add(a, b); };
// or:
op = std::bind(&Calculator::add, &calc, std::placeholders::_1, std::placeholders::_2);

// Cost: std::function has overhead:
// - Heap allocation for large callables (small buffer optimization varies)
// - Virtual dispatch internally
// - Cannot be used as a constexpr

// In performance-critical embedded code, prefer:
// 1. Template parameter (policy-based: zero overhead)
template<typename Callback>
void on_receive(Callback cb) { cb(data); }

// 2. Function pointer (if you need runtime selection but don't need captures):
void (*fp)(int) = [](int x){ process(x); };  // only works for non-capturing lambdas!

// 3. std::function only when you need type erasure with captures + runtime selection
```

---

## 9. Threading Model — std::thread

```cpp
#include <thread>

// Create a thread:
std::thread t([]{
    std::cout << "thread " << std::this_thread::get_id() << "\n";
});
t.join();   // wait for thread to finish (must call join or detach!)
// Forgetting join/detach: std::terminate() called in destructor

// Thread with arguments:
void task(int id, std::string name) { /* ... */ }
std::thread t2(task, 42, "uart_thread");
t2.join();

// Detached thread: runs independently until function returns
std::thread t3(background_logger);
t3.detach();  // fire and forget

// Thread ID:
std::thread::id main_id = std::this_thread::get_id();

// Hardware concurrency (number of logical CPUs):
unsigned n = std::thread::hardware_concurrency();

// Sleep:
std::this_thread::sleep_for(std::chrono::milliseconds(100));
std::this_thread::sleep_until(std::chrono::steady_clock::now() + std::chrono::seconds(1));

// Yield:
std::this_thread::yield();  // hint to scheduler to switch away
```

### RAII Thread Wrapper

```cpp
// Problem: std::thread terminates on destruction if not joined
// Solution: RAII wrapper that joins on destruction

class JoiningThread {
    std::thread t_;
public:
    template<typename F, typename... Args>
    explicit JoiningThread(F&& f, Args&&... args)
        : t_(std::forward<F>(f), std::forward<Args>(args)...) {}

    ~JoiningThread() {
        if (t_.joinable()) t_.join();
    }

    JoiningThread(const JoiningThread&) = delete;
    JoiningThread& operator=(const JoiningThread&) = delete;
    JoiningThread(JoiningThread&&) = default;
};
// C++20: std::jthread does this + supports cooperative cancellation
```

---

## 10. Mutexes, Locks & Condition Variables

### Mutex Types

```cpp
#include <mutex>
#include <shared_mutex>

// Basic mutex: one thread at a time
std::mutex mtx;
mtx.lock();
mtx.unlock();
mtx.try_lock();  // returns false immediately if locked

// Recursive mutex: same thread can lock multiple times
std::recursive_mutex rec_mtx;

// Timed mutex: try_lock_for / try_lock_until
std::timed_mutex timed_mtx;
timed_mtx.try_lock_for(std::chrono::milliseconds(100));

// Shared mutex (C++17): readers-writer lock
std::shared_mutex sh_mtx;
// Multiple readers at once OR one writer:
sh_mtx.lock_shared();    // readers acquire shared lock
sh_mtx.unlock_shared();
sh_mtx.lock();           // writer acquires exclusive lock
sh_mtx.unlock();
```

### RAII Lock Guards

```cpp
// lock_guard: simple RAII wrapper, no unlock method
{
    std::lock_guard<std::mutex> guard(mtx);  // locks
    // critical section
}  // unlocks here

// C++17 CTAD:
std::lock_guard guard(mtx);

// unique_lock: more flexible — can unlock early, try_lock, timed lock
{
    std::unique_lock<std::mutex> lk(mtx);
    // critical section
    lk.unlock();    // early unlock possible
    // ... non-critical work ...
    lk.lock();      // re-lock
}

// shared_lock (C++17): RAII for shared_mutex read lock
{
    std::shared_lock<std::shared_mutex> read_lk(sh_mtx);  // shared (read) lock
    // multiple threads can hold this simultaneously
}

// scoped_lock (C++17): lock multiple mutexes deadlock-free
std::scoped_lock lock(mtx1, mtx2, mtx3);  // all or nothing, deadlock-safe order

// Avoiding deadlock manually:
// Always acquire locks in the same order across all threads
// Or use std::lock() then adopt:
std::lock(mtx1, mtx2);                      // deadlock-free: tries both, backs off if needed
std::lock_guard<std::mutex> lg1(mtx1, std::adopt_lock);  // RAII-adopt already-locked mutex
std::lock_guard<std::mutex> lg2(mtx2, std::adopt_lock);
```

### Condition Variables

```cpp
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;
bool data_ready = false;
std::queue<int> data_queue;

// Producer thread:
void producer() {
    for (int i = 0; i < 10; i++) {
        {
            std::lock_guard lock(mtx);
            data_queue.push(i);
            data_ready = true;
        }  // unlock before notify (avoid waking consumer while holding lock)
        cv.notify_one();   // wake one waiting thread
    }
}

// Consumer thread:
void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lk(mtx);  // unique_lock needed for cv.wait()
        cv.wait(lk, []{ return data_ready || finished; });
        // wait: atomically releases lock and blocks
        //       when notified: reacquires lock and checks predicate
        //       if predicate false: spurious wakeup protection -> wait again

        if (!data_queue.empty()) {
            int val = data_queue.front();
            data_queue.pop();
        }
    }
}

// notify_one: wake one waiting thread
// notify_all: wake all waiting threads
// Always use a predicate with wait() — protects against spurious wakeups
```

---

## 11. std::atomic — The Complete Model

### Basic Atomic Operations

```cpp
#include <atomic>

// Atomic types: all operations are guaranteed to be thread-safe
std::atomic<int>      counter{0};
std::atomic<bool>     flag{false};
std::atomic<uint32_t> reg_val{0};

// Load and store:
int val = counter.load();                         // atomic read
counter.store(42);                                // atomic write
int old = counter.exchange(99);                   // atomic swap: returns old value

// Compare-exchange: the CAS (Compare-And-Swap) primitive
int expected = 0;
bool success = counter.compare_exchange_strong(expected, 1);
// If counter == expected: set counter = 1, return true
// If counter != expected: set expected = counter's current value, return false

// Weak version (may spuriously fail, faster on some architectures):
counter.compare_exchange_weak(expected, 1);  // use in loop

// Arithmetic:
counter.fetch_add(1);          // atomic increment, returns OLD value
counter++;                     // same as fetch_add(1)
counter.fetch_sub(1);          // atomic decrement
counter.fetch_and(0xFF);       // bitwise AND
counter.fetch_or(0x01);        // bitwise OR
counter.fetch_xor(0xF0);       // bitwise XOR
```

### `std::atomic_flag` — The Lock-Free Primitive

```cpp
// atomic_flag: guaranteed lock-free (the ONLY atomic guaranteed lock-free by standard)
std::atomic_flag flag = ATOMIC_FLAG_INIT;  // must initialize with this macro

// test_and_set: sets flag to true, returns OLD value
bool was_set = flag.test_and_set();  // if was false: returns false (we got the flag)

// clear: set to false
flag.clear();

// Spinlock using atomic_flag:
class Spinlock {
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
public:
    void lock() {
        while (flag_.test_and_set(std::memory_order_acquire))
            ;  // spin: busy wait
    }
    void unlock() {
        flag_.clear(std::memory_order_release);
    }
};

// Use sparingly: wastes CPU. Only for very short critical sections.
```

---

## 12. Memory Ordering

This is the most complex topic in C++ concurrency. Memory ordering controls which memory operations other threads observe and in what order.

### The Six Memory Orders

```cpp
// seq_cst: Sequentially consistent — strongest guarantee
// All operations appear in a single total order across all threads
std::atomic<int> x;
x.store(1, std::memory_order_seq_cst);  // default if not specified
x.load(std::memory_order_seq_cst);

// acquire/release: synchronize specific producer-consumer pairs
// release (store): all writes before this point are visible to threads that acquire
// acquire (load): all writes by the releasing thread are visible here

std::atomic<bool> ready{false};
int data = 0;  // non-atomic!

// Producer thread:
data = 42;                                      // write to non-atomic data
ready.store(true, std::memory_order_release);   // release: data write is "published"

// Consumer thread:
while (!ready.load(std::memory_order_acquire)); // acquire: establishes happens-before
// Now safe to read data: happens-before relationship established
assert(data == 42);  // guaranteed to see 42

// relaxed: no ordering guarantees — just atomicity
// Use for counters where you only care about the final value:
std::atomic<size_t> event_count{0};
event_count.fetch_add(1, std::memory_order_relaxed);  // no synchronization needed
// Reading the count:
size_t n = event_count.load(std::memory_order_relaxed); // just read the value

// acq_rel: for read-modify-write operations (compare_exchange, fetch_add used as sync)
```

### Practical Memory Ordering Guide

```cpp
// 99% of correct concurrent code uses only these patterns:

// Pattern 1: Flag + data (publish-subscribe)
std::atomic<bool> flag{false};
std::string data;

// Producer:
data = "hello";
flag.store(true, std::memory_order_release);  // RELEASE after writing data

// Consumer:
while (!flag.load(std::memory_order_acquire));  // ACQUIRE before reading data
std::cout << data;  // guaranteed to see "hello"

// Pattern 2: Shared counter (no inter-thread data dependency)
std::atomic<int> hits{0};
// anywhere:
hits.fetch_add(1, std::memory_order_relaxed);  // just counting, no sync needed

// Pattern 3: Mutex-protected data (not atomic data)
std::mutex mtx;
int shared_data = 0;
// mutex provides all necessary ordering — no atomic needed for the protected data

// Pattern 4: Lock-free queue with seq_cst
// Use seq_cst when you're not sure — it's always correct (just possibly slower)
```

---

## 13. std::future, std::promise & async

```cpp
#include <future>

// async: run a function asynchronously, get result via future
std::future<int> result = std::async(std::launch::async, []() {
    // runs in a new thread
    return expensive_computation();
});

// Do other work while computation runs:
do_other_stuff();

// Get result (blocks until ready):
int val = result.get();   // blocks if not yet done; throws if exception occurred in task

// Launch policies:
std::async(std::launch::async, f);    // definitely new thread
std::async(std::launch::deferred, f); // run lazily on get()
std::async(std::launch::async | std::launch::deferred, f);  // implementation decides

// promise: explicit producer-consumer
std::promise<int> prom;
std::future<int>  fut = prom.get_future();

// In another thread:
std::thread producer([&prom]() {
    int result = compute();
    prom.set_value(result);        // fulfill the promise
    // or on error: prom.set_exception(std::current_exception());
});

int result = fut.get();  // blocks until producer calls set_value
producer.join();

// shared_future: multiple threads can wait for the same result
std::shared_future<int> sf = prom.get_future().share();
// Multiple threads can call sf.get()
```

---

## 14. Lock-Free Data Structures

### Single-Producer Single-Consumer (SPSC) Ring Buffer

The most important lock-free structure for embedded Linux (ISR → task, producer → consumer):

```cpp
#include <atomic>
#include <array>

template<typename T, size_t N>
class SpscQueue {
    static_assert((N & (N-1)) == 0, "N must be power of 2");

    std::array<T, N> buf_;
    std::atomic<size_t> head_{0};  // consumer reads from head
    std::atomic<size_t> tail_{0};  // producer writes to tail

public:
    // Producer (single thread only):
    bool push(const T& val) {
        const size_t t = tail_.load(std::memory_order_relaxed);
        const size_t next_t = (t + 1) & (N - 1);  // power-of-2 wrapping

        if (next_t == head_.load(std::memory_order_acquire))
            return false;  // full

        buf_[t] = val;
        tail_.store(next_t, std::memory_order_release);
        return true;
    }

    // Consumer (single thread only):
    bool pop(T& val) {
        const size_t h = head_.load(std::memory_order_relaxed);

        if (h == tail_.load(std::memory_order_acquire))
            return false;  // empty

        val = buf_[h];
        head_.store((h + 1) & (N - 1), std::memory_order_release);
        return true;
    }

    bool empty() const {
        return head_.load(std::memory_order_acquire) ==
               tail_.load(std::memory_order_acquire);
    }
};

// Usage: UART RX ISR → processing task
SpscQueue<uint8_t, 256> uart_rx_queue;

// ISR (producer):
void UART_IRQ_Handler() {
    uint8_t byte = UART_DR;
    uart_rx_queue.push(byte);  // lock-free, ISR-safe
}

// Task (consumer):
void uart_task() {
    uint8_t byte;
    while (uart_rx_queue.pop(byte)) {
        process(byte);
    }
}
```

---

## 15. Exception Handling

### try/catch/throw

```cpp
#include <stdexcept>

// Throwing:
void check_range(int val, int max) {
    if (val >= max)
        throw std::out_of_range("value exceeds maximum");
    if (val < 0)
        throw std::invalid_argument("negative value");
}

// Catching:
try {
    check_range(100, 50);
} catch (const std::out_of_range& e) {
    std::cerr << "range error: " << e.what();
} catch (const std::exception& e) {
    std::cerr << "standard exception: " << e.what();
} catch (...) {
    std::cerr << "unknown exception";
}

// Re-throwing:
catch (const std::exception& e) {
    log_error(e.what());
    throw;  // re-throw the same exception (not: throw e; which may slice)
}
```

### Custom Exceptions

```cpp
class HardwareError : public std::runtime_error {
    int error_code_;
public:
    HardwareError(int code, const std::string& msg)
        : std::runtime_error(msg), error_code_(code) {}

    int code() const noexcept { return error_code_; }
};

// Exception hierarchy: more specific first, more general later
try {
    init_hardware();
} catch (const HardwareError& e) {
    // handles HardwareError
} catch (const std::runtime_error& e) {
    // handles other runtime errors
} catch (const std::exception& e) {
    // handles all standard exceptions
}
```

### `noexcept` Specification

```cpp
// Declare that a function never throws:
void cleanup() noexcept { /* ... */ }

// Conditional noexcept (C++11):
template<typename T>
void swap(T& a, T& b) noexcept(std::is_nothrow_swappable_v<T>) {
    T tmp = std::move(a);
    a = std::move(b);
    b = std::move(tmp);
}

// Query at compile time:
bool ne = noexcept(cleanup());   // true

// Rules:
// - Destructors are implicitly noexcept (must not throw!)
// - Move constructors/assignments: mark noexcept for performance
// - If noexcept function throws: std::terminate() called immediately
```

### Exceptions in Embedded

Many embedded environments disable exceptions (`-fno-exceptions`) for:
- Binary size reduction (exception tables take significant space)
- No dynamic allocation in exception infrastructure
- Deterministic timing (stack unwinding has variable cost)

Without exceptions, use alternatives:
```cpp
// std::expected<T, E> (C++23) or similar pattern:
template<typename T, typename E>
class Expected {
    union { T value_; E error_; };
    bool has_value_;
public:
    static Expected ok(T val)  { return Expected(std::move(val), true); }
    static Expected err(E err) { return Expected(std::move(err), false); }
    bool  has_value() const  { return has_value_; }
    T&    value()            { return value_; }
    E&    error()            { return error_; }
    explicit operator bool() { return has_value_; }
};

Expected<int, ErrCode> open_device(const char* path) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) return Expected<int, ErrCode>::err(ErrCode::NotFound);
    return Expected<int, ErrCode>::ok(fd);
}

auto result = open_device("/dev/uart0");
if (!result) {
    handle_error(result.error());
} else {
    use_fd(result.value());
}
```

---

## Key Interview Q&A Summary

**Q: When does a shared_ptr cause a memory leak?**
A: Cyclic references — A holds shared_ptr to B, B holds shared_ptr to A. Neither ref count drops to zero. Fix with weak_ptr for back-pointers.

**Q: What's the difference between lock_guard and unique_lock?**
A: `lock_guard` is simpler — locks in constructor, unlocks in destructor, cannot unlock early. `unique_lock` is flexible — can unlock/relock, try_lock, move ownership. Required for condition variable usage.

**Q: What is a spurious wakeup and how do you protect against it?**
A: A thread waiting on a condition variable may wake up even when the condition hasn't been set — spurious wakeup. Always use `cv.wait(lock, predicate)` — the predicate is rechecked and if false, waits again.

**Q: Why is memory_order_relaxed safe for a counter but not for publishing data?**
A: Relaxed guarantees atomicity of the operation but no ordering relative to other memory accesses. For a counter you only care about the final value — no other data depends on this specific operation. For publishing data, you need release/acquire to ensure the data writes are visible before the flag is set.

**Q: What makes std::atomic_flag the only truly lock-free atomic?**
A: The C++ standard only guarantees that `atomic_flag` operations are always lock-free. For `atomic<T>`, the implementation may use locks if the hardware can't do atomic operations on T natively. Check with `atomic<T>::is_lock_free()`.

---

*Document 3 of 5 — Continue to Document 4: Modern C++ Features C++11 through C++20*
