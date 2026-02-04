---
title: "Singleton class in SystemVerilog: why it matters, where to use it, and a practical example"
date: 2026-02-04T10:56:00-06:00
draft: false
tags: ["systemverilog", "uvm", "verification", "design-patterns"]
---

Singleton is a classic object-oriented pattern: **a class that guarantees exactly one instance exists** (per simulation/process), and provides a global access point to that instance.

In SystemVerilog (SV) testbenches—especially UVM—Singletons show up naturally for things like **configuration/state holders, registries, and shared services**.

This post covers:

- What a Singleton is (in SV terms)
- Why you might want it in a verification environment
- Common applications (UVM examples)
- A practical, copy-paste SV implementation + usage
- Pitfalls and better alternatives when needed

## 1) What is a Singleton class?

A Singleton class:

1. **Prevents external code from calling `new()`** freely (usually by making the constructor `local`/`protected`).
2. Provides a `static function get()` (or similar) that:
   - creates the instance the first time
   - returns the same instance on every subsequent call

Conceptually:

- *No matter how many times you ask, you get the same object.*

## 2) Why it matters in SystemVerilog verification

SV/UVM testbenches are built from many objects created at different times:

- sequences
- drivers/monitors/agents
- scoreboards
- tests
- utility objects

Sometimes you want **a single shared service** that everyone can access without having to thread handles through 10 layers of constructors.

Singletons can help when you need:

- **One consistent view of state** (counters, run metadata, feature toggles)
- **Centralized logging/telemetry** beyond what `uvm_report_*` gives you
- **A registry** (e.g., mapping IDs → human-readable names)
- **A shared resource manager** (e.g., allocating unique stream IDs)

That said, you should use them intentionally—because they are global state.

## 3) Typical applications (especially in UVM)

Here are real places you’ll see (or can use) Singleton-like patterns:

### A) Run metadata / build info
A single object that holds:

- git commit/version string
- seed
- command-line knobs parsed from `+args`

### B) Central registry
- register protocol IDs and decode names
- maintain a database of coverage bin names

### C) Shared configuration defaults
UVM already has `uvm_config_db`, but a Singleton can hold **defaults**, computed values, or pre-parsed knobs.

### D) Resource allocator
If many sequences need a unique ID (stream ID, transaction tag, etc.), a Singleton allocator avoids collisions.

## 4) A practical Singleton in SV (copy/paste)

Below is a small but practical example: a **global transaction ID allocator** used by any component/sequence.

### The Singleton class

```systemverilog
// file: txn_id_allocator.sv

class txn_id_allocator;
  // the single instance
  static txn_id_allocator m_inst;

  // internal state
  int unsigned m_next_id;

  // prevent arbitrary construction
  local function new();
    m_next_id = 1;
  endfunction

  // global access point
  static function txn_id_allocator get();
    if (m_inst == null) begin
      m_inst = new();
    end
    return m_inst;
  endfunction

  // service API
  function int unsigned alloc();
    return m_next_id++;
  endfunction

  function void reset(int unsigned start_id = 1);
    m_next_id = start_id;
  endfunction
endclass
```

### Example usage (sequence or component)

```systemverilog
class my_seq extends uvm_sequence #(my_item);
  `uvm_object_utils(my_seq)

  function new(string name = "my_seq");
    super.new(name);
  endfunction

  virtual task body();
    my_item req;

    req = my_item::type_id::create("req");
    req.txn_id = txn_id_allocator::get().alloc();

    `uvm_info(get_type_name(), $sformatf("Allocated txn_id=%0d", req.txn_id), UVM_MEDIUM)

    start_item(req);
    finish_item(req);
  endtask
endclass
```

With this, every caller uses **the same allocator instance**, so IDs are unique across the whole testbench.

## 5) Importance: why a Singleton can be the right tool

A well-chosen Singleton:

- **reduces plumbing** (no need to pass handles everywhere)
- **standardizes behavior** (everyone uses the same service)
- **avoids accidental duplication** (two allocators causing collisions)

In verification, the “one truth” aspect can prevent subtle bugs.

## 6) Pitfalls (and how to avoid them)

Singletons are global state. Common issues:

### A) Hidden dependencies
If any code can call `get()`, it becomes harder to understand what depends on what.

**Mitigation:** keep the API small and name the class clearly (`*_registry`, `*_allocator`, etc.).

### B) Reset/reproducibility
A global object can carry state across tests *within the same simulation* if you run multiple tests in one run (or if you do unusual flows).

**Mitigation:** provide an explicit `reset()` and call it in a known phase (e.g., test `build_phase` or `start_of_simulation_phase`).

### C) Parallelism / re-entrancy
If multiple threads allocate at the same time, you can get ordering surprises.

In most SV simulators, a simple increment is safe in terms of atomicity at the language level, but interleaving can still produce unexpected sequences of IDs.

**Mitigation:** if ordering matters, consider a mailbox/queue-based allocator or a single-threaded service.

### D) Overuse
If everything becomes a Singleton, your testbench becomes hard to reason about.

**Rule of thumb:** use Singleton for **services/registries**—not for “business logic” or components.

## 7) When to use UVM mechanisms instead

Before writing a Singleton, consider whether one of these is a better fit:

- **`uvm_config_db`** for configuration distribution
- **`uvm_resource_db`** for globally named resources
- **Factory overrides** (`type_id::set_type_override`) to swap implementations
- **Central component** (e.g., env-level object) passed by handles if you want explicit dependencies

A Singleton is fine when you truly want a global service with minimal coupling—but don’t reach for it by default.

## 8) Summary

- A Singleton in SV ensures **one instance + global access**.
- It’s useful for **allocators, registries, run metadata, shared utilities**.
- Be careful with **hidden dependencies** and **state reset**.
- Prefer UVM-native mechanisms for config/resources when appropriate.

If you tell me your exact use-case (e.g., “global config object”, “logger”, “scoreboard registry”, “unique transaction ID across agents”), I can tailor the example to your environment and show the cleanest UVM-phase integration.
