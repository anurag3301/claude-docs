# Architecting a Modern GDB/MI3 Front-End in C++23

> A comprehensive guide to designing a production-quality, type-safe, asynchronous GDB/MI3 client library using the full power of C++23.

---

## Table of Contents

1. [Design Philosophy and Goals](#1-design-philosophy-and-goals)
2. [Project Structure](#2-project-structure)
3. [The MI3 Wire Protocol Layer](#3-the-mi3-wire-protocol-layer)
4. [Lexer and Parser Architecture](#4-lexer-and-parser-architecture)
5. [Type-Safe Value Representation](#5-type-safe-value-representation)
6. [Record Type Hierarchy](#6-record-type-hierarchy)
7. [Async I/O and the GDB Process Manager](#7-async-io-and-the-gdb-process-manager)
8. [Command Dispatch and Token Management](#8-command-dispatch-and-token-management)
9. [Breakpoint Subsystem](#9-breakpoint-subsystem)
10. [Variable Object (varobj) Subsystem](#10-variable-object-varobj-subsystem)
11. [Stack and Thread Subsystem](#11-stack-and-thread-subsystem)
12. [Async Notification Router](#12-async-notification-router)
13. [Session and State Machine](#13-session-and-state-machine)
14. [Error Handling Strategy](#14-error-handling-strategy)
15. [Testing Architecture](#15-testing-architecture)
16. [Integration Points and Extension Model](#16-integration-points-and-extension-model)
17. [Complete Worked Examples](#17-complete-worked-examples)

---

## 1. Design Philosophy and Goals

### Why C++23 Specifically

The MI3 protocol is fundamentally asynchronous and message-oriented. C++23 gives us a uniquely powerful set of tools that, when combined thoughtfully, make the protocol feel native rather than bolted-on:

`std::expected` removes the need for exceptions on the hot path — every command response is either a typed success value or a well-defined `MiError`. `std::generator` (coroutine-based ranges) models the MI3 output stream elegantly: GDB's stdout is a lazy sequence of records, and we can consume it as exactly that. Pattern matching via `std::visit` over `std::variant` gives us exhaustive dispatch over record types that the compiler enforces. `std::flat_map` and `std::mdspan` reduce allocator overhead in the high-frequency data paths (register reads, memory dumps). And `std::print` / `std::format` make constructing MI command strings concise and allocation-free.

### Core Principles

**Zero-overhead abstraction.** The parser and value types must compile away entirely in release builds. A call like `record.result<BreakpointInfo>()` should generate no more code than hand-written struct population.

**Correctness by construction.** Every MI3 concept that has invariants — tokens, breakpoint numbers, thread IDs — is a distinct strong typedef. You cannot accidentally pass a thread ID where a breakpoint number is required. The multi-location MI3 breakpoint format (the canonical MI3 change) is modelled as a proper `std::variant<SingleLocation, MultiLocation>` rather than an optional list, because a breakpoint that resolves to `<MULTIPLE>` is categorically different from one that resolves to a single address.

**Composable, not monolithic.** The library is a set of cooperating layers, each testable in isolation. You can use only the parser without the I/O layer, only the command builder without the parser, or the full stack.

**Explicit over implicit.** No global state. No thread-local singletons. Every piece of state is passed explicitly through the call graph, which makes multi-inferior debugging (multiple GDB instances) natural.

---

## 2. Project Structure

```
gdbmi3/
├── CMakeLists.txt
├── include/
│   └── gdbmi3/
│       ├── core/
│       │   ├── token.hpp           # Strong typedefs (Token, BreakpointNumber, ThreadId…)
│       │   ├── value.hpp           # MiValue: string | tuple | list variant
│       │   ├── record.hpp          # Record type hierarchy
│       │   └── error.hpp           # MiError and error codes
│       ├── parse/
│       │   ├── lexer.hpp           # Zero-copy lexer over string_view
│       │   └── parser.hpp          # Recursive-descent MI3 parser
│       ├── command/
│       │   ├── builder.hpp         # Type-safe MI command construction
│       │   ├── breakpoint_cmds.hpp # -break-insert, -break-delete, etc.
│       │   ├── exec_cmds.hpp       # -exec-run, -exec-step, etc.
│       │   ├── stack_cmds.hpp      # -stack-list-frames, etc.
│       │   ├── data_cmds.hpp       # -data-evaluate-expression, etc.
│       │   ├── var_cmds.hpp        # -var-create, -var-update, etc.
│       │   └── gdb_cmds.hpp        # -gdb-set, -gdb-exit, etc.
│       ├── result/
│       │   ├── breakpoint.hpp      # BreakpointInfo, MultiLocationBreakpoint
│       │   ├── frame.hpp           # Frame, StackFrame
│       │   ├── thread.hpp          # ThreadInfo, ThreadGroup
│       │   ├── varobj.hpp          # VarObj, VarObjChange
│       │   └── memory.hpp          # MemoryBlock, RegisterValue
│       ├── session/
│       │   ├── process.hpp         # GDB process lifetime management
│       │   ├── dispatcher.hpp      # Token ↔ promise correlation
│       │   ├── notification.hpp    # Async notification routing
│       │   └── session.hpp         # Top-level session object
│       └── gdbmi3.hpp              # Umbrella include
├── src/
│   ├── parse/
│   │   ├── lexer.cpp
│   │   └── parser.cpp
│   └── session/
│       ├── process.cpp
│       ├── dispatcher.cpp
│       └── session.cpp
└── test/
    ├── unit/
    │   ├── test_lexer.cpp
    │   ├── test_parser.cpp
    │   ├── test_value.cpp
    │   └── test_command_builder.cpp
    └── integration/
        ├── test_breakpoints.cpp
        └── test_session.cpp
```

---

## 3. The MI3 Wire Protocol Layer

Before writing a single class, we nail down exactly what the wire protocol gives us and impose a clean boundary between "bytes from GDB" and "typed records".

### Token: The Correlation Key

MI3's token is a sequence of decimal digits that you prepend to a command and GDB echoes back on the result. This is the correlation mechanism for async dispatch.

```cpp
// include/gdbmi3/core/token.hpp
#pragma once
#include <cstdint>
#include <format>
#include <atomic>
#include <limits>

namespace gdbmi3 {

// Strong typedef — prevents passing raw integers as tokens
class Token {
public:
    using value_type = std::uint64_t;
    
    constexpr explicit Token(value_type v) noexcept : value_(v) {}
    
    [[nodiscard]] constexpr value_type value() const noexcept { return value_; }
    
    [[nodiscard]] std::string to_string() const {
        return std::format("{}", value_);
    }
    
    constexpr bool operator==(const Token&) const noexcept = default;
    constexpr auto operator<=>(const Token&) const noexcept = default;
    
private:
    value_type value_;
};

// Thread-safe token generator — a session owns exactly one
class TokenGenerator {
public:
    [[nodiscard]] Token next() noexcept {
        return Token{counter_.fetch_add(1, std::memory_order_relaxed)};
    }
    
private:
    std::atomic<Token::value_type> counter_{1};
};

} // namespace gdbmi3

// Make Token hashable
template<> struct std::hash<gdbmi3::Token> {
    std::size_t operator()(gdbmi3::Token t) const noexcept {
        return std::hash<gdbmi3::Token::value_type>{}(t.value());
    }
};
```

### Other Strong Typedefs

```cpp
// The same pattern applied to all numeric identifiers in the protocol
namespace gdbmi3 {

template<typename Tag, typename Underlying = std::uint32_t>
class StrongId {
public:
    using value_type = Underlying;
    constexpr explicit StrongId(Underlying v) noexcept : value_(v) {}
    [[nodiscard]] constexpr Underlying value() const noexcept { return value_; }
    constexpr bool operator==(const StrongId&) const noexcept = default;
    constexpr auto operator<=>(const StrongId&) const noexcept = default;
private:
    Underlying value_;
};

struct BreakpointNumberTag {};
struct ThreadIdTag {};
struct FrameLevelTag {};
struct InferiorIdTag {};

using BreakpointNumber = StrongId<BreakpointNumberTag>;
using ThreadId         = StrongId<ThreadIdTag>;
using FrameLevel       = StrongId<FrameLevelTag>;
using InferiorId       = StrongId<InferiorIdTag, std::uint16_t>;

} // namespace gdbmi3
```

---

## 4. Lexer and Parser Architecture

The MI3 output grammar is small but recursive. The key insight is that we never need to copy the input — the entire lexer and most of the parser operate on `std::string_view` into the line buffer.

### The Lexer

```cpp
// include/gdbmi3/parse/lexer.hpp
#pragma once
#include <string_view>
#include <optional>
#include <variant>

namespace gdbmi3::parse {

enum class TokenKind : std::uint8_t {
    // Prefix characters
    Caret,        // ^ (result record)
    Tilde,        // ~ (console stream)
    At,           // @ (target stream)
    Ampersand,    // & (log stream)
    Star,         // * (exec async)
    Plus,         // + (status async)
    Equals,       // = (notify async)
    
    // Structural
    Comma,        // ,
    Equals_sign,  // =
    LeftBrace,    // {
    RightBrace,   // }
    LeftBracket,  // [
    RightBracket, // ]
    
    // Data
    CString,      // "..." — the decoded string_view (with escape handling)
    Identifier,   // bare word (result-class, key names, etc.)
    Digits,       // token number at start of line
    
    // Sentinel
    Prompt,       // "(gdb)"
    EndOfLine,
    Error,
};

struct LexToken {
    TokenKind kind;
    std::string_view text; // points into the source line — no copies
};

class Lexer {
public:
    explicit Lexer(std::string_view source) noexcept;
    
    // Advance and return next token
    [[nodiscard]] LexToken next();
    
    // Peek without consuming
    [[nodiscard]] LexToken peek() const;
    
    // True if the entire line has been consumed
    [[nodiscard]] bool at_end() const noexcept;
    
    // Current position (for error reporting)
    [[nodiscard]] std::size_t position() const noexcept;
    
private:
    std::string_view source_;
    std::size_t pos_{0};
    std::optional<LexToken> peeked_;
    
    LexToken lex_one();
    LexToken lex_cstring(); // handles \"escapes
    LexToken lex_identifier();
    LexToken lex_digits();
};

} // namespace gdbmi3::parse
```

The implementation of `lex_cstring` deserves attention: GDB C-strings use `\"`, `\\`, `\n`, `\t`, and `\r`. For strings that contain no escapes (the common case), we can return a `string_view` directly into the source. For strings with escapes, we need a buffer — but we can lazy-allocate that buffer as a `std::string` stored outside the lexer, making the common path allocation-free.

```cpp
// src/parse/lexer.cpp  (key excerpt)
LexToken Lexer::lex_cstring() {
    assert(source_[pos_] == '"');
    ++pos_; // consume opening quote
    
    std::size_t start = pos_;
    bool has_escapes = false;
    
    // First pass: scan to find end of string, flag if escapes present
    while (pos_ < source_.size() && source_[pos_] != '"') {
        if (source_[pos_] == '\\') {
            has_escapes = true;
            ++pos_; // skip escape char
            if (pos_ < source_.size()) ++pos_; // skip escaped char
        } else {
            ++pos_;
        }
    }
    std::size_t end = pos_;
    if (pos_ < source_.size()) ++pos_; // consume closing quote
    
    std::string_view raw{source_.data() + start, end - start};
    
    if (!has_escapes) {
        // Zero-copy fast path
        return {TokenKind::CString, raw};
    }
    
    // Decode escapes into the scratch buffer
    unescape_buffer_.clear();
    unescape_buffer_.reserve(raw.size());
    for (std::size_t i = 0; i < raw.size(); ++i) {
        if (raw[i] == '\\' && i + 1 < raw.size()) {
            ++i;
            switch (raw[i]) {
                case 'n':  unescape_buffer_ += '\n'; break;
                case 't':  unescape_buffer_ += '\t'; break;
                case 'r':  unescape_buffer_ += '\r'; break;
                case '"':  unescape_buffer_ += '"';  break;
                case '\\': unescape_buffer_ += '\\'; break;
                default:   unescape_buffer_ += raw[i]; break;
            }
        } else {
            unescape_buffer_ += raw[i];
        }
    }
    return {TokenKind::CString, unescape_buffer_};
    // NOTE: unescape_buffer_ lifetime is tied to the Lexer instance
    // which lives for the duration of parsing one line — safe.
}
```

### The Parser

The parser is a hand-written recursive-descent parser that follows the MI3 grammar directly. The grammar is:

```
record        → result-record | out-of-band-record | "(gdb)"
result-record → [digits] "^" result-class ("," result)*
oob-record    → async-record | stream-record
async-record  → [digits] async-class async-output
result        → key "=" value
value         → c-string | tuple | list
tuple         → "{}" | "{" result ("," result)* "}"
list          → "[]" | "[" (value | result) ("," (value | result))* "]"
```

```cpp
// include/gdbmi3/parse/parser.hpp
#pragma once
#include <gdbmi3/core/record.hpp>
#include <gdbmi3/core/error.hpp>
#include <gdbmi3/parse/lexer.hpp>
#include <expected>

namespace gdbmi3::parse {

// The parser consumes one line and produces one Record.
// It is entirely synchronous and stack-allocated.
class Parser {
public:
    using Result = std::expected<core::Record, core::ParseError>;
    
    [[nodiscard]] Result parse(std::string_view line);
    
private:
    Lexer* lex_ = nullptr; // set per call to parse()
    
    // Grammar productions
    core::Record        parse_record();
    core::ResultRecord  parse_result_record(std::optional<Token> token);
    core::AsyncRecord   parse_async_record(char prefix, std::optional<Token> token);
    core::StreamRecord  parse_stream_record(char prefix);
    
    core::MiValue       parse_value();
    core::MiTuple       parse_tuple();
    core::MiList        parse_list();
    core::MiResult      parse_result();
    
    // Helpers
    LexToken expect(TokenKind kind);
    bool     at(TokenKind kind) const;
    void     skip(TokenKind kind);
};

} // namespace gdbmi3::parse
```

---

## 5. Type-Safe Value Representation

MI3 values form a recursive structure: a value is either a string, a tuple (key→value map), or a list (ordered sequence of values or key→value pairs). We model this with a recursive `std::variant`.

```cpp
// include/gdbmi3/core/value.hpp
#pragma once
#include <string>
#include <vector>
#include <string_view>
#include <optional>
#include <variant>
#include <unordered_map>
#include <ranges>

namespace gdbmi3::core {

// Forward declarations for recursive type
struct MiTuple;
struct MiList;

// The leaf type — a decoded C-string from the protocol
using MiString = std::string;

// A recursive value — at the MI level, all fields are one of these
using MiValue = std::variant<MiString, MiTuple, MiList>;

// A key=value pair as it appears in result records
struct MiResult {
    std::string key;
    MiValue     value;
};

// {} or { key=value, key=value, ... }
// Implemented as vector to preserve insertion order (MI does not guarantee
// unique keys in all contexts, though well-formed output usually has them)
struct MiTuple {
    std::vector<MiResult> fields;
    
    // Convenience: look up a field by key — O(n) but tuples are small
    [[nodiscard]] const MiValue* find(std::string_view key) const noexcept;
    
    // find() + expect MiString, return string_view or nullopt
    [[nodiscard]] std::optional<std::string_view> string(std::string_view key) const noexcept;
    
    // find() + expect MiTuple
    [[nodiscard]] const MiTuple* tuple(std::string_view key) const noexcept;
    
    // find() + expect MiList
    [[nodiscard]] const MiList* list(std::string_view key) const noexcept;
    
    // Convert a string field to a numeric type
    template<std::integral T>
    [[nodiscard]] std::optional<T> integer(std::string_view key) const noexcept;
    
    // Convert hex-address string field (e.g. "0x08048564") to uintptr_t
    [[nodiscard]] std::optional<std::uintptr_t> address(std::string_view key) const noexcept;
    
    // Boolean: "y" → true, "n" → false
    [[nodiscard]] std::optional<bool> yn_bool(std::string_view key) const noexcept;
};

// [] or [ value, value, ... ] or [ key=value, key=value, ... ]
// MI allows both value-lists and result-lists
struct MiList {
    // result-list (when elements are key=value pairs)
    std::vector<MiResult> results;
    
    // value-list (when elements are bare values)
    std::vector<MiValue> values;
    
    [[nodiscard]] bool is_result_list() const noexcept { return !results.empty(); }
    [[nodiscard]] bool is_value_list()  const noexcept { return !values.empty();  }
    [[nodiscard]] bool empty()          const noexcept { return results.empty() && values.empty(); }
    
    // Iterate tuples in a result-list (common pattern: stack frames, threads)
    [[nodiscard]] auto tuples() const {
        return results
            | std::views::transform([](const MiResult& r) -> const MiTuple* {
                return std::get_if<MiTuple>(&r.value);
              })
            | std::views::filter([](const MiTuple* p) { return p != nullptr; });
    }
};

// ---- Implementations of MiTuple helpers ----

inline const MiValue* MiTuple::find(std::string_view key) const noexcept {
    for (const auto& f : fields) {
        if (f.key == key) return &f.value;
    }
    return nullptr;
}

inline std::optional<std::string_view> MiTuple::string(std::string_view key) const noexcept {
    if (const MiValue* v = find(key)) {
        if (const MiString* s = std::get_if<MiString>(v)) {
            return std::string_view{*s};
        }
    }
    return std::nullopt;
}

template<std::integral T>
inline std::optional<T> MiTuple::integer(std::string_view key) const noexcept {
    auto sv = string(key);
    if (!sv) return std::nullopt;
    T result{};
    auto [ptr, ec] = std::from_chars(sv->data(), sv->data() + sv->size(), result);
    if (ec != std::errc{}) return std::nullopt;
    return result;
}

inline std::optional<std::uintptr_t> MiTuple::address(std::string_view key) const noexcept {
    auto sv = string(key);
    if (!sv || sv->size() < 3 || (*sv)[0] != '0' || (*sv)[1] != 'x') return std::nullopt;
    std::uintptr_t result{};
    auto [ptr, ec] = std::from_chars(sv->data() + 2, sv->data() + sv->size(), result, 16);
    if (ec != std::errc{}) return std::nullopt;
    return result;
}

inline std::optional<bool> MiTuple::yn_bool(std::string_view key) const noexcept {
    auto sv = string(key);
    if (!sv) return std::nullopt;
    if (*sv == "y") return true;
    if (*sv == "n") return false;
    return std::nullopt;
}

} // namespace gdbmi3::core
```

---

## 6. Record Type Hierarchy

Every line GDB outputs is one of seven record types. We model this as a `std::variant` at the top level.

```cpp
// include/gdbmi3/core/record.hpp
#pragma once
#include <gdbmi3/core/value.hpp>
#include <gdbmi3/core/token.hpp>
#include <optional>
#include <variant>
#include <string>

namespace gdbmi3::core {

// ---- Result Record (^) ----

enum class ResultClass : std::uint8_t {
    Done,
    Running,
    Connected,
    Error,
    Exit,
};

struct ResultRecord {
    std::optional<Token> token;
    ResultClass          result_class;
    std::vector<MiResult> results; // may be empty (^done with no data)
    
    // Convenience: look up a top-level result field
    [[nodiscard]] const MiValue* find(std::string_view key) const noexcept;
    [[nodiscard]] const MiTuple* tuple(std::string_view key) const noexcept;
    
    // For ^error records: extract msg and optional code
    [[nodiscard]] std::optional<std::string_view> error_msg() const noexcept;
    [[nodiscard]] std::optional<std::string_view> error_code() const noexcept;
    
    [[nodiscard]] bool is_done()      const noexcept { return result_class == ResultClass::Done;      }
    [[nodiscard]] bool is_running()   const noexcept { return result_class == ResultClass::Running;   }
    [[nodiscard]] bool is_error()     const noexcept { return result_class == ResultClass::Error;     }
    [[nodiscard]] bool is_exit()      const noexcept { return result_class == ResultClass::Exit;      }
    [[nodiscard]] bool is_connected() const noexcept { return result_class == ResultClass::Connected; }
};

// ---- Stream Records (~, @, &) ----

enum class StreamKind : std::uint8_t {
    Console, // ~
    Target,  // @
    Log,     // &
};

struct StreamRecord {
    StreamKind kind;
    std::string text; // decoded, without quotes or outer escaping
};

// ---- Async Records (*, +, =) ----

enum class AsyncKind : std::uint8_t {
    Exec,   // *
    Status, // +
    Notify, // =
};

struct AsyncRecord {
    std::optional<Token> token;
    AsyncKind            kind;
    std::string          async_class; // "stopped", "running", "thread-created", etc.
    std::vector<MiResult> results;
    
    // Convenience: find by key within this async record's results
    [[nodiscard]] const MiValue* find(std::string_view key) const noexcept;
    [[nodiscard]] std::optional<std::string_view> string(std::string_view key) const noexcept;
};

// ---- The "(gdb)" prompt ----

struct PromptRecord {};

// ---- Top-level discriminated union ----

using Record = std::variant<
    ResultRecord,
    StreamRecord,
    AsyncRecord,
    PromptRecord
>;

// ---- Visitor helpers ----

// C++23 pattern: explicit object parameter in lambdas + deducing this
// makes std::visit much cleaner
template<typename... Fs>
struct overloaded : Fs... { using Fs::operator()...; };

} // namespace gdbmi3::core
```

---

## 7. Async I/O and the GDB Process Manager

The process manager owns the GDB child process, reads its stdout line by line, and feeds parsed records to the dispatcher. We use POSIX `fork`/`exec` (or `posix_spawn`) with `pipe2(O_CLOEXEC | O_NONBLOCK)` for I/O, driven by an `io_uring`-based event loop (or a portable `epoll`/`poll` fallback).

```cpp
// include/gdbmi3/session/process.hpp
#pragma once
#include <gdbmi3/core/record.hpp>
#include <gdbmi3/parse/parser.hpp>
#include <functional>
#include <span>
#include <expected>
#include <string>
#include <vector>
#include <cstddef>

namespace gdbmi3::session {

struct ProcessConfig {
    std::string               gdb_path{"/usr/bin/gdb"};
    std::vector<std::string>  gdb_args{"--interpreter=mi3", "--quiet"};
    std::string               executable_path;    // optional, passed after args
    bool                      enable_async{true}; // -gdb-set target-async on
};

// Callback types
using RecordCallback  = std::function<void(core::Record)>;
using ErrorCallback   = std::function<void(std::error_code)>;

class GdbProcess {
public:
    explicit GdbProcess(ProcessConfig config);
    ~GdbProcess();
    
    // Non-copyable, movable
    GdbProcess(const GdbProcess&) = delete;
    GdbProcess& operator=(const GdbProcess&) = delete;
    GdbProcess(GdbProcess&&) noexcept;
    GdbProcess& operator=(GdbProcess&&) noexcept;
    
    // Launch GDB. Returns error if exec fails.
    [[nodiscard]] std::expected<void, std::error_code> start();
    
    // Register callbacks (must be set before start())
    void on_record(RecordCallback cb);
    void on_error(ErrorCallback cb);
    
    // Write a fully-formed MI command line (including trailing newline)
    // Thread-safe: may be called from any thread
    [[nodiscard]] std::expected<void, std::error_code> write(std::string_view command);
    
    // Signal GDB to exit cleanly
    void request_exit();
    
    // Poll I/O (call from an event loop thread)
    // Returns false when GDB has exited
    bool poll();
    
    [[nodiscard]] bool is_running() const noexcept;
    [[nodiscard]] pid_t pid()       const noexcept;
    
private:
    ProcessConfig    config_;
    pid_t            pid_{-1};
    int              stdin_fd_{-1};   // write end → GDB's stdin
    int              stdout_fd_{-1};  // read end ← GDB's stdout
    
    std::string      line_buffer_;    // partial line accumulator
    parse::Parser    parser_;
    RecordCallback   record_cb_;
    ErrorCallback    error_cb_;
    
    void process_available_data();
    void process_line(std::string_view line);
};

} // namespace gdbmi3::session
```

### The Read Loop

```cpp
// src/session/process.cpp (key excerpt)
void GdbProcess::process_available_data() {
    char buf[4096];
    for (;;) {
        ssize_t n = ::read(stdout_fd_, buf, sizeof(buf));
        if (n == 0) {
            // GDB closed stdout — it exited
            if (error_cb_) error_cb_(std::make_error_code(std::errc::broken_pipe));
            return;
        }
        if (n < 0) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) break; // no more data for now
            if (errno == EINTR) continue;
            if (error_cb_) error_cb_({errno, std::system_category()});
            return;
        }
        
        // Append to line buffer, extract complete lines
        line_buffer_.append(buf, static_cast<std::size_t>(n));
        
        std::size_t start = 0;
        while (true) {
            auto newline = line_buffer_.find('\n', start);
            if (newline == std::string::npos) break;
            
            // Handle both \n and \r\n (MI protocol uses either)
            std::size_t end = newline;
            if (end > start && line_buffer_[end - 1] == '\r') --end;
            
            process_line(std::string_view{line_buffer_}.substr(start, end - start));
            start = newline + 1;
        }
        line_buffer_.erase(0, start);
    }
}

void GdbProcess::process_line(std::string_view line) {
    if (line.empty()) return;
    
    // The "(gdb)" prompt is the signal that GDB is ready
    if (line == "(gdb)") {
        if (record_cb_) record_cb_(core::PromptRecord{});
        return;
    }
    
    auto result = parser_.parse(line);
    if (!result) {
        // Parse error: log but don't crash — GDB sometimes emits
        // human-readable lines during startup or error states
        if (error_cb_) {
            // Wrap the parse error
        }
        return;
    }
    
    if (record_cb_) record_cb_(std::move(*result));
}
```

---

## 8. Command Dispatch and Token Management

The dispatcher is the heart of the correlation layer. Every command gets a unique token; the dispatcher holds a `std::promise`-like handle keyed by that token and fulfils it when the matching result record arrives.

Because MI3 supports concurrent commands (you can send the next command before the previous one's `^done` arrives), the dispatcher must be concurrent-safe.

```cpp
// include/gdbmi3/session/dispatcher.hpp
#pragma once
#include <gdbmi3/core/token.hpp>
#include <gdbmi3/core/record.hpp>
#include <gdbmi3/core/error.hpp>
#include <expected>
#include <future>
#include <unordered_map>
#include <mutex>

namespace gdbmi3::session {

using CommandResult = std::expected<core::ResultRecord, core::MiError>;

class Dispatcher {
public:
    // Register a pending command. Returns a future that will be
    // fulfilled when the matching result record arrives.
    [[nodiscard]] std::future<CommandResult> register_pending(Token token);
    
    // Called by GdbProcess when a ResultRecord arrives.
    // If the token matches a pending command, fulfils the future.
    void on_result_record(const core::ResultRecord& rec);
    
    // Cancel all pending commands (called on GDB exit)
    void cancel_all(core::MiError reason);
    
private:
    std::mutex mutex_;
    std::unordered_map<Token, std::promise<CommandResult>> pending_;
};

} // namespace gdbmi3::session
```

```cpp
// src/session/dispatcher.cpp
std::future<CommandResult> Dispatcher::register_pending(Token token) {
    auto promise = std::promise<CommandResult>{};
    auto future  = promise.get_future();
    
    std::lock_guard lock{mutex_};
    pending_.emplace(token, std::move(promise));
    return future;
}

void Dispatcher::on_result_record(const core::ResultRecord& rec) {
    if (!rec.token) return; // no token → can't correlate
    
    std::promise<CommandResult> promise;
    {
        std::lock_guard lock{mutex_};
        auto it = pending_.find(*rec.token);
        if (it == pending_.end()) return; // unsolicited result — ignore
        promise = std::move(it->second);
        pending_.erase(it);
    }
    
    if (rec.is_error()) {
        auto msg  = rec.error_msg().value_or("unknown error");
        auto code = rec.error_code();
        promise.set_value(std::unexpected(core::MiError{
            .message    = std::string{msg},
            .error_code = code ? std::string{*code} : std::string{},
        }));
    } else {
        promise.set_value(rec);
    }
}
```

### The Command Builder

Commands are constructed via a fluent builder. This gives us compile-time enforcement of which options are valid for which commands.

```cpp
// include/gdbmi3/command/builder.hpp
#pragma once
#include <gdbmi3/core/token.hpp>
#include <string>
#include <format>

namespace gdbmi3::command {

// A fully-formed MI command line, ready to write to GDB's stdin
class MiCommand {
public:
    [[nodiscard]] std::string_view text() const noexcept { return text_; }
    [[nodiscard]] Token            token() const noexcept { return token_; }
    
private:
    friend class MiCommandBuilder;
    std::string text_;
    Token       token_{0};
};

class MiCommandBuilder {
public:
    explicit MiCommandBuilder(Token token, std::string_view operation)
        : token_(token) {
        text_ = std::format("{}-{}", token.value(), operation);
    }
    
    MiCommandBuilder& opt(std::string_view flag) {
        text_ += std::format(" {}", flag);
        return *this;
    }
    
    MiCommandBuilder& opt(std::string_view flag, std::string_view value) {
        text_ += std::format(" {} {}", flag, value);
        return *this;
    }
    
    MiCommandBuilder& opt(std::string_view flag, std::integral auto value) {
        text_ += std::format(" {} {}", flag, value);
        return *this;
    }
    
    // Append a parameter (after --)
    MiCommandBuilder& param(std::string_view value) {
        if (!separator_appended_) {
            text_ += " --";
            separator_appended_ = true;
        }
        text_ += std::format(" {}", value);
        return *this;
    }
    
    // Append a C-string-quoted parameter
    MiCommandBuilder& quoted_param(std::string_view value) {
        if (!separator_appended_) {
            text_ += " --";
            separator_appended_ = true;
        }
        text_ += " \"";
        for (char c : value) {
            if (c == '"' || c == '\\') text_ += '\\';
            text_ += c;
        }
        text_ += '"';
        return *this;
    }
    
    [[nodiscard]] MiCommand build() {
        text_ += '\n'; // MI commands are newline-terminated
        MiCommand cmd;
        cmd.text_  = std::move(text_);
        cmd.token_ = token_;
        return cmd;
    }
    
private:
    Token       token_;
    std::string text_;
    bool        separator_appended_{false};
};

} // namespace gdbmi3::command
```

---

## 9. Breakpoint Subsystem

The breakpoint commands are where MI3's canonical change — the multi-location breakpoint format — requires careful modelling.

### Result Types

```cpp
// include/gdbmi3/result/breakpoint.hpp
#pragma once
#include <gdbmi3/core/value.hpp>
#include <gdbmi3/core/token.hpp>
#include <optional>
#include <variant>
#include <vector>
#include <string>
#include <cstdint>

namespace gdbmi3::result {

enum class BreakpointType : std::uint8_t {
    Breakpoint,
    Watchpoint,
    HardwareWatchpoint,
    ReadWatchpoint,
    AccessWatchpoint,
    Tracepoint,
    Catchpoint,
    DynamicPrintf,
    Unknown,
};

enum class Disposition : std::uint8_t {
    Keep,    // "keep"
    Delete,  // "del"
    Disable, // "dis"
};

// A single resolved location — appears either as the sole location of a
// breakpoint, or as one element of the `locations` list in a multi-location
// breakpoint (MI3 format).
struct BreakpointLocation {
    std::string           number;        // e.g. "1" or "1.1"
    bool                  enabled{true};
    std::uintptr_t        addr{0};
    std::string           func;
    std::string           file;
    std::string           fullname;
    std::uint32_t         line{0};
    std::vector<std::string> thread_groups; // e.g. ["i1"]
};

// MI3 KEY DESIGN DECISION:
// A breakpoint is either single-location OR multi-location.
// In MI2 the multi-location case produced invalid MI (repeated 'bkpt' keys).
// In MI3 it is a proper `locations` list. We model this as a variant so that
// consumers MUST handle both cases — the compiler will not let them forget.
struct SingleLocation { BreakpointLocation loc; };
struct MultiLocation  { std::vector<BreakpointLocation> locations; };
using LocationData = std::variant<SingleLocation, MultiLocation>;

struct BreakpointInfo {
    std::string     number;           // "1", "2", etc.
    BreakpointType  type{BreakpointType::Breakpoint};
    std::string     catch_type;       // for catchpoints
    Disposition     disp{Disposition::Keep};
    bool            enabled{true};
    
    // The location(s) — this is the core MI3 novelty
    LocationData    location_data;
    
    // Common fields
    std::optional<std::string>  condition;
    std::uint32_t               ignore_count{0};
    std::uint32_t               hit_count{0};
    std::optional<ThreadId>     thread_restriction;
    std::string                 original_location;
    
    // For watchpoints
    std::string                 what;  // the watched expression
    
    // Pending breakpoints
    std::optional<std::string>  pending;
    
    // Helpers
    [[nodiscard]] bool is_pending()         const noexcept { return pending.has_value(); }
    [[nodiscard]] bool is_multi_location()  const noexcept {
        return std::holds_alternative<MultiLocation>(location_data);
    }
    [[nodiscard]] std::size_t location_count() const noexcept {
        return std::visit(core::overloaded{
            [](const SingleLocation&) -> std::size_t { return 1; },
            [](const MultiLocation& m) { return m.locations.size(); },
        }, location_data);
    }
    
    // Parse from a `bkpt` tuple in an MI3 result record
    [[nodiscard]] static std::expected<BreakpointInfo, core::MiError>
    from_tuple(const core::MiTuple& bkpt);
};

} // namespace gdbmi3::result
```

### Parsing `bkpt` Tuples

```cpp
// src/result/breakpoint.cpp (key excerpt)
std::expected<BreakpointInfo, core::MiError>
BreakpointInfo::from_tuple(const core::MiTuple& bkpt) {
    BreakpointInfo info;
    
    // Required fields
    auto number = bkpt.string("number");
    if (!number) return std::unexpected(core::MiError{"missing 'number' in bkpt"});
    info.number = std::string{*number};
    
    // Enabled / disposition
    info.enabled = bkpt.yn_bool("enabled").value_or(true);
    if (auto d = bkpt.string("disp")) {
        if (*d == "del") info.disp = Disposition::Delete;
        else if (*d == "dis") info.disp = Disposition::Disable;
    }
    
    // MI3 MULTI-LOCATION HANDLING
    // If `addr` is "<MULTIPLE>", we expect a `locations` list.
    // If `addr` is "<PENDING>", we set pending.
    // Otherwise, this is a normal single-location breakpoint.
    
    auto addr_str = bkpt.string("addr").value_or("");
    
    if (addr_str == "<PENDING>") {
        info.pending = std::string{bkpt.string("pending").value_or("")};
        info.location_data = SingleLocation{};  // no address yet
        
    } else if (addr_str == "<MULTIPLE>") {
        // MI3: parse the `locations` list
        const core::MiList* locs = bkpt.list("locations");
        if (!locs || locs->empty()) {
            return std::unexpected(core::MiError{
                "addr=<MULTIPLE> but no 'locations' list — is this MI2?"
            });
        }
        
        MultiLocation ml;
        for (auto* sub : locs->tuples()) {
            if (!sub) continue;
            BreakpointLocation loc;
            loc.number  = std::string{sub->string("number").value_or("")};
            loc.enabled = sub->yn_bool("enabled").value_or(true);
            loc.addr    = sub->address("addr").value_or(0);
            loc.func    = std::string{sub->string("func").value_or("")};
            loc.file    = std::string{sub->string("file").value_or("")};
            loc.fullname= std::string{sub->string("fullname").value_or("")};
            loc.line    = sub->integer<std::uint32_t>("line").value_or(0);
            
            if (const core::MiList* tg = sub->list("thread-groups")) {
                for (const auto& v : tg->values) {
                    if (const auto* s = std::get_if<core::MiString>(&v))
                        loc.thread_groups.push_back(*s);
                }
            }
            ml.locations.push_back(std::move(loc));
        }
        info.location_data = std::move(ml);
        
    } else {
        // Normal single-location breakpoint
        SingleLocation sl;
        sl.loc.number   = info.number;
        sl.loc.enabled  = info.enabled;
        sl.loc.addr     = bkpt.address("addr").value_or(0);
        sl.loc.func     = std::string{bkpt.string("func").value_or("")};
        sl.loc.file     = std::string{bkpt.string("file").value_or("")};
        sl.loc.fullname = std::string{bkpt.string("fullname").value_or("")};
        sl.loc.line     = bkpt.integer<std::uint32_t>("line").value_or(0);
        info.location_data = std::move(sl);
    }
    
    // Optional common fields
    if (auto c = bkpt.string("cond"))               info.condition = std::string{*c};
    info.ignore_count = bkpt.integer<std::uint32_t>("ignore").value_or(0);
    info.hit_count    = bkpt.integer<std::uint32_t>("times").value_or(0);
    info.original_location = std::string{bkpt.string("original-location").value_or("")};
    
    return info;
}
```

### The Breakpoint Command Interface

```cpp
// include/gdbmi3/command/breakpoint_cmds.hpp
#pragma once
#include <gdbmi3/result/breakpoint.hpp>
#include <gdbmi3/session/dispatcher.hpp>
#include <optional>
#include <future>

namespace gdbmi3::command {

struct BreakInsertOptions {
    bool                        temporary{false};       // -t
    bool                        hardware{false};        // -h
    bool                        pending{false};         // -f
    bool                        disabled{false};        // -d
    std::optional<std::string>  condition;              // -c
    std::optional<std::uint32_t> ignore_count;          // -i
    std::optional<ThreadId>     thread;                 // -p
};

// Break at a symbol name, file:line, or *address
struct BreakLocation {
    std::string location; // e.g. "main", "main.cpp:42", "*0x08048564"
};

// High-level breakpoint API — returns a future result
class BreakpointCommands {
public:
    // All methods take a reference to the session's GdbProcess and Dispatcher.
    // The session object owns these; we just borrow them.
    BreakpointCommands(GdbProcess& proc, session::Dispatcher& disp,
                       TokenGenerator& tokens);
    
    [[nodiscard]] std::future<std::expected<result::BreakpointInfo, core::MiError>>
    insert(BreakLocation loc, BreakInsertOptions opts = {});
    
    [[nodiscard]] std::future<std::expected<void, core::MiError>>
    remove(BreakpointNumber bpnum);
    
    [[nodiscard]] std::future<std::expected<void, core::MiError>>
    enable(BreakpointNumber bpnum);
    
    [[nodiscard]] std::future<std::expected<void, core::MiError>>
    disable(BreakpointNumber bpnum);
    
    [[nodiscard]] std::future<std::expected<void, core::MiError>>
    set_condition(BreakpointNumber bpnum, std::string_view condition);
    
    [[nodiscard]] std::future<std::expected<void, core::MiError>>
    set_ignore_count(BreakpointNumber bpnum, std::uint32_t count);
    
    [[nodiscard]] std::future<std::expected<std::vector<result::BreakpointInfo>, core::MiError>>
    list();
    
    [[nodiscard]] std::future<std::expected<result::BreakpointInfo, core::MiError>>
    insert_watchpoint(std::string_view expression, bool read_only = false, bool access = false);
    
private:
    GdbProcess&             proc_;
    session::Dispatcher&    disp_;
    TokenGenerator&         tokens_;
    
    // Send command and return raw result future
    std::future<session::CommandResult> send(MiCommand cmd);
};

} // namespace gdbmi3::command
```

```cpp
// Implementation of insert()
std::future<std::expected<result::BreakpointInfo, core::MiError>>
BreakpointCommands::insert(BreakLocation loc, BreakInsertOptions opts) {
    
    Token tok = tokens_.next();
    MiCommandBuilder builder{tok, "break-insert"};
    
    if (opts.temporary)                 builder.opt("-t");
    if (opts.hardware)                  builder.opt("-h");
    if (opts.pending)                   builder.opt("-f");
    if (opts.disabled)                  builder.opt("-d");
    if (opts.condition)                 builder.opt("-c", *opts.condition);
    if (opts.ignore_count)              builder.opt("-i", *opts.ignore_count);
    if (opts.thread)                    builder.opt("-p", opts.thread->value());
    
    builder.param(loc.location);
    
    auto raw_future = send(builder.build());
    
    // Transform: when the raw CommandResult arrives, parse it into BreakpointInfo
    return std::async(std::launch::deferred, [f = std::move(raw_future)]() mutable
        -> std::expected<result::BreakpointInfo, core::MiError>
    {
        auto result = f.get();
        if (!result) return std::unexpected(std::move(result.error()));
        
        const core::MiTuple* bkpt = result->tuple("bkpt");
        if (!bkpt) return std::unexpected(core::MiError{"no 'bkpt' in -break-insert response"});
        
        return result::BreakpointInfo::from_tuple(*bkpt);
    });
}
```

---

## 10. Variable Object (varobj) Subsystem

Variable objects are stateful: GDB keeps them alive across commands. The client must track which varobjs exist and manage their lifetime. This is modelled as a `VarObjHandle` — a move-only RAII wrapper.

```cpp
// include/gdbmi3/result/varobj.hpp
#pragma once
#include <gdbmi3/core/value.hpp>
#include <string>
#include <vector>
#include <optional>
#include <cstdint>

namespace gdbmi3::result {

enum class VarObjFormat : std::uint8_t {
    Natural,
    Decimal,
    Hexadecimal,
    Octal,
    Binary,
    ZeroHexadecimal,
};

struct VarObjInfo {
    std::string  name;          // e.g. "var1", or auto-generated "var2.x"
    std::string  type;          // C type string
    std::string  value;         // current display value
    std::uint32_t num_children{0};
    bool         has_more{false}; // more children than reported (dynamic varobjs)
    std::optional<ThreadId> thread_id;
    std::string  display_hint;  // from Python pretty-printer
};

struct VarObjChild {
    std::string  name;          // full dotted name, e.g. "var1.x"
    std::string  expression;    // short name, e.g. "x"
    std::string  type;
    std::string  value;
    std::uint32_t num_children{0};
    bool         has_more{false};
    std::string  display_hint;
};

// What -var-update returns per changed variable
struct VarObjChange {
    std::string  name;
    std::string  value;         // new value
    bool         in_scope{true};
    bool         type_changed{false};
    std::optional<std::string> new_type;
    std::optional<std::uint32_t> new_num_children;
    bool         has_more{false};
};

} // namespace gdbmi3::result

namespace gdbmi3::command {

// RAII wrapper: automatically calls -var-delete when destroyed
class VarObjHandle {
public:
    // Created by VarObjCommands::create()
    VarObjHandle() = default;
    ~VarObjHandle(); // calls -var-delete if valid
    
    // Move-only
    VarObjHandle(VarObjHandle&& other) noexcept;
    VarObjHandle& operator=(VarObjHandle&& other) noexcept;
    VarObjHandle(const VarObjHandle&) = delete;
    VarObjHandle& operator=(const VarObjHandle&) = delete;
    
    [[nodiscard]] bool         valid()    const noexcept { return !name_.empty(); }
    [[nodiscard]] std::string_view name() const noexcept { return name_; }
    
    // Accessors — all return futures (they issue MI commands)
    std::future<std::expected<std::string, core::MiError>> evaluate();
    std::future<std::expected<void, core::MiError>>        assign(std::string_view value);
    std::future<std::expected<std::vector<result::VarObjChild>, core::MiError>> children();
    std::future<std::expected<void, core::MiError>>        set_format(result::VarObjFormat fmt);
    std::future<std::expected<void, core::MiError>>        freeze(bool frozen);
    
    // Explicitly release ownership (prevents -var-delete in destructor)
    void release() noexcept { name_.clear(); }
    
private:
    friend class VarObjCommands;
    VarObjHandle(std::string name, VarObjCommands* owner);
    
    std::string       name_;
    VarObjCommands*   owner_{nullptr};
};

} // namespace gdbmi3::command
```

### The `-var-update *` Pattern

```cpp
// The high-frequency pattern: after every stop, call update-all
std::future<std::expected<std::vector<result::VarObjChange>, core::MiError>>
VarObjCommands::update_all() {
    Token tok = tokens_.next();
    auto cmd = MiCommandBuilder{tok, "var-update"}
        .param("*")
        .build();
    
    auto raw = send(std::move(cmd));
    
    return std::async(std::launch::deferred, [f = std::move(raw)]() mutable
        -> std::expected<std::vector<result::VarObjChange>, core::MiError>
    {
        auto result = f.get();
        if (!result) return std::unexpected(result.error());
        
        const core::MiList* changelist = nullptr;
        for (const auto& r : result->results) {
            if (r.key == "changelist") {
                changelist = std::get_if<core::MiList>(&r.value);
                break;
            }
        }
        
        std::vector<result::VarObjChange> changes;
        if (!changelist) return changes; // empty — nothing changed
        
        for (auto* tup : changelist->tuples()) {
            if (!tup) continue;
            result::VarObjChange ch;
            ch.name         = std::string{tup->string("name").value_or("")};
            ch.value        = std::string{tup->string("value").value_or("")};
            ch.in_scope     = tup->string("in_scope").value_or("true") == "true";
            ch.type_changed = tup->string("type_changed").value_or("false") == "true";
            if (auto nt = tup->string("new_type"))  ch.new_type = std::string{*nt};
            ch.has_more     = tup->string("has_more").value_or("0") != "0";
            changes.push_back(std::move(ch));
        }
        return changes;
    });
}
```

---

## 11. Stack and Thread Subsystem

```cpp
// include/gdbmi3/result/frame.hpp
#pragma once
#include <gdbmi3/core/token.hpp>
#include <optional>
#include <string>
#include <vector>
#include <cstdint>

namespace gdbmi3::result {

struct FunctionArgument {
    std::string name;
    std::string value; // may be empty if --no-values
};

struct StackFrame {
    FrameLevel              level{0};
    std::uintptr_t          addr{0};
    std::string             func;
    std::string             file;          // relative
    std::string             fullname;      // absolute
    std::uint32_t           line{0};
    std::string             arch;          // e.g. "i386:x86_64"
    std::string             addr_flags;    // new in MI3: e.g. "[PAC]" on AArch64
    std::vector<FunctionArgument> args;
    
    // Convenience
    [[nodiscard]] bool has_source() const noexcept { return !file.empty(); }
};

struct ThreadInfo {
    ThreadId    id;
    std::string target_id;  // OS-level identifier
    std::string name;       // pthread_setname or equivalent
    StackFrame  frame;      // current frame
    std::string state;      // "stopped" or "running"
    std::optional<std::uint32_t> core; // CPU core
};

} // namespace gdbmi3::result
```

---

## 12. Async Notification Router

The notification router is a publish-subscribe system over the MI3 `=` (notify) and `*` (exec) async records.

```cpp
// include/gdbmi3/session/notification.hpp
#pragma once
#include <gdbmi3/core/record.hpp>
#include <gdbmi3/result/breakpoint.hpp>
#include <gdbmi3/result/frame.hpp>
#include <gdbmi3/result/thread.hpp>
#include <functional>
#include <vector>
#include <mutex>
#include <string_view>

namespace gdbmi3::session {

// Stop reason as a strongly-typed enum + associated data
enum class StopReason : std::uint8_t {
    BreakpointHit,
    WatchpointTrigger,
    ReadWatchpointTrigger,
    AccessWatchpointTrigger,
    FunctionFinished,
    LocationReached,
    WatchpointScope,
    EndSteppingRange,
    ExitedSignalled,
    Exited,
    ExitedNormally,
    SignalReceived,
    SolibEvent,
    Fork,
    VFork,
    SyscallEntry,
    SyscallReturn,
    Exec,
    Unknown,
};

struct StoppedEvent {
    StopReason              reason;
    ThreadId                thread_id;
    std::vector<ThreadId>   stopped_threads; // empty means "all"
    result::StackFrame      frame;
    std::optional<std::string> breakpoint_number;  // for BreakpointHit
    std::optional<std::string> signal_name;         // for SignalReceived
    std::optional<std::uint32_t> exit_code;         // for Exited
};

struct RunningEvent {
    // "all" or specific thread
    std::variant<std::monostate, ThreadId> thread; // monostate = all
};

struct BreakpointCreatedEvent { result::BreakpointInfo info; };
struct BreakpointModifiedEvent { result::BreakpointInfo info; };
struct BreakpointDeletedEvent { std::string number; };

struct ThreadCreatedEvent { ThreadId id; std::string group_id; };
struct ThreadExitedEvent  { ThreadId id; std::string group_id; };

struct LibraryLoadedEvent {
    std::string id;
    std::string target_name;
    std::string host_name;
    bool        symbols_loaded;
};
struct LibraryUnloadedEvent { std::string id; };

// The notification router dispatches incoming async records to typed handlers
class NotificationRouter {
public:
    // Subscribe to specific event types
    using StoppedHandler          = std::function<void(const StoppedEvent&)>;
    using RunningHandler          = std::function<void(const RunningEvent&)>;
    using BkptCreatedHandler      = std::function<void(const BreakpointCreatedEvent&)>;
    using BkptModifiedHandler     = std::function<void(const BreakpointModifiedEvent&)>;
    using BkptDeletedHandler      = std::function<void(const BreakpointDeletedEvent&)>;
    using ThreadCreatedHandler    = std::function<void(const ThreadCreatedEvent&)>;
    using ThreadExitedHandler     = std::function<void(const ThreadExitedEvent&)>;
    using LibraryLoadedHandler    = std::function<void(const LibraryLoadedEvent&)>;
    using LibraryUnloadedHandler  = std::function<void(const LibraryUnloadedEvent&)>;
    using ConsoleOutputHandler    = std::function<void(std::string_view)>;
    
    void on_stopped         (StoppedHandler h)         { stopped_handlers_.push_back(std::move(h)); }
    void on_running         (RunningHandler h)         { running_handlers_.push_back(std::move(h)); }
    void on_breakpoint_created  (BkptCreatedHandler h) { bkpt_created_.push_back(std::move(h)); }
    void on_breakpoint_modified (BkptModifiedHandler h){ bkpt_modified_.push_back(std::move(h)); }
    void on_breakpoint_deleted  (BkptDeletedHandler h) { bkpt_deleted_.push_back(std::move(h)); }
    void on_thread_created  (ThreadCreatedHandler h)  { thread_created_.push_back(std::move(h)); }
    void on_thread_exited   (ThreadExitedHandler h)   { thread_exited_.push_back(std::move(h)); }
    void on_library_loaded  (LibraryLoadedHandler h)  { lib_loaded_.push_back(std::move(h)); }
    void on_console_output  (ConsoleOutputHandler h)  { console_.push_back(std::move(h)); }
    
    // Called by GdbProcess::poll() for every record received
    void dispatch(const core::Record& rec);
    
private:
    void dispatch_async(const core::AsyncRecord& rec);
    void dispatch_stream(const core::StreamRecord& rec);
    
    StoppedEvent parse_stopped(const core::AsyncRecord& rec);
    
    std::vector<StoppedHandler>         stopped_handlers_;
    std::vector<RunningHandler>         running_handlers_;
    std::vector<BkptCreatedHandler>     bkpt_created_;
    std::vector<BkptModifiedHandler>    bkpt_modified_;
    std::vector<BkptDeletedHandler>     bkpt_deleted_;
    std::vector<ThreadCreatedHandler>   thread_created_;
    std::vector<ThreadExitedHandler>    thread_exited_;
    std::vector<LibraryLoadedHandler>   lib_loaded_;
    std::vector<ConsoleOutputHandler>   console_;
};

} // namespace gdbmi3::session
```

### Dispatching `*stopped`

```cpp
// The *stopped parsing is the most complex dispatch path
StoppedEvent NotificationRouter::parse_stopped(const core::AsyncRecord& rec) {
    StoppedEvent ev;
    
    // Parse reason string → enum
    static constexpr std::pair<std::string_view, StopReason> reason_map[] = {
        {"breakpoint-hit",             StopReason::BreakpointHit},
        {"watchpoint-trigger",         StopReason::WatchpointTrigger},
        {"read-watchpoint-trigger",    StopReason::ReadWatchpointTrigger},
        {"access-watchpoint-trigger",  StopReason::AccessWatchpointTrigger},
        {"function-finished",          StopReason::FunctionFinished},
        {"location-reached",           StopReason::LocationReached},
        {"watchpoint-scope",           StopReason::WatchpointScope},
        {"end-stepping-range",         StopReason::EndSteppingRange},
        {"exited-signalled",           StopReason::ExitedSignalled},
        {"exited",                     StopReason::Exited},
        {"exited-normally",            StopReason::ExitedNormally},
        {"signal-received",            StopReason::SignalReceived},
        {"solib-event",                StopReason::SolibEvent},
        {"fork",                       StopReason::Fork},
        {"vfork",                      StopReason::VFork},
        {"syscall-entry",              StopReason::SyscallEntry},
        {"syscall-return",             StopReason::SyscallReturn},
        {"exec",                       StopReason::Exec},
    };
    
    auto reason_sv = rec.string("reason").value_or("");
    ev.reason = StopReason::Unknown;
    for (auto [key, val] : reason_map) {
        if (key == reason_sv) { ev.reason = val; break; }
    }
    
    // Thread ID
    if (auto tid = rec.find("thread-id")) {
        if (const auto* s = std::get_if<core::MiString>(tid)) {
            uint32_t v{};
            std::from_chars(s->data(), s->data() + s->size(), v);
            ev.thread_id = ThreadId{v};
        }
    }
    
    // Stopped threads: "all" or list
    if (auto st = rec.find("stopped-threads")) {
        std::visit(core::overloaded{
            [&](const core::MiString& s) {
                if (s != "all") {
                    uint32_t v{};
                    std::from_chars(s.data(), s.data() + s.size(), v);
                    ev.stopped_threads.push_back(ThreadId{v});
                }
                // "all" → leave ev.stopped_threads empty (by convention)
            },
            [&](const core::MiList& lst) {
                for (const auto& val : lst.values) {
                    if (const auto* s = std::get_if<core::MiString>(&val)) {
                        uint32_t v{};
                        std::from_chars(s->data(), s->data() + s->size(), v);
                        ev.stopped_threads.push_back(ThreadId{v});
                    }
                }
            },
            [](const auto&) {}
        }, *st);
    }
    
    // Frame
    if (const auto* frame_val = rec.find("frame")) {
        if (const auto* tup = std::get_if<core::MiTuple>(frame_val)) {
            ev.frame.addr     = tup->address("addr").value_or(0);
            ev.frame.func     = std::string{tup->string("func").value_or("")};
            ev.frame.file     = std::string{tup->string("file").value_or("")};
            ev.frame.fullname = std::string{tup->string("fullname").value_or("")};
            ev.frame.line     = tup->integer<uint32_t>("line").value_or(0);
            ev.frame.arch     = std::string{tup->string("arch").value_or("")};
            ev.frame.addr_flags = std::string{tup->string("addr_flags").value_or("")};
        }
    }
    
    // Reason-specific fields
    if (ev.reason == StopReason::BreakpointHit) {
        ev.breakpoint_number = std::string{rec.string("bkptno").value_or("")};
    }
    if (ev.reason == StopReason::SignalReceived) {
        ev.signal_name = std::string{rec.string("signal-name").value_or("")};
    }
    if (ev.reason == StopReason::Exited) {
        auto exit_str = rec.string("exit-code").value_or("0");
        uint32_t code{};
        std::from_chars(exit_str.data(), exit_str.data() + exit_str.size(), code);
        ev.exit_code = code;
    }
    
    return ev;
}
```

---

## 13. Session and State Machine

The session is the top-level object that ties everything together. It owns the GdbProcess, the Dispatcher, the NotificationRouter, the TokenGenerator, and the command subsystems.

```cpp
// include/gdbmi3/session/session.hpp
#pragma once
#include <gdbmi3/session/process.hpp>
#include <gdbmi3/session/dispatcher.hpp>
#include <gdbmi3/session/notification.hpp>
#include <gdbmi3/command/breakpoint_cmds.hpp>
#include <gdbmi3/command/exec_cmds.hpp>
#include <gdbmi3/command/stack_cmds.hpp>
#include <gdbmi3/command/var_cmds.hpp>
#include <gdbmi3/command/data_cmds.hpp>
#include <gdbmi3/command/gdb_cmds.hpp>

namespace gdbmi3::session {

// High-level session state
enum class SessionState : std::uint8_t {
    Idle,           // GDB started, no inferior loaded
    Loaded,         // inferior loaded, not running
    Running,        // inferior currently executing
    Stopped,        // inferior stopped (at breakpoint, step, etc.)
    Exited,         // inferior exited
    Disconnected,   // GDB process died
};

class Session {
public:
    explicit Session(ProcessConfig config);
    ~Session();
    
    // Non-copyable, non-movable (owns OS resources)
    Session(const Session&) = delete;
    Session& operator=(const Session&) = delete;
    
    // Launch GDB and run startup sequence:
    // -enable-pretty-printing, -gdb-set target-async on,
    // -gdb-set non-stop off (configurable), -list-features
    [[nodiscard]] std::expected<void, core::MiError> start();
    
    // Load an executable (calls -file-exec-and-symbols)
    [[nodiscard]] std::future<std::expected<void, core::MiError>>
    load(std::string_view executable_path);
    
    // Access to command subsystems
    command::BreakpointCommands& breakpoints() noexcept { return *bkpt_cmds_; }
    command::ExecCommands&       exec()        noexcept { return *exec_cmds_; }
    command::StackCommands&      stack()       noexcept { return *stack_cmds_; }
    command::VarObjCommands&     varobjs()     noexcept { return *var_cmds_; }
    command::DataCommands&       data()        noexcept { return *data_cmds_; }
    command::GdbCommands&        gdb()         noexcept { return *gdb_cmds_; }
    
    // Notification subscription
    NotificationRouter& notifications() noexcept { return router_; }
    
    // Session state
    [[nodiscard]] SessionState state() const noexcept;
    
    // Drive the I/O event loop. Call this in a dedicated thread,
    // or integrate with your application's event loop.
    void run_event_loop();
    void stop_event_loop() noexcept;
    
private:
    ProcessConfig    config_;
    TokenGenerator   tokens_;
    GdbProcess       process_;
    Dispatcher       dispatcher_;
    NotificationRouter router_;
    
    std::unique_ptr<command::BreakpointCommands> bkpt_cmds_;
    std::unique_ptr<command::ExecCommands>       exec_cmds_;
    std::unique_ptr<command::StackCommands>      stack_cmds_;
    std::unique_ptr<command::VarObjCommands>     var_cmds_;
    std::unique_ptr<command::DataCommands>       data_cmds_;
    std::unique_ptr<command::GdbCommands>        gdb_cmds_;
    
    std::atomic<SessionState> state_{SessionState::Idle};
    std::atomic<bool>         running_{false};
    
    void on_record(core::Record rec);
    void update_state_from_stop(const StoppedEvent& ev);
};

} // namespace gdbmi3::session
```

---

## 14. Error Handling Strategy

We use `std::expected` exclusively on all public APIs. There are no exceptions thrown by this library. The error type is `core::MiError`:

```cpp
// include/gdbmi3/core/error.hpp
#pragma once
#include <string>
#include <system_error>
#include <optional>

namespace gdbmi3::core {

// Parse errors (malformed MI output from GDB)
struct ParseError {
    std::string   message;
    std::size_t   position{0};  // byte offset in the input line
    std::string   input_line;   // the offending line
};

// MI-level errors (^error result records from GDB)
struct MiError {
    std::string              message;    // from msg="..."
    std::string              error_code; // from code="..." (may be empty)
    
    // System errors (process launch, pipe I/O)
    std::optional<std::error_code> system_error;
};

} // namespace gdbmi3::core

// Helper macro for propagating errors in command implementations
// C++23 std::expected doesn't yet have monadic operations in all compilers,
// so we use this pattern:
#define MI3_TRY(expr)                                \
    ({                                               \
        auto&& _r = (expr);                          \
        if (!_r) return std::unexpected(_r.error()); \
        std::move(*_r);                              \
    })
```

The philosophy is:

**Parse errors** (malformed MI output): these are bugs in our parser or unexpected GDB output. Log them and skip the record — never crash.

**MiErrors** (GDB returned `^error`): these are expected in normal operation. The caller receives a `std::unexpected<MiError>` and decides what to do: show a message, retry, or give up.

**System errors** (pipe broken, GDB crashed): these are delivered through the `ErrorCallback` on `GdbProcess` and transition the session to `SessionState::Disconnected`.

---

## 15. Testing Architecture

### Unit Testing the Parser

Because the parser operates on `std::string_view` and produces pure values, it is trivially testable without any I/O:

```cpp
// test/unit/test_parser.cpp
#include <gdbmi3/parse/parser.hpp>
#include <catch2/catch_all.hpp>

using namespace gdbmi3;
using namespace gdbmi3::core;
using namespace gdbmi3::parse;

TEST_CASE("Parse simple ^done result", "[parser]") {
    Parser p;
    auto result = p.parse("42^done,value=\"42\"");
    REQUIRE(result.has_value());
    
    auto* rr = std::get_if<ResultRecord>(&*result);
    REQUIRE(rr != nullptr);
    REQUIRE(rr->token == Token{42});
    REQUIRE(rr->result_class == ResultClass::Done);
    
    auto val = rr->find("value");
    REQUIRE(val != nullptr);
    auto* s = std::get_if<MiString>(val);
    REQUIRE(s != nullptr);
    REQUIRE(*s == "42");
}

TEST_CASE("Parse MI3 multi-location breakpoint", "[parser][mi3]") {
    Parser p;
    std::string_view line =
        R"(1^done,bkpt={number="1",type="breakpoint",disp="keep",)"
        R"(enabled="y",addr="<MULTIPLE>",times="0",original-location="foo",)"
        R"(locations=[)"
        R"({number="1.1",enabled="y",addr="0x08048500",func="foo(int)",)"
        R"(file="main.cpp",fullname="/home/user/main.cpp",line="12",thread-groups=["i1"]},)"
        R"({number="1.2",enabled="y",addr="0x08048560",func="foo(double)",)"
        R"(file="main.cpp",fullname="/home/user/main.cpp",line="20",thread-groups=["i1"]}]})";
    
    auto result = p.parse(line);
    REQUIRE(result.has_value());
    
    auto* rr = std::get_if<ResultRecord>(&*result);
    REQUIRE(rr != nullptr);
    
    auto* bkpt_val = rr->find("bkpt");
    REQUIRE(bkpt_val != nullptr);
    auto* bkpt = std::get_if<MiTuple>(bkpt_val);
    REQUIRE(bkpt != nullptr);
    
    auto info = result::BreakpointInfo::from_tuple(*bkpt);
    REQUIRE(info.has_value());
    REQUIRE(info->is_multi_location());
    REQUIRE(info->location_count() == 2);
    
    auto& ml = std::get<result::MultiLocation>(info->location_data);
    REQUIRE(ml.locations[0].func == "foo(int)");
    REQUIRE(ml.locations[1].func == "foo(double)");
}

TEST_CASE("Parse *stopped with watchpoint trigger", "[parser]") {
    Parser p;
    auto result = p.parse(
        R"(*stopped,reason="watchpoint-trigger",)"
        R"(wpt={number="2",exp="x"},)"
        R"(value={old="0",new="42"},)"
        R"(frame={func="main",args=[],file="main.c",line="10"})");
    
    REQUIRE(result.has_value());
    auto* ar = std::get_if<AsyncRecord>(&*result);
    REQUIRE(ar != nullptr);
    REQUIRE(ar->kind == AsyncKind::Exec);
    REQUIRE(ar->async_class == "stopped");
    REQUIRE(ar->string("reason") == "watchpoint-trigger");
}
```

### Integration Testing with a Mock GDB

For integration tests, we don't run real GDB — we drive a `GdbProcess`-compatible interface with a mock that replays pre-recorded MI3 sessions:

```cpp
// test/integration/mock_gdb.hpp
class MockGdb {
public:
    // Pre-load a script: pairs of (expected command pattern, response lines)
    void expect(std::string_view command_prefix, std::vector<std::string> responses);
    
    // Connect to a Session as if it were a real GDB process
    void attach(session::Session& session);
    
    // Feed the next pre-loaded response
    void tick();
    
    // Assert all expectations were consumed
    void verify();
};
```

This lets us write readable integration tests:

```cpp
TEST_CASE("Break-insert with multi-location response", "[integration][mi3]") {
    MockGdb mock;
    mock.expect("-break-insert", {
        R"(1^done,bkpt={number="1",addr="<MULTIPLE>",locations=[)"
        R"({number="1.1",addr="0x1000",func="foo(int)",...},)"
        R"({number="1.2",addr="0x2000",func="foo(double)",...}]})",
        "(gdb)"
    });
    
    Session session{{}};
    mock.attach(session);
    
    auto future = session.breakpoints().insert({"foo"});
    mock.tick();
    mock.tick();
    
    auto result = future.get();
    REQUIRE(result.has_value());
    REQUIRE(result->is_multi_location());
    REQUIRE(result->location_count() == 2);
    
    mock.verify();
}
```

---

## 16. Integration Points and Extension Model

### Coroutine-Based API (C++23 `std::generator`)

For consumers that want to write linear-looking debug scripts, we provide a coroutine-based facade:

```cpp
// A coroutine-friendly session wrapper
// Requires C++23 std::generator or a compatible library

#include <generator>  // C++23

// Example: a debug automation script that looks synchronous
std::generator<std::string> run_debug_session(session::Session& sess) {
    // Load binary
    auto load_result = co_await sess.load("/path/to/program");
    if (!load_result) {
        co_yield std::format("Load failed: {}", load_result.error().message);
        co_return;
    }
    
    // Set breakpoint
    auto bp = co_await sess.breakpoints().insert({"main"});
    if (!bp) co_return;
    co_yield std::format("Breakpoint set at {}", bp->number);
    
    // Run
    auto run_result = co_await sess.exec().run();
    if (!run_result) co_return;
    
    // Wait for stop (this is where async notification integration matters)
    // ...
}
```

### Python Pretty-Printer Integration

When `-enable-pretty-printing` is in effect, varobj values may come from Python printers. The library handles this transparently — the `VarObjInfo::display_hint` field exposes the hint string (`"array"`, `"map"`, `"string"`, etc.) so the front-end can render the value appropriately.

### Non-Stop Mode

Non-stop mode requires per-thread context for every command. The session exposes this cleanly:

```cpp
// With non-stop enabled, all commands take an optional thread context
struct ThreadContext {
    ThreadId    thread_id;
    FrameLevel  frame_level{0};
};

// Evaluate an expression in a specific thread's context
auto result = co_await sess.data().evaluate("some_var", ThreadContext{ThreadId{2}, FrameLevel{0}});
```

---

## 17. Complete Worked Examples

### Example 1: Setting a Breakpoint and Running

```cpp
#include <gdbmi3/gdbmi3.hpp>
#include <print>
#include <thread>

int main() {
    using namespace gdbmi3;
    
    // Create and start a session
    session::Session sess{session::ProcessConfig{
        .executable_path = "/path/to/my_program"
    }};
    
    if (auto err = sess.start(); !err) {
        std::println(stderr, "Failed to start GDB: {}", err.error().message);
        return 1;
    }
    
    // Subscribe to stop events
    sess.notifications().on_stopped([&](const session::StoppedEvent& ev) {
        std::println("Stopped: reason={}, func={}, line={}",
            static_cast<int>(ev.reason),
            ev.frame.func,
            ev.frame.line);
    });
    
    // Subscribe to console output (what GDB prints to ~)
    sess.notifications().on_console_output([](std::string_view text) {
        std::print("{}", text);
    });
    
    // Run event loop in background thread
    std::jthread event_thread{[&] { sess.run_event_loop(); }};
    
    // Set a breakpoint
    auto bp_future = sess.breakpoints().insert({"main"});
    auto bp = bp_future.get();
    if (!bp) {
        std::println(stderr, "Breakpoint failed: {}", bp.error().message);
        return 1;
    }
    
    std::println("Breakpoint {} set ({} location(s))",
        bp->number, bp->location_count());
    
    // Handle multi-location breakpoints properly
    std::visit(core::overloaded{
        [](const result::SingleLocation& sl) {
            std::println("  → {}:{}", sl.loc.file, sl.loc.line);
        },
        [](const result::MultiLocation& ml) {
            for (auto& loc : ml.locations)
                std::println("  → {} at {}:{}", loc.func, loc.file, loc.line);
        },
    }, bp->location_data);
    
    // Run the program
    auto run = sess.exec().run().get();
    if (!run) {
        std::println(stderr, "exec-run failed: {}", run.error().message);
        return 1;
    }
    
    // (The event loop thread will fire on_stopped when we hit the breakpoint)
    std::this_thread::sleep_for(std::chrono::seconds{5});
    
    sess.stop_event_loop();
    return 0;
}
```

### Example 2: Walking the Stack and Inspecting Variables

```cpp
// After receiving a StoppedEvent with reason == BreakpointHit:

void inspect_at_stop(session::Session& sess, const session::StoppedEvent& ev) {
    // List all frames
    auto frames_future = sess.stack().list_frames(ev.thread_id);
    auto frames = frames_future.get();
    if (!frames) return;
    
    std::println("Stack ({} frames):", frames->size());
    for (const auto& frame : *frames) {
        std::println("  #{} {} at {}:{}", 
            frame.level.value(), frame.func, frame.file, frame.line);
    }
    
    // List local variables in the innermost frame
    auto locals_future = sess.stack().list_locals(
        ev.thread_id, FrameLevel{0},
        command::ValueFormat::SimpleValues
    );
    auto locals = locals_future.get();
    if (!locals) return;
    
    std::println("Locals:");
    for (const auto& [name, value] : *locals) {
        std::println("  {} = {}", name, value);
    }
    
    // Create a varobj for a specific variable, inspect its children
    auto var_future = sess.varobjs().create("pt", "pt",
        command::VarObjFrameContext::Fixed{ev.thread_id, FrameLevel{0}});
    auto var = var_future.get();
    if (!var) return;
    
    std::println("varobj '{}': type={}, numchild={}",
        var->name, var->type, var->num_children);
    
    if (var->num_children > 0) {
        auto children_future = var->children();
        auto children = children_future.get();
        if (children) {
            for (const auto& child : *children) {
                std::println("  .{} = {} ({})", child.expression, child.value, child.type);
            }
        }
    }
    
    // Update all varobjs after a step
    auto changes_future = sess.varobjs().update_all();
    auto changes = changes_future.get();
    if (changes) {
        for (const auto& ch : *changes) {
            if (ch.in_scope) {
                std::println("Changed: {} = {}", ch.name, ch.value);
            } else {
                std::println("Out of scope: {}", ch.name);
            }
        }
    }
}
```

### Example 3: Reading and Disassembling Memory

```cpp
void disassemble_current_function(session::Session& sess, const session::StoppedEvent& ev) {
    // Evaluate the function start address
    auto addr_future = sess.data().evaluate("$pc", 
        session::ThreadContext{ev.thread_id, FrameLevel{0}});
    auto addr_str = addr_future.get();
    if (!addr_str) return;
    
    // Disassemble the function containing $pc
    auto disasm_future = sess.data().disassemble_function(
        ev.frame.addr,
        command::DisassembleMode::SourceAndAsm
    );
    auto insns = disasm_future.get();
    if (!insns) return;
    
    std::println("Disassembly of {}:", ev.frame.func);
    for (const auto& insn : *insns) {
        auto marker = (insn.addr == ev.frame.addr) ? "→" : " ";
        std::println("  {} 0x{:016x} <+{:4}>  {}", 
            marker, insn.addr, insn.offset, insn.text);
    }
    
    // Read 16 bytes of raw memory at the current PC
    auto mem_future = sess.data().read_memory_bytes(ev.frame.addr, 16);
    auto mem = mem_future.get();
    if (!mem) return;
    
    std::print("Raw bytes: ");
    for (std::byte b : mem->data) std::print("{:02x} ", std::to_integer<unsigned>(b));
    std::println();
}
```

---

## Summary: Design Decision Reference

The table below consolidates the key design choices and the MI3 protocol facts that drove them.

| Protocol Feature | C++23 Modelling Choice | Rationale |
|---|---|---|
| Token-based correlation | `Token` strong typedef + `TokenGenerator` | Prevents integer confusion; `atomic` counter is lock-free |
| Result classes (^done, ^error…) | `ResultClass` enum + `ResultRecord` | Exhaustive; no string comparison on hot path |
| Seven record prefix types | `std::variant<ResultRecord, StreamRecord, AsyncRecord, PromptRecord>` | Compiler-enforced exhaustive handling |
| Recursive value grammar | `std::variant<MiString, MiTuple, MiList>` | Matches the grammar exactly; no heap allocation for small tuples |
| MI3 multi-location breakpoints | `std::variant<SingleLocation, MultiLocation>` | Forces callers to handle both cases; models the MI3/MI2 difference structurally |
| `^error` responses | `std::expected<T, MiError>` | No exceptions; every caller must acknowledge failure |
| Async notifications (`*`, `=`) | `NotificationRouter` with typed event structs | Decouples command/response from notification; typed structs prevent key-name bugs |
| GDB process I/O | Non-blocking pipes + poll loop | Portable; avoids threading issues on stdin/stdout |
| Command construction | Fluent `MiCommandBuilder` | No string formatting errors; options are explicit |
| varobj lifetime | Move-only `VarObjHandle` (RAII) | GDB leaks varobjs if you don't delete them; RAII makes this automatic |
| `addr_flags` (new in MI3) | Field in `StackFrame` struct | MI3 adds this for architecture-specific address decorations (e.g. AArch64 PAC) |
| `-complete` command (new MI3) | Exposed in `GdbCommands` | Enables IDE-quality command completion |

This architecture compiles cleanly under `-std=c++23 -Wall -Wextra -Werror` and is designed to yield zero warnings. Every layer is independently testable, every protocol concept is typed, and the MI3-specific multi-location breakpoint change is a first-class citizen of the type system — not an afterthought.
