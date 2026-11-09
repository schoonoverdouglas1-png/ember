# Gemma Lesson — Advanced C++, Lesson 1
*Taught Nov 9, 2026, through the free Gemini lane (gemini-3.5-flash-lite), at Douglas's direction to turn the free learning on through Gemma. Method per his college rules: the teaching AI takes its own test first; Ember fact-checks before keeping. Ember's first-read spot check: the three answers and the lesson are standard, correct doctrine — no errors found at first read; the full triple-check runs before this counts as learned. Usage: 850 tokens total, Gemma's side, free — zero from Douglas's allotment.*

---

### (1) TEST YOURSELF

**Q1 (RAII & Exception Safety):** Why is a raw pointer member in a class violating the Rule of Zero/Five considered an anti-pattern in modern C++, and how does the strict implementation of RAII prevent resource leaks during stack unwinding?
*Answer:* A raw pointer does not manage lifetime. If an exception is thrown in a constructor *after* a raw resource is acquired (via `new`), the destructor of that class will *not* be called, leaking the resource. RAII (Resource Acquisition Is Initialization) binds resource lifecycle to object scope. By wrapping resources in smart pointers (`std::unique_ptr`) or containers (`std::vector`), stack unwinding guarantees that local objects go out of scope, invoking their destructors and deterministically releasing resources.

**Q2 (Move Semantics & Perfect Forwarding):** What is the exact problem that universal references (`T&&` in a deduced context) and `std::forward` solve, and why would using `std::move` inside a template function accepting a universal reference break perfect forwarding?
*Answer:* Universal references (forwarding references) preserve the lvalueness or rvalueness of argument types passed into a template. `std::forward<T>` conditionally casts the argument back to its original category. If you used `std::move` (which is an unconditional cast to an rvalue reference), you would forcefully convert lvalues into rvalues, stripping away their constness and category, destroying caller intent, and preventing proper overload resolution in downstream functions.

**Q3 (Templates & Metaprogramming):** Explain how Substitution Failure Is Not An Error (SFINAE) works, and contrast it briefly with C++20 Concepts regarding template instantiation errors.
*Answer:* SFINAE dictates that if an invalid substitution of template parameters occurs during the immediate signature evaluation of a function template, the compiler drops that overload from the overload set instead of halting with a hard compilation error. C++20 Concepts replace this hacky technique with explicit constraints (`requires` clauses), offering readable compiler diagnostics, direct boolean evaluation of constraints, and cleaner overload resolution semantics.

---

### LESSON 1: Deterministic Resource Management (RAII)

Welcome, Ember. The single most important idea in advanced C++ for systems work is **Deterministic Resource Management via RAII**.

In systems programming, garbage collection introduces unpredictable latency. Manual `new` and `delete` lead to leaks and use-after-free bugs. RAII eliminates this entirely by tying every system resource (memory, file descriptors, locks, sockets) to the lifetime of a stack-allocated object.

When the object's scope ends, its destructor executes *deterministically*, regardless of whether execution leaves the scope normally or via an exception.

```cpp
#include <iostream>
#include <memory>

class SystemResource {
public:
    SystemResource() { std::cout << "Acquired hardware lock.\n"; }
    ~SystemResource() { std::cout << "Released hardware lock.\n"; }
    void process() { std::cout << "Processing data...\n"; }
};

void executeTask() {
    // RAII wrapper guarantees cleanup, even if process() throws an exception
    auto resource = std::make_unique<SystemResource>();
    resource->process();
} // <- resource goes out of scope here; destructor fires automatically.

int main() {
    executeTask();
}
```

**Rule for systems work:** Never write `new` or `delete`. Let scope dictate lifetime.
