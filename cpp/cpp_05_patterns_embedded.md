# C++ Mastery Guide — Document 5 of 5
## Design Patterns, Error Handling & Embedded C++ Best Practices

> **Scope:** Software design patterns in modern C++, robust error handling, embedded-specific C++ guidelines, performance-oriented design, API design principles, and a comprehensive interview Q&A section. This is the "senior engineer" document — the topics that separate someone who knows C++ syntax from someone who can architect C++ systems.

---

## Table of Contents

1. [Error Handling Strategies](#1-error-handling-strategies)
2. [Creational Design Patterns](#2-creational-design-patterns)
3. [Structural Design Patterns](#3-structural-design-patterns)
4. [Behavioral Design Patterns](#4-behavioral-design-patterns)
5. [State Machine Design in C++](#5-state-machine-design-in-c)
6. [Observer / Signal-Slot Event System](#6-observer--signal-slot-event-system)
7. [Dependency Injection in C++](#7-dependency-injection-in-c)
8. [API Design Principles](#8-api-design-principles)
9. [Embedded C++ Best Practices](#9-embedded-c-best-practices)
10. [Performance-Oriented C++](#10-performance-oriented-c)
11. [Undefined Behavior — The Enemy](#11-undefined-behavior--the-enemy)
12. [Testing C++ Code](#12-testing-c-code)
13. [Common C++ Pitfalls & Interview Traps](#13-common-c-pitfalls--interview-traps)
14. [Code Review Checklist](#14-code-review-checklist)
15. [Final Interview Scenarios & Q&A](#15-final-interview-scenarios--qa)

---

## 1. Error Handling Strategies

### Strategy 1: Exceptions (Standard C++)

```cpp
// Use when: exceptions are enabled, the error is genuinely exceptional
// Pros: Propagates automatically, can't be silently ignored, call sites stay clean
// Cons: Runtime overhead, binary size, disallowed in many embedded environments

class SensorError : public std::runtime_error {
    int sensor_id_;
public:
    SensorError(int id, const std::string& msg)
        : std::runtime_error(msg), sensor_id_(id) {}
    int sensor_id() const noexcept { return sensor_id_; }
};

float read_sensor(int id) {
    if (!sensor_ready(id))
        throw SensorError(id, "sensor not ready");
    return raw_read(id);
}

try {
    float val = read_sensor(3);
} catch (const SensorError& e) {
    log("Sensor {} failed: {}", e.sensor_id(), e.what());
} catch (const std::exception& e) {
    log("Unexpected error: {}", e.what());
}
```

### Strategy 2: Error Codes (Embedded Standard)

```cpp
// Use when: -fno-exceptions, safety-critical, or C interop
// Pros: Zero overhead, deterministic, C-compatible
// Cons: Easy to silently ignore, manual propagation, verbose

enum class ErrCode : int32_t {
    OK           =  0,
    NOT_READY    = -1,
    TIMEOUT      = -2,
    INVALID_ARG  = -3,
    HW_FAULT     = -4,
    OVERFLOW     = -5,
};

// Output parameter pattern:
ErrCode read_sensor(int id, float* out) {
    if (!out)             return ErrCode::INVALID_ARG;
    if (!sensor_ready(id)) return ErrCode::NOT_READY;
    *out = raw_read(id);
    return ErrCode::OK;
}

float val;
if (auto err = read_sensor(3, &val); err != ErrCode::OK) {
    log("error: {}", static_cast<int>(err));
    return err;  // manually bubble up
}
```

### Strategy 3: `std::optional` — Value or Nothing

```cpp
// Use when: absence is normal and expected, no error detail needed
// Zero overhead, type-safe, clean call sites

std::optional<float> read_sensor(int id) {
    if (!sensor_ready(id)) return std::nullopt;
    return raw_read(id);
}

auto val = read_sensor(3);
if (val) {
    process(*val);
} else {
    // sensor simply not available right now
}

// With a default:
float temperature = read_sensor(3).value_or(25.0f);
```

### Strategy 4: `Expected<T, E>` — The Best of Both Worlds

This pattern (Rust's `Result<T, E>`) is ideal for embedded C++ without exceptions. C++23 standardizes it as `std::expected`. For C++17/20, implement it or use a library (tl::expected, etc.):

```cpp
// C++23 standard version:
#include <expected>

std::expected<float, ErrCode> read_sensor(int id) {
    if (!sensor_ready(id))
        return std::unexpected(ErrCode::NOT_READY);
    return raw_read(id);  // implicit success
}

auto result = read_sensor(3);
if (result) {
    process(*result);           // dereference for value
} else {
    log("code {}", static_cast<int>(result.error()));
}

// Chaining without exceptions:
auto final = read_sensor(3)
    .and_then([](float v) -> std::expected<float, ErrCode> {
        if (v > 150.0f) return std::unexpected(ErrCode::HW_FAULT);
        return v * 1.8f + 32.0f;  // to Fahrenheit
    })
    .transform([](float f) { return std::round(f); });

// C++17 manual implementation sketch:
template<typename T, typename E>
class Expected {
    union { T val_; E err_; };
    bool ok_;
public:
    static Expected ok(T v)  { Expected e; e.ok_ = true;  new(&e.val_) T(std::move(v)); return e; }
    static Expected err(E e) { Expected x; x.ok_ = false; new(&x.err_) E(std::move(e)); return x; }
    explicit operator bool() const { return ok_; }
    T& value()               { assert(ok_);  return val_; }
    E& error()               { assert(!ok_); return err_; }
    ~Expected() { if (ok_) val_.~T(); else err_.~E(); }
};
```

### Strategy 5: Assert / Panic (Programming Errors)

```cpp
// Use when: the error indicates a programming bug, not a runtime condition
// In embedded: often triggers a system reset or logs a crash dump

[[noreturn]] void panic(const char* file, int line, const char* msg) noexcept {
    uart_write_emergency(file, line, msg);   // log before dying
    NVIC_SystemReset();                       // reset the MCU
    __builtin_unreachable();
}

#define ASSERT(cond) \
    do { if (!(cond)) panic(__FILE__, __LINE__, #cond); } while(0)

void set_baud_rate(uint32_t baud) {
    ASSERT(baud > 0 && baud <= 10'000'000);  // programming error if violated
    // ... configure hardware
}
```

### Choosing the Right Strategy

| Situation | Strategy |
|---|---|
| Exceptions enabled, truly exceptional error | `throw` |
| No detail needed, absence is normal | `std::optional` |
| Need error code, no exceptions | `Expected<T, E>` |
| Simple, C-interop, no exceptions | Error code enum |
| Programming error (should never happen) | `ASSERT` / panic |
| Safety-critical, no recovery possible | Controlled reset |

---

## 2. Creational Design Patterns

### Factory Method

```cpp
// Create objects without specifying the concrete class

class IDriver {
public:
    virtual ~IDriver() = default;
    virtual bool    init()                              = 0;
    virtual ssize_t write(const uint8_t* buf, size_t n) = 0;
    virtual ssize_t read(uint8_t* buf, size_t n)        = 0;
};

class UartDriver : public IDriver { /* UART implementation */ };
class SpiDriver  : public IDriver { /* SPI  implementation */ };
class I2cDriver  : public IDriver { /* I2C  implementation */ };

enum class BusType { UART, SPI, I2C };

// Factory: returns the right concrete type behind the interface
std::unique_ptr<IDriver> make_driver(BusType type) {
    switch (type) {
        case BusType::UART: return std::make_unique<UartDriver>();
        case BusType::SPI:  return std::make_unique<SpiDriver>();
        case BusType::I2C:  return std::make_unique<I2cDriver>();
    }
    return nullptr;
}

// Client code never knows which concrete type it has:
auto drv = make_driver(BusType::UART);
drv->init();
```

### Abstract Factory

```cpp
// Create families of related objects — critical for HAL abstraction in embedded

// Interface family:
class ITimer { public: virtual void start(uint32_t ms) = 0; virtual ~ITimer() = default; };
class IGpio  { public: virtual void set(bool v)        = 0; virtual ~IGpio()  = default; };
class IUart  { public: virtual void write(uint8_t b)   = 0; virtual ~IUart()  = default; };

// Abstract factory: creates a consistent set
class IHalFactory {
public:
    virtual ~IHalFactory() = default;
    virtual std::unique_ptr<ITimer> create_timer()          = 0;
    virtual std::unique_ptr<IGpio>  create_gpio(int pin)    = 0;
    virtual std::unique_ptr<IUart>  create_uart(uint32_t b) = 0;
};

// Real hardware factory:
class Imx6HalFactory : public IHalFactory {
    std::unique_ptr<ITimer> create_timer()          override { return std::make_unique<Imx6Timer>(); }
    std::unique_ptr<IGpio>  create_gpio(int pin)    override { return std::make_unique<Imx6Gpio>(pin); }
    std::unique_ptr<IUart>  create_uart(uint32_t b) override { return std::make_unique<Imx6Uart>(b); }
};

// Test factory — no real hardware needed:
class MockHalFactory : public IHalFactory {
    std::unique_ptr<ITimer> create_timer()          override { return std::make_unique<MockTimer>(); }
    std::unique_ptr<IGpio>  create_gpio(int pin)    override { return std::make_unique<MockGpio>(pin); }
    std::unique_ptr<IUart>  create_uart(uint32_t b) override { return std::make_unique<MockUart>(); }
};

// System works with either factory — no changes:
class System {
    std::unique_ptr<ITimer> timer_;
    std::unique_ptr<IGpio>  status_led_;
public:
    explicit System(IHalFactory& hal)
        : timer_(hal.create_timer())
        , status_led_(hal.create_gpio(LED_PIN)) {}
};
```

### Builder Pattern

```cpp
// Problem: constructors with many optional parameters become unreadable
// Solution: method chaining builder

class UartConfig {
public:
    uint32_t    baud_rate    = 115200;
    uint8_t     data_bits    = 8;
    char        parity       = 'N';
    uint8_t     stop_bits    = 1;
    bool        flow_control = false;
    uint32_t    timeout_ms   = 1000;
    std::string device       = "/dev/ttyS0";

    UartConfig& with_baud(uint32_t b)     { baud_rate = b;          return *this; }
    UartConfig& with_parity(char p)       { parity = p;             return *this; }
    UartConfig& with_timeout(uint32_t ms) { timeout_ms = ms;        return *this; }
    UartConfig& with_device(std::string d){ device = std::move(d);  return *this; }
    UartConfig& with_flow_control()       { flow_control = true;    return *this; }
};

// Fluent, self-documenting:
auto cfg = UartConfig{}
    .with_baud(9600)
    .with_parity('E')
    .with_device("/dev/ttyUSB0")
    .with_timeout(500);

// C++20 designated initializers as lighter-weight alternative:
UartConfig cfg2{ .baud_rate = 9600, .parity = 'E', .timeout_ms = 500 };
```

### Singleton (Use Sparingly)

```cpp
// Use ONLY for truly global, unique resources (e.g., a crash logger)
// Singletons are hidden dependencies — they make testing hard

class CrashLogger {
public:
    static CrashLogger& instance() {
        static CrashLogger inst;  // C++11: guaranteed thread-safe (magic static)
        return inst;
    }

    void log(const char* msg, const char* file, int line) {
        std::lock_guard lock(mutex_);
        // write to persistent storage before reset
        write_to_flash(msg, file, line);
    }

    CrashLogger(const CrashLogger&) = delete;
    CrashLogger& operator=(const CrashLogger&) = delete;

private:
    CrashLogger() = default;
    std::mutex mutex_;
};

CrashLogger::instance().log("watchdog expired", __FILE__, __LINE__);
```

---

## 3. Structural Design Patterns

### Adapter Pattern

```cpp
// Make an incompatible interface work with the expected interface

// What the rest of the system expects:
class IStream {
public:
    virtual ~IStream() = default;
    virtual ssize_t read(uint8_t* buf, size_t len)        = 0;
    virtual ssize_t write(const uint8_t* buf, size_t len) = 0;
};

// Adapts a raw POSIX file descriptor to IStream:
class FdStream : public IStream {
    int fd_;
public:
    explicit FdStream(int fd) : fd_(fd) {}
    ssize_t read(uint8_t* buf, size_t len) override {
        return ::read(fd_, buf, len);
    }
    ssize_t write(const uint8_t* buf, size_t len) override {
        return ::write(fd_, buf, len);
    }
};

// Now POSIX fd works anywhere IStream is expected:
void process_stream(IStream& s) { /* ... */ }
FdStream uart_fd(open("/dev/ttyS0", O_RDWR));
process_stream(uart_fd);
```

### Decorator Pattern

```cpp
// Add behavior to an object without modifying its class
// Wraps the original, delegates, then adds extra behavior

class IDriver {
public:
    virtual ~IDriver() = default;
    virtual ssize_t write(const uint8_t* buf, size_t len) = 0;
};

class UartDriver : public IDriver {
    ssize_t write(const uint8_t* buf, size_t len) override { /* raw write */ return len; }
};

// Decorator 1: adds logging
class LoggingDriver : public IDriver {
    IDriver& inner_;
    const char* tag_;
public:
    LoggingDriver(IDriver& inner, const char* tag) : inner_(inner), tag_(tag) {}
    ssize_t write(const uint8_t* buf, size_t len) override {
        log("[{}] writing {} bytes", tag_, len);
        auto r = inner_.write(buf, len);
        log("[{}] wrote {} bytes", tag_, r);
        return r;
    }
};

// Decorator 2: adds CRC
class CrcDriver : public IDriver {
    IDriver& inner_;
public:
    explicit CrcDriver(IDriver& inner) : inner_(inner) {}
    ssize_t write(const uint8_t* buf, size_t len) override {
        inner_.write(buf, len);
        uint16_t crc = compute_crc16(buf, len);
        inner_.write(reinterpret_cast<const uint8_t*>(&crc), sizeof(crc));
        return static_cast<ssize_t>(len);
    }
};

// Stack them — order matters:
UartDriver    raw;
LoggingDriver logged(raw, "UART0");
CrcDriver     crc_logged(logged);
crc_logged.write(payload, payload_size);  // CRC → Logging → Raw UART
```

### Proxy Pattern

```cpp
// Control access to an object: lazy init, access control, caching

// Lazy initialization proxy — only creates the real object on first use:
class ExpensiveSensor {
public:
    float read() { /* slow hardware init + read */ return 0.0f; }
};

class LazySensor {
    mutable std::unique_ptr<ExpensiveSensor> sensor_;
    mutable std::once_flag                   flag_;

    void ensure_init() const {
        std::call_once(flag_, [this]{
            sensor_ = std::make_unique<ExpensiveSensor>();
        });
    }
public:
    float read() const { ensure_init(); return sensor_->read(); }
};

// Thread-safe: std::once_flag guarantees init runs exactly once
LazySensor s;
s.read();  // sensor initialized here, on first access
s.read();  // reuses initialized sensor
```

---

## 4. Behavioral Design Patterns

### Command Pattern

```cpp
// Encapsulate a request as an object: queue it, log it, undo it

class ICommand {
public:
    virtual ~ICommand() = default;
    virtual void execute() = 0;
    virtual void undo()    {}
};

class SetGainCommand : public ICommand {
    AudioEngine& engine_;
    float new_gain_, old_gain_;
public:
    SetGainCommand(AudioEngine& e, float g)
        : engine_(e), new_gain_(g), old_gain_(e.gain()) {}
    void execute() override { engine_.set_gain(new_gain_); }
    void undo()    override { engine_.set_gain(old_gain_); }
};

// Command history for undo:
class CommandHistory {
    std::stack<std::unique_ptr<ICommand>> history_;
public:
    void execute(std::unique_ptr<ICommand> cmd) {
        cmd->execute();
        history_.push(std::move(cmd));
    }
    void undo() {
        if (!history_.empty()) {
            history_.top()->undo();
            history_.pop();
        }
    }
};

// For IOC protocol: use std::function for lightweight commands (no vtable):
using Command = std::function<void()>;
std::queue<Command> ioc_cmd_queue;
ioc_cmd_queue.push([&ioc]{ ioc.send_shutdown_ack(); });
ioc_cmd_queue.push([&gpio]{ gpio.set_power_good(false); });
```

### Strategy Pattern

```cpp
// Select algorithm at compile time (zero cost) or runtime

// Compile-time strategy — policy-based (preferred for hot paths):
struct NoChecksum    { uint16_t compute(const uint8_t*, size_t) { return 0;  } };
struct Crc16Checksum { uint16_t compute(const uint8_t* d, size_t n) { return crc16(d,n); } };

template<typename ChecksumPolicy>
class Serializer {
    ChecksumPolicy cs_;
public:
    std::vector<uint8_t> serialize(const Message& msg) {
        auto payload = msg.to_bytes();
        uint16_t cs = cs_.compute(payload.data(), payload.size());
        payload.push_back(cs >> 8);
        payload.push_back(cs & 0xFF);
        return payload;
    }
};

Serializer<NoChecksum>    fast_ser;   // zero checksum overhead
Serializer<Crc16Checksum> safe_ser;   // CRC protected

// Runtime strategy — when algorithm must be selected at runtime:
class IChecksumStrategy {
public:
    virtual ~IChecksumStrategy() = default;
    virtual uint16_t compute(const uint8_t* data, size_t len) = 0;
};

class RuntimeSerializer {
    std::unique_ptr<IChecksumStrategy> strategy_;
public:
    void set_strategy(std::unique_ptr<IChecksumStrategy> s) {
        strategy_ = std::move(s);
    }
};
```

### Template Method Pattern

```cpp
// Base class defines the algorithm skeleton; subclasses fill in specific steps

class DiagnosticRun {
public:
    // The template method — algorithm structure is fixed here
    void run() {
        if (!prepare())          { report(Status::SETUP_FAILED); return; }
        auto results = run_tests();
        generate_report(results);
        cleanup();
    }

protected:
    virtual bool                   prepare()     = 0;
    virtual std::vector<TestResult> run_tests()  = 0;
    virtual void                   cleanup()     {}  // optional hook

private:
    void generate_report(const std::vector<TestResult>& r) {
        // shared, non-overridable: always the same format
        for (auto& res : r) std::cout << res.name << ": " << (res.pass ? "PASS":"FAIL") << "\n";
    }
    void report(Status s) { std::cout << "diagnostic failed at setup\n"; }
};

class UartDiagnostic : public DiagnosticRun {
    bool prepare()   override { return init_uart_loopback(); }
    std::vector<TestResult> run_tests() override {
        return { test_tx_rx(), test_baud_rates(), test_framing() };
    }
    void cleanup()   override { deinit_uart(); }
};
```

---

## 5. State Machine Design in C++

### Switch-Case: Simple and Fast

```cpp
enum class UartState { Idle, Receiving, Processing, Error };

class UartFramer {
    UartState state_  = UartState::Idle;
    std::vector<uint8_t> buf_;

public:
    void feed(uint8_t byte) {
        switch (state_) {
            case UartState::Idle:
                if (byte == 0xAA) { buf_.clear(); state_ = UartState::Receiving; }
                break;

            case UartState::Receiving:
                if (byte == 0x55) {
                    state_ = UartState::Processing;
                    dispatch(buf_);
                    state_ = UartState::Idle;
                } else if (buf_.size() >= MAX_FRAME) {
                    state_ = UartState::Error;
                } else {
                    buf_.push_back(byte);
                }
                break;

            case UartState::Error:
                if (byte == 0xFF) state_ = UartState::Idle;  // reset sequence
                break;

            default: break;
        }
    }
};
```

### `std::variant` State Machine — Type-Safe, No Heap

Each state is its own type. Illegal data access between states is a compile error:

```cpp
#include <variant>

// States carry only their own relevant data:
struct Idle    {};
struct Receiving { std::vector<uint8_t> buf; };
struct Processing{ std::vector<uint8_t> packet; size_t bytes_processed = 0; };
struct ErrorState{ int code; };

using State = std::variant<Idle, Receiving, Processing, ErrorState>;

class Protocol {
    State state_ = Idle{};

public:
    void on_byte(uint8_t byte) {
        std::visit([&](auto& s) {
            using S = std::decay_t<decltype(s)>;
            if      constexpr (std::is_same_v<S, Idle>) {
                if (byte == START) state_ = Receiving{};
            }
            else if constexpr (std::is_same_v<S, Receiving>) {
                if (byte == END) {
                    state_ = Processing{std::move(s.buf)};
                } else if (s.buf.size() > MAX_PKT) {
                    state_ = ErrorState{ERR_OVERFLOW};
                } else {
                    s.buf.push_back(byte);
                }
            }
            else if constexpr (std::is_same_v<S, Processing>) {
                process_packet(s.packet);
                state_ = Idle{};
            }
            else if constexpr (std::is_same_v<S, ErrorState>) {
                if (byte == RESET) state_ = Idle{};
            }
        }, state_);
    }

    bool is_idle() const { return std::holds_alternative<Idle>(state_); }
};
```

### Transition Table State Machine — Most Maintainable

```cpp
enum class State { Idle, Running, Paused, Stopped };
enum class Event { Start, Pause, Resume, Stop, Error };

struct Transition {
    State from;
    Event on;
    State to;
    std::function<void()> action;  // fired on transition
};

class Fsm {
    State current_;
    std::vector<Transition> table_;

public:
    explicit Fsm(State init) : current_(init) {}

    Fsm& add(State from, Event on, State to, std::function<void()> action = {}) {
        table_.push_back({from, on, to, std::move(action)});
        return *this;
    }

    bool process(Event e) {
        for (auto& t : table_) {
            if (t.from == current_ && t.on == e) {
                if (t.action) t.action();
                current_ = t.to;
                return true;
            }
        }
        return false;  // no transition: event ignored or error
    }

    State state() const { return current_; }
};

// Setup reads like a spec:
Fsm fsm(State::Idle);
fsm.add(State::Idle,    Event::Start,  State::Running, [&]{ start_hw();  })
   .add(State::Running, Event::Pause,  State::Paused,  [&]{ pause_hw();  })
   .add(State::Running, Event::Stop,   State::Stopped, [&]{ stop_hw();   })
   .add(State::Paused,  Event::Resume, State::Running, [&]{ resume_hw(); })
   .add(State::Paused,  Event::Stop,   State::Stopped, [&]{ stop_hw();   });
```

---

## 6. Observer / Signal-Slot Event System

```cpp
// Decouple producers from consumers of events

template<typename... Args>
class Signal {
    using Handler = std::function<void(Args...)>;
    std::vector<Handler> handlers_;
    mutable std::mutex   mutex_;

public:
    // Subscribe:
    void connect(Handler h) {
        std::lock_guard lock(mutex_);
        handlers_.push_back(std::move(h));
    }

    // Emit to all subscribers:
    void emit(Args... args) const {
        std::lock_guard lock(mutex_);
        for (auto& h : handlers_) h(args...);
    }
};

// In a power manager:
class PowerManager {
public:
    Signal<bool>          on_ignition_changed;  // bool: is_on
    Signal<PowerState>    on_state_changed;
    Signal<WakeupSource>  on_wakeup;

    void process_ignition_on() {
        state_ = PowerState::Active;
        on_ignition_changed.emit(true);
        on_state_changed.emit(PowerState::Active);
    }
};

// Subscribers are completely decoupled from the power manager:
class DisplayController {
public:
    DisplayController(PowerManager& pm) {
        pm.on_state_changed.connect([this](PowerState s) {
            if (s == PowerState::Sleep) turn_off();
            else if (s == PowerState::Active) turn_on();
        });
    }
};

class AudioController {
public:
    AudioController(PowerManager& pm) {
        pm.on_state_changed.connect([this](PowerState s) {
            if (s == PowerState::Sleep) mute();
        });
    }
};

// DisplayController and AudioController don't know about each other
// Adding a third subscriber requires no changes to any existing code
```

---

## 7. Dependency Injection in C++

```cpp
// Hard-coded dependencies = untestable, inflexible code

// BAD: hidden dependencies
class Protocol_Bad {
    UartDriver driver_{"/dev/ttyS0"};  // can't swap for tests
    SysLogger  logger_{"syslog"};      // can't redirect in tests
};

// GOOD: constructor injection (preferred)
class Protocol {
    IUartDriver& driver_;
    ILogger&     logger_;
public:
    Protocol(IUartDriver& drv, ILogger& log) : driver_(drv), logger_(log) {}

    void send(const uint8_t* data, size_t len) {
        logger_.info("sending {} bytes", len);
        driver_.write(data, len);
    }
};

// Production wiring:
RealUartDriver uart("/dev/ttyS0");
SysLogger      logger;
Protocol       proto(uart, logger);

// Test wiring — zero real hardware needed:
MockUartDriver mock_uart;
NullLogger     null_log;
Protocol       proto_test(mock_uart, null_log);

// Template DI — zero virtual dispatch overhead (for hot paths):
template<typename Driver, typename Logger>
class ProtocolT {
    Driver& drv_;
    Logger& log_;
public:
    ProtocolT(Driver& d, Logger& l) : drv_(d), log_(l) {}
    void send(const uint8_t* data, size_t len) {
        log_.info("sending {} bytes", len);  // fully inlined by compiler
        drv_.write(data, len);               // fully inlined
    }
};

// Wired at compile time — both policies completely inlined:
ProtocolT<RealUartDriver, SysLogger>    prod_proto(uart, logger);
ProtocolT<MockUartDriver, NullLogger>   test_proto(mock_uart, null_log);
```

---

## 8. API Design Principles

### Make Interfaces Hard to Misuse

```cpp
// BAD: which is width, which is height? Easy to swap:
void set_window(int x, int y, int w, int h);
set_window(800, 600, 10, 20);  // intended: 800 wide, 600 tall OR position?

// GOOD: strong types eliminate argument swapping errors
struct PixelX     { int value; };
struct PixelY     { int value; };
struct PixelWidth { int value; };
struct PixelHeight{ int value; };

void set_window(PixelX x, PixelY y, PixelWidth w, PixelHeight h);
set_window(PixelX{10}, PixelY{20}, PixelWidth{800}, PixelHeight{600});
// Swapping width and height: compile error!
```

### `[[nodiscard]]` Everything That Matters

```cpp
// If ignoring the return value is almost always a bug, mark it nodiscard:

[[nodiscard]] ErrCode         init_hardware();
[[nodiscard]] std::optional<int> try_read();
[[nodiscard]] bool            write_register(uint32_t addr, uint32_t val);

// C++20: add a message
[[nodiscard("check for hardware error before proceeding")]]
ErrCode configure_uart(const UartConfig&);

// Callers get a warning if they ignore the result:
// init_hardware();        // WARNING: ignoring nodiscard value
auto err = init_hardware(); // OK: checking the result
```

### `explicit` on Single-Argument Constructors

```cpp
class BufferSize {
    size_t bytes_;
public:
    // Without explicit: BufferSize b = 42; would silently compile
    // With explicit: must be intentional
    explicit BufferSize(size_t bytes) : bytes_(bytes) {}
};

void allocate(BufferSize sz);
// allocate(4096);         // ERROR: no implicit conversion
allocate(BufferSize{4096}); // GOOD: intent is clear
```

### Design for Testability

```cpp
// Every component should:
// 1. Depend on interfaces, not concretions
// 2. Receive dependencies via constructor (not create them internally)
// 3. Have no hidden global state
// 4. Be deterministic (inject time, random, etc. — don't call them directly)

class ITimeSource {
public:
    virtual ~ITimeSource() = default;
    virtual uint64_t now_ms() const = 0;
};

class Timeout {
    const ITimeSource& time_;
    uint64_t start_ms_;
    uint64_t duration_ms_;
public:
    Timeout(const ITimeSource& ts, uint64_t ms)
        : time_(ts), start_ms_(ts.now_ms()), duration_ms_(ms) {}
    bool expired() const { return (time_.now_ms() - start_ms_) >= duration_ms_; }
};

// Test with fake time — no sleep() needed in tests:
class FakeTime : public ITimeSource {
    uint64_t t_ = 0;
public:
    void advance(uint64_t ms) { t_ += ms; }
    uint64_t now_ms() const override { return t_; }
};

FakeTime ft;
Timeout t(ft, 500);
assert(!t.expired());
ft.advance(400);
assert(!t.expired());
ft.advance(101);
assert(t.expired());   // test runs in microseconds, no real waiting
```

---

## 9. Embedded C++ Best Practices

### Avoid Dynamic Allocation After Startup

```cpp
// In long-running embedded systems heap fragmentation causes allocation failure
// All heap allocation should happen once during initialization

// Strategy 1: Static arrays
static std::array<uint8_t, 4096> uart_rx_buf;
static std::array<uint8_t, 4096> uart_tx_buf;

// Strategy 2: Fixed-size object pool (no heap fragmentation)
template<typename T, size_t N>
class ObjectPool {
    alignas(T) char storage_[sizeof(T) * N];
    std::bitset<N>  used_;
public:
    template<typename... Args>
    T* construct(Args&&... args) {
        for (size_t i = 0; i < N; i++) {
            if (!used_[i]) {
                used_[i] = true;
                return new(storage_ + i * sizeof(T)) T(std::forward<Args>(args)...);
            }
        }
        return nullptr;  // pool exhausted
    }
    void destroy(T* p) {
        p->~T();
        size_t i = (reinterpret_cast<char*>(p) - storage_) / sizeof(T);
        used_[i] = false;
    }
};

ObjectPool<Packet, 32> packet_pool;  // 32 pre-allocated packet slots
auto* pkt = packet_pool.construct(data, len);
// ...
packet_pool.destroy(pkt);  // returns to pool, no heap involved

// Strategy 3: std::array instead of std::vector for fixed-max collections
std::array<Sensor*, 16> active_sensors{};
size_t sensor_count = 0;
```

### `volatile` for Hardware Registers — Correctly

```cpp
// ALWAYS use volatile for MMIO:
volatile uint32_t* const UART_DR  = reinterpret_cast<volatile uint32_t*>(0x40013804);
volatile uint32_t* const UART_SR  = reinterpret_cast<volatile uint32_t*>(0x40013800);

// Struct of volatile registers:
struct Uart {
    volatile uint32_t SR;   // Status register  — offset 0x00
    volatile uint32_t DR;   // Data register    — offset 0x04
    volatile uint32_t BRR;  // Baud rate reg    — offset 0x08
    volatile uint32_t CR1;  // Control reg 1    — offset 0x0C
};
Uart* const USART1 = reinterpret_cast<Uart*>(0x40013800);

// Correct read-modify-write (volatile ensures each read/write is a real access):
USART1->CR1 |=  (1u << 13);  // Set UE bit
USART1->CR1 &= ~(1u << 5);   // Clear RXNEIE bit

// DO NOT use volatile for thread synchronization:
volatile bool ready = false;  // WRONG for thread sync — no memory ordering
std::atomic<bool> ready_atomic{false};  // CORRECT — use atomic
```

### ISR Best Practices

```cpp
// ISR = Interrupt Service Routine: must be fast and non-blocking

// WRONG: heavy work in ISR
extern "C" void USART1_IRQHandler() {
    auto data = parse_protocol(USART1->DR);  // WRONG: slow parsing in ISR
    write_to_database(data);                  // WRONG: blocking operation
}

// CORRECT: minimal ISR, defer to task
static SpscQueue<uint8_t, 256> uart_rx_queue;  // lock-free, ISR-safe

extern "C" void USART1_IRQHandler() {
    if (USART1->SR & RXNE_BIT) {
        uint8_t byte = static_cast<uint8_t>(USART1->DR);
        uart_rx_queue.push(byte);  // non-blocking, lock-free
    }
    // Return immediately: < 50ns total
}

// Processing task runs at lower priority:
void uart_task() {
    while (true) {
        uint8_t byte;
        while (uart_rx_queue.pop(byte)) {
            protocol_feed(byte);  // safe: not in ISR context
        }
        wait_for_notification();  // sleep until ISR signals more data
    }
}

// Rules for ISR code:
// ✅ Read hardware registers
// ✅ Write to lock-free queues or set atomic flags
// ✅ Signal an RTOS task with FromISR variants
// ❌ No dynamic memory allocation (malloc is not re-entrant)
// ❌ No mutexes (can block, risk of priority inversion)
// ❌ No printf / std::cout (not async-signal-safe)
// ❌ No floating point (unless FPU state explicitly saved)
// ❌ No long loops or blocking operations
```

### Avoid These Common Embedded C++ Mistakes

```cpp
// 1. Using std::string in ISR or tight real-time loop
//    std::string heap-allocates, unpredictable latency
//    Use: std::string_view, std::array<char, N>, fixed-capacity string

// 2. Using std::vector in fixed-size contexts
//    std::vector heap-allocates on push when capacity exceeded
//    Use: std::array<T, N> or a fixed-capacity vector

// 3. Exceptions in callbacks or constructors of global objects
//    Global constructors run before main() — exception propagation may fail
//    Use: two-phase init (construct + explicit init() call)

// 4. Global/static non-trivial objects with non-trivial constructors
//    Static initialization order fiasco: order across TUs is undefined
//    Use: function-local statics (Meyers singleton), or constexpr

// 5. Forgetting noexcept on destructors and move operations
//    Without noexcept on move ctor: std::vector copies instead of moves
//    Without noexcept on dtor: double exception = terminate()
```

---

## 10. Performance-Oriented C++

### Data-Oriented Design (Cache-Friendly)

```cpp
// CPU cache line = 64 bytes. Data must be in cache to be fast.

// BAD: Array of Structures — one sensor per cache line (lots of waste)
struct Sensor_AoS {
    uint32_t id;        // 4 bytes
    float    value;     // 4 bytes
    bool     active;    // 1 byte
    char     name[55];  // 55 bytes — padding to 64 bytes
    // reading 1000 values: 1000 cache misses, 1000 cache lines loaded
};
std::array<Sensor_AoS, 1000> sensors;
for (auto& s : sensors) total += s.value;  // 1000 cache line loads

// GOOD: Structure of Arrays — values are contiguous
struct Sensors_SoA {
    std::array<uint32_t, 1000> id;
    std::array<float,    1000> value;
    std::array<bool,     1000> active;
    // ... name in a separate array if needed
};
Sensors_SoA sensors;
for (float v : sensors.value) total += v;  // ~16 cache lines for 1000 floats (4KB/64B)
// SIMD can process 8 floats per instruction with AVX
```

### Move Semantics for Performance

```cpp
// Always move when transferring large objects out of functions:
std::vector<uint8_t> build_packet(const Message& msg) {
    std::vector<uint8_t> pkt;
    pkt.reserve(msg.serialized_size() + 4);  // pre-allocate
    serialize_header(pkt);
    serialize_body(msg, pkt);
    return pkt;  // NRVO or implicit move — no copy
}

// emplace_back vs push_back:
std::vector<std::string> names;
names.push_back(std::string("hello"));  // creates temp string, then moves into vector
names.emplace_back("hello");            // constructs string directly in-place (better)

// Reserve before filling a vector to avoid repeated reallocations:
std::vector<int> v;
v.reserve(1000);      // one allocation up front
for (int i = 0; i < 1000; i++) v.push_back(i);  // no reallocations
```

### Avoiding Virtual Dispatch in Hot Paths

```cpp
// Virtual call: ~1-5ns + kills inlining + icache pressure
// In audio: 44100 calls/sec × 5ns = 220µs wasted per second — significant

// Use CRTP for compile-time polymorphism on known types:
template<typename Derived>
class AudioFilter {
public:
    float process(float sample) {
        // Calls derived implementation without virtual dispatch:
        return static_cast<Derived*>(this)->process_impl(sample);
    }
};

class LowPassFilter : public AudioFilter<LowPassFilter> {
    float state_ = 0.0f, alpha_;
public:
    explicit LowPassFilter(float cutoff) : alpha_(cutoff) {}
    float process_impl(float x) {
        state_ = alpha_ * x + (1.0f - alpha_) * state_;
        return state_;
    }
};

// Hot loop: fully inlined, no vtable
template<typename F>
void process_audio_block(F& filter, float* block, size_t n) {
    for (size_t i = 0; i < n; i++) block[i] = filter.process(block[i]);
}

LowPassFilter lpf(0.1f);
process_audio_block(lpf, audio_buf, 512);  // zero virtual dispatch overhead
```

---

## 11. Undefined Behavior — The Enemy

UB is behavior the C++ standard doesn't define. The compiler assumes UB never happens and optimizes accordingly — producing silently wrong code.

### Most Critical UB in Embedded

```cpp
// 1. Signed integer overflow — compiler ASSUMES it never happens
int find_last(int* arr, int n) {
    for (int i = 0; i <= n; i++) {      // if n = INT_MAX, i++ overflows
        if (arr[i] == 0) return i;       // compiler may eliminate overflow check!
    }
    return -1;
}
// Fix: use unsigned, or check before incrementing

// 2. Strict aliasing violation — accessing memory through wrong type
float f = 3.14f;
uint32_t bits = *reinterpret_cast<uint32_t*>(&f);  // UB: aliasing violation
// Fix: use memcpy for type punning
uint32_t bits_safe;
std::memcpy(&bits_safe, &f, sizeof(bits_safe));     // well-defined

// 3. Null pointer dereference
void process(const Config* cfg) {
    cfg->timeout;  // UB if cfg is null
}
// Fix: assert(cfg != nullptr) or use reference parameter

// 4. Data race on non-atomic variable
int counter = 0;        // two threads read-modify-write: UB!
std::atomic<int> safe_counter{0};  // Fix: use atomic

// 5. Use after free
auto* p = new Sensor();
delete p;
p->read();    // UB: memory might have been reused
// Fix: unique_ptr makes this impossible

// 6. Uninitialized read
int x;        // uninitialized
if (x > 0) { /* UB: reading indeterminate value */ }
// Fix: always initialize: int x = 0;

// 7. Shift by >= bit width
uint32_t x = 1u << 32;  // UB! 32 >= 32 (bit width of uint32_t)
uint64_t y = 1ULL << 32; // OK: shift within 64-bit type

// Detecting UB: compile with sanitizers during testing
// -fsanitize=undefined  catches most UB at runtime
// -fsanitize=address    catches memory errors
// -fsanitize=thread     catches data races
```

---

## 12. Testing C++ Code

### Test Structure with Google Test

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>

// Mock class using GMock:
class MockUartDriver : public IUartDriver {
public:
    MOCK_METHOD(bool,    init,  (uint32_t baud),              (override));
    MOCK_METHOD(ssize_t, write, (const uint8_t*, size_t),     (override));
    MOCK_METHOD(ssize_t, read,  (uint8_t*, size_t, int),      (override));
};

// Test fixture:
class ProtocolTest : public ::testing::Test {
protected:
    MockUartDriver mock_driver_;
    Protocol       proto_{mock_driver_};  // inject mock

    void SetUp() override {
        EXPECT_CALL(mock_driver_, init(115200)).WillOnce(testing::Return(true));
        proto_.init(115200);
    }
};

// Test cases:
TEST_F(ProtocolTest, SendsStartByte) {
    EXPECT_CALL(mock_driver_, write(testing::_, testing::_))
        .WillOnce([](const uint8_t* buf, size_t len) -> ssize_t {
            EXPECT_EQ(buf[0], 0xAA);  // start byte
            return static_cast<ssize_t>(len);
        });
    proto_.send({0x01, 0x02, 0x03});
}

TEST_F(ProtocolTest, ReturnsErrorOnWriteFailure) {
    EXPECT_CALL(mock_driver_, write(testing::_, testing::_))
        .WillOnce(testing::Return(-1));
    auto result = proto_.send({0x01});
    EXPECT_FALSE(result);
    EXPECT_EQ(result.error(), ErrCode::HW_FAULT);
}
```

### Hand-Rolled Mock (No Framework)

```cpp
// Sometimes simpler than GMock — good for embedded where test frameworks are limited

class MockUartDriver : public IUartDriver {
public:
    // Recorded calls for verification:
    std::vector<std::vector<uint8_t>> write_calls;
    std::vector<uint8_t>              next_read_data;
    bool                              init_returns = true;

    bool    init(uint32_t) override { return init_returns; }
    ssize_t write(const uint8_t* buf, size_t n) override {
        write_calls.emplace_back(buf, buf + n);
        return static_cast<ssize_t>(n);
    }
    ssize_t read(uint8_t* buf, size_t n, int) override {
        size_t count = std::min(n, next_read_data.size());
        std::memcpy(buf, next_read_data.data(), count);
        next_read_data.erase(next_read_data.begin(), next_read_data.begin() + count);
        return static_cast<ssize_t>(count);
    }

    void verify_wrote(const std::vector<uint8_t>& expected, size_t call_idx = 0) {
        assert(write_calls.size() > call_idx);
        assert(write_calls[call_idx] == expected);
    }
};
```

---

## 13. Common C++ Pitfalls & Interview Traps

### Trap 1: Object Slicing

```cpp
class Animal { public: virtual std::string speak() const = 0; };
class Dog    : public Animal { public: std::string speak() const override { return "Woof"; } };

Dog  d;
Animal a = d;     // SLICING: the Dog part is cut off, a is just an Animal
// a.speak();     // ERROR: speak() is pure virtual — crashes at runtime

// Correct polymorphism always through pointer or reference:
Animal& ref = d;   // no slicing
Animal* ptr = &d;  // no slicing
ref.speak();       // "Woof" — correct
```

### Trap 2: Iterator Invalidation

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};

// Erasing while iterating — classic bug:
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it == 3) v.erase(it);  // BUG: it is invalidated by erase!
}

// Correct pattern:
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 3) it = v.erase(it);  // erase() returns next valid iterator
    else          ++it;
}

// Or use erase-remove idiom (C++20: std::erase):
std::erase(v, 3);                              // remove all 3s
std::erase_if(v, [](int x){ return x % 2; }); // remove all odds

// push_back may invalidate ALL iterators (reallocation):
auto it = v.begin();
v.push_back(99);  // may reallocate — it is now DANGLING
// *it = 0;       // UB!
```

### Trap 3: Dangling Reference

```cpp
const std::string& get_name() {
    std::string name = "sensor_01";
    return name;  // DANGLING: name destroyed on return
}

// Also common:
std::string_view get_view() {
    std::string s = "hello";
    return s;   // DANGLING: string_view points to destroyed string
}

// And with temporaries:
const std::string& r = std::string("hello");  // C++: lifetime extended — OK
std::string_view sv = std::string("hello");   // NOT extended for string_view — DANGLING
```

### Trap 4: The Most Vexing Parse

```cpp
// This is NOT creating a Widget:
Widget w();     // Declares a function named w returning Widget!

// Fix: use brace init or drop parentheses:
Widget w{};    // default-constructed Widget
Widget w2;     // also works (default construction)
```

### Trap 5: Accidental Copy vs Move

```cpp
std::vector<std::string> names;
std::string s = "hello";

names.push_back(s);            // COPY: s still usable after (costs allocation)
names.push_back(std::move(s)); // MOVE: s is now empty (no allocation)
// After move: s == "" (valid but empty)

// In return: don't std::move (prevents NRVO which is better than move):
std::string make_name() {
    std::string name = "sensor";
    return name;              // GOOD: NRVO may eliminate ALL copies
    // return std::move(name); // BAD: disables NRVO!
}
```

### Trap 6: Integer Type Mismatch

```cpp
size_t size = get_size();  // unsigned: 0 to SIZE_MAX
int    n    = -1;

if (n < size)              // WARNING: n promoted to size_t → becomes SIZE_MAX!
    process(n);            // never reached when n = -1

// Fix: cast explicitly or use signed comparison
if (n >= 0 && static_cast<size_t>(n) < size)
    process(n);
```

### Trap 7: `std::shared_ptr` Cycles

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;  // BUG: cycle → neither ref count reaches 0 → leak!
};

// Fix: back-pointers use weak_ptr
struct Node_Fixed {
    std::shared_ptr<Node_Fixed> next;
    std::weak_ptr<Node_Fixed>   prev;  // doesn't extend lifetime
};
```

### Trap 8: Thread Safety of `shared_ptr` Itself

```cpp
// shared_ptr reference counting IS thread-safe
// But the pointed-to object is NOT thread-safe through shared_ptr

std::shared_ptr<int> sp = std::make_shared<int>(0);

// SAFE: multiple threads copying/destroying shared_ptr (ref count is atomic)
auto sp2 = sp;  // thread-safe

// UNSAFE: multiple threads accessing the int through sp
*sp += 1;  // data race if called from multiple threads!

// Fix: protect the object itself, not the shared_ptr
std::shared_ptr<std::atomic<int>> safe_sp = std::make_shared<std::atomic<int>>(0);
(*safe_sp)++;  // safe
```

---

## 14. Code Review Checklist

Use this when reviewing your own code or someone else's. Each item should be consciously checked.

### Ownership & Memory
- [ ] Every `new` has a corresponding `delete`, or better: using `unique_ptr`/`shared_ptr`
- [ ] No `new[]` without `delete[]` — prefer `make_unique<T[]>`
- [ ] No raw owning pointers in class members — use smart pointers
- [ ] Move constructors and move assignments are `noexcept`
- [ ] Destructors don't throw
- [ ] Rule of Zero / Three / Five respected

### Type Safety
- [ ] No C-style casts — use `static_cast`, `dynamic_cast`, `const_cast`, `reinterpret_cast`
- [ ] No implicit narrowing conversions — use brace init `{}`
- [ ] Signed/unsigned comparison warnings addressed
- [ ] `enum class` used instead of plain `enum`
- [ ] `nullptr` used instead of `NULL` or `0`

### Const Correctness
- [ ] Member functions that don't modify state are `const`
- [ ] Parameters passed by reference are `const&` when not modified
- [ ] Class members that are logically immutable are `const`
- [ ] `mutable` only used for caching/lazy init, never to bypass logical const

### Concurrency
- [ ] All shared mutable state protected by mutex or is `std::atomic`
- [ ] `volatile` NOT used for thread synchronization
- [ ] No deadlocks: locks acquired in consistent order
- [ ] Condition variable `wait()` uses a predicate (spurious wakeup protection)
- [ ] No data races (run with `-fsanitize=thread`)
- [ ] ISR code does NOT use mutexes, heap allocation, or blocking

### Error Handling
- [ ] All error returns checked (use `[[nodiscard]]`)
- [ ] No swallowed exceptions (`catch (...)` without logging)
- [ ] Failed constructor throws or sets error state (no silent partial init)
- [ ] RAII used for all resources (no naked open/close pairs)

### Performance
- [ ] No unnecessary copies — use `const&`, `std::move`, `emplace_back`
- [ ] No heap allocation in real-time / ISR paths
- [ ] `reserve()` called before filling a `vector` of known size
- [ ] `string_view` / `span` used instead of copying strings/arrays for read-only access
- [ ] Hot paths avoid virtual dispatch where CRTP or templates would do

### General Quality
- [ ] No UB: no signed overflow, no null deref, no OOB, no use-after-free
- [ ] `static_assert` for compile-time constraints (size, alignment, enum values)
- [ ] Public API functions are `[[nodiscard]]` where appropriate
- [ ] Comments explain *why*, not *what* (code explains what)
- [ ] No magic numbers — use named constants or `constexpr`

---

## 15. Final Interview Scenarios & Q&A

### Scenario: Design a Thread-Safe Event Queue

**Question:** Design a thread-safe queue for passing events between threads in an embedded Linux system. No dynamic allocation allowed.

```cpp
template<typename T, size_t N>
class BlockingQueue {
    std::array<T, N>        buf_;
    size_t                  head_ = 0, tail_ = 0, count_ = 0;
    mutable std::mutex      mtx_;
    std::condition_variable not_empty_;
    std::condition_variable not_full_;

public:
    // Producer (blocks if full):
    void push(T val) {
        std::unique_lock lock(mtx_);
        not_full_.wait(lock, [this]{ return count_ < N; });
        buf_[tail_] = std::move(val);
        tail_ = (tail_ + 1) % N;
        ++count_;
        not_empty_.notify_one();
    }

    // Consumer (blocks until item available):
    T pop() {
        std::unique_lock lock(mtx_);
        not_empty_.wait(lock, [this]{ return count_ > 0; });
        T val = std::move(buf_[head_]);
        head_ = (head_ + 1) % N;
        --count_;
        not_full_.notify_one();
        return val;
    }

    // Non-blocking try versions:
    bool try_push(T val) {
        std::lock_guard lock(mtx_);
        if (count_ == N) return false;
        buf_[tail_] = std::move(val);
        tail_ = (tail_ + 1) % N;
        ++count_;
        not_empty_.notify_one();
        return true;
    }

    bool try_pop(T& val) {
        std::lock_guard lock(mtx_);
        if (count_ == 0) return false;
        val = std::move(buf_[head_]);
        head_ = (head_ + 1) % N;
        --count_;
        not_full_.notify_one();
        return true;
    }

    size_t size() const {
        std::lock_guard lock(mtx_);
        return count_;
    }
};

// Usage: IOC message passing
BlockingQueue<IocMessage, 32> ioc_queue;

void ioc_receive_thread() {
    while (running_) {
        auto msg = ioc_queue.pop();  // blocks until message available
        dispatch(msg);
    }
}
```

---

### Deep Q&A: The Questions Interviewers Actually Ask

**Q: What is the difference between `std::unique_ptr` and a raw owning pointer?**

A: `unique_ptr` enforces single ownership through the type system — it cannot be copied (only moved), and it automatically calls `delete` (or a custom deleter) in its destructor. A raw owning pointer has none of these guarantees: it can be accidentally copied, the ownership isn't expressed in the type, and forgetting `delete` is a leak. `unique_ptr` is zero-overhead over a raw pointer — it compiles to identical machine code in release mode.

---

**Q: Explain the difference between deep copy and shallow copy. When does the default copy constructor do a shallow copy?**

A: A shallow copy copies the pointer value — both objects end up pointing to the same underlying data. A deep copy allocates new storage and copies the data. The compiler-generated copy constructor does a shallow copy: it copies each member, including pointers, without following them. This is wrong for any class that owns heap memory — both copies will try to `delete` the same pointer on destruction, causing a double-free (UB). This is exactly why the Rule of Three exists: if you have a destructor that `delete`s memory, you must also write a copy constructor that deep-copies.

---

**Q: What happens if you throw an exception from a destructor?**

A: If a destructor throws while another exception is being propagated (during stack unwinding), `std::terminate()` is called immediately — no recovery. Even outside of exception propagation, throwing from a destructor is bad practice because it makes it impossible to guarantee cleanup. Destructors are implicitly `noexcept` since C++11 — if a destructor throws, `std::terminate` is called. Always catch and log errors in destructors, never re-throw.

---

**Q: What is the difference between `std::mutex` and `std::atomic`? When would you use each?**

A: `std::atomic<T>` provides lock-free atomic operations on a single value — it's used when you need to read or modify a single variable from multiple threads without overhead. `std::mutex` protects a critical section — a block of code — that may access multiple variables or resources. Use atomic for: counters, flags, single-value publish-subscribe. Use mutex for: protecting a struct, a container, or any multi-variable invariant. Importantly: `volatile` is neither — it prevents compiler caching but provides no thread-safety guarantees at all.

---

**Q: What is RAII and why is it the most important C++ idiom?**

A: RAII = Resource Acquisition Is Initialization. The rule: tie every resource (memory, file handle, mutex lock, socket, DMA channel) to an object whose constructor acquires it and whose destructor releases it. Because C++ guarantees destructor calls when objects go out of scope — even during exception propagation — RAII guarantees no resource leaks regardless of how a function exits (normal return, early return, or exception). It's the most important idiom because it makes resource management automatic, correct, and composable without any additional effort at call sites.

---

**Q: In your Computer simulator assignment, `exec_op` wrote directly to `std::cout`. How would you redesign this?**

A: The problem is coupling the execution logic to a specific output mechanism. I'd refactor using dependency injection: accept an `std::function<void(int)>` or an abstract `IOutput` interface in the constructor. The `exec_op` calls this output handler instead of `std::cout` directly. For the test case runner, pass a lambda that appends to a `std::vector<int>`. For the interactive runner, pass a lambda that prints to `std::cout`. This makes `exec_op` testable in isolation — tests can verify the exact sequence of output values without capturing stdout.

---

**Q: What is `std::move` and what does it actually do to the object it's called on?**

A: `std::move` does NOT move anything. It's a cast — specifically, it casts its argument to an rvalue reference (`T&&`), which signals to the compiler "this object is safe to move from." The actual movement happens when a move constructor or move assignment operator uses that rvalue reference. After a move, the moved-from object is in a valid but unspecified state — it can be safely destroyed or reassigned, but its contents are unspecified (typically empty or zero).

---

**Q: What is undefined behavior? Give three examples critical in embedded systems.**

A: Undefined behavior is behavior the C++ standard doesn't define — the program may do anything, and the compiler is allowed to assume it never happens (enabling optimizations that produce silently wrong code). Three critical embedded examples:
1. **Signed integer overflow** — the compiler assumes `i+1 > i` is always true for signed `i`, which can eliminate overflow checks in loops, causing infinite loops or incorrect bounds.
2. **Accessing memory through the wrong type (strict aliasing)** — `reinterpret_cast<int*>(&float_var)` — the compiler assumes `int*` and `float*` never alias, so it may not see the write through `int*` when reading through `float*`, causing stale data reads. Use `memcpy` for type punning.
3. **Data race on non-atomic shared variable** — reading and writing a plain variable from two threads without synchronization is UB — the compiler may cache the value in a register, transforming a flag check into an infinite loop.

---

**Q: What is the difference between `const int*`, `int* const`, and `const int* const`?**

A: Read right to left after the variable name:
- `const int* p`: pointer to const int — `p` can be redirected, but `*p` cannot be written through `p`
- `int* const p`: const pointer to int — `p` cannot be redirected (must be initialized), but `*p` can be modified
- `const int* const p`: const pointer to const int — neither can change

The pattern matters in embedded: `volatile uint32_t* const REG = 0x40000000` means the register address is fixed (const pointer) but the register value changes (volatile, non-const pointee).

---

**Q: When would you choose `std::variant` over a class hierarchy with virtual functions?**

A: `std::variant` is better when:
- The set of types is closed and known at compile time (exactly these N types, no more)
- You want value semantics (stack-allocated, copyable, no heap)
- You want exhaustive handling enforced by the compiler (forgetting a case in `std::visit` is a compile error)
- Performance is critical (no vtable, no heap, no indirection)

A class hierarchy with virtuals is better when:
- The set of types is open — third parties can add new derived classes
- You need to store heterogeneous types in a container (`std::vector<std::unique_ptr<Base>>`)
- The types need to be selected and passed around by base pointer at runtime

In an embedded state machine: use `std::variant`. In a plugin system where new devices can be added: use virtual.

---

**Q: What does `[[nodiscard]]` do and why should you use it on error-returning functions?**

A: `[[nodiscard]]` on a function or type causes a compiler warning if the caller discards the return value. This is critical for error-returning functions because the most common bug is calling a function that returns an error code and ignoring it. `ErrCode init()` — without `[[nodiscard]]`, `init();` silently ignores whether init succeeded. With `[[nodiscard]]`, the compiler warns: "return value of 'init' annotated with nodiscard is discarded." The caller must do *something* with the result, even if just assign to `[[maybe_unused]] auto _ = init()` — which makes the discarding explicit and intentional.

---

*End of Document 5 — You now have the complete C++ mastery series*

---

## Complete Series Summary

| Document | Focus |
|---|---|
| **Doc 1** | Core fundamentals: types, pointers, references, classes, RAII, casting |
| **Doc 2** | Move semantics, templates, SFINAE, Concepts, CRTP, policy-based design |
| **Doc 3** | Standard library, smart pointers, containers, concurrency, atomics |
| **Doc 4** | Modern C++11/14/17/20 features by version |
| **Doc 5** | Design patterns, error handling, embedded best practices, interview Q&A |

**The single most important thing to demonstrate in a C++ interview:**
You don't just know the syntax — you understand *why* each feature exists, *what problem it solves*, and *when not to use it*. That judgment, not memorization, is what distinguishes a senior C++ engineer.
