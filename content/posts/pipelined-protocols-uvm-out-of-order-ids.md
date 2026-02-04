---
title: "Pipelined Protocols in UVM (Out-of-Order Responses) — A Practical Pattern"
date: 2026-02-03T21:25:59-0600
draft: false
summary: "A practical, end-to-end pattern for verifying ID-tagged pipelined protocols in UVM: monitors, in-flight tracking, a reorder buffer scoreboard, and sequences that stress concurrency." 
tags: ["UVM", "SystemVerilog", "Verification", "Pipelining", "Scoreboard", "Out-of-Order", "AXI", "Transaction IDs", "Monitors", "Testbench Architecture"]
---

## Why pipelining (with IDs) changes your verification problems

A non-pipelined protocol is comfortable: you send a request, you wait, you get a response, repeat. Your scoreboard can be a simple FIFO.

A **pipelined protocol** lets you have **multiple in-flight requests**. An **ID-tagged, out-of-order** protocol goes further: responses can return **in a different order** than requests.

That breaks the “one FIFO matches everything” assumption.

### The core verification question

> For every request you accepted, did you eventually observe the correct response **with the same ID**, and did the DUT obey the protocol rules for ordering, backpressure, and ID reuse?

You need a testbench that is:

- **Concurrent** (many outstanding requests)
- **Order-aware** (or rather, order-agnostic but **ID-aware**)
- **Debuggable** when something returns late, never returns, or returns with the wrong payload

This post lays out a pattern that works well for AXI-like systems and any “ID + out-of-order completion” design.

---

## Mental model: in-flight table + completion matching

Think of your traffic as two streams:

- `REQ`: accepted requests, each with an **ID**
- `RSP`: observed responses, each with an **ID**

The protocol allows:

- many in-flight requests (different IDs or even same ID depending on spec)
- responses can return in any order (but usually **per-ID ordering** rules apply)

### Diagram: request/response reordering

```
Time →

REQ:  ID=3 ─────┐
REQ:  ID=7 ───┐ │
REQ:  ID=3 ─┐ │ │   (multiple in-flight; IDs may repeat depending on rules)
            │ │ │
RSP:     ID=7 ◀┘ │   (response for ID=7 returns first)
RSP:        ID=3 ◀──┘ (later, response(s) for ID=3)
```

Your scoreboard should behave like:

- When you see a **request accept**: store “expected completion” keyed by ID
- When you see a **response**: look up the matching expected entry (by ID), compare, then retire

This is a **reorder buffer** (ROB) conceptually.

---

## A practical UVM architecture that scales

Here’s a clean decomposition:

- **Driver / Sequencer**: generates requests with IDs and randomization
- **Request monitor**: observes `req_valid && req_ready` and publishes `req_txn`
- **Response monitor**: observes `rsp_valid && rsp_ready` and publishes `rsp_txn`
- **Scoreboard**:
  - collects reqs and rsps
  - maintains in-flight tracking
  - matches responses by ID
  - enforces protocol rules (timeouts, illegal ID reuse, max outstanding, etc.)

### Diagram: agent + scoreboard dataflow

```
                    +-------------------+
Sequence  ------->  | uvm_sequencer     |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    | uvm_driver        |  (drives req channel)
                    +---------+---------+
                              |
                              v
                         [  DUT  ]
                              ^
                              |
                    +---------+---------+
                    | rsp_monitor       |----> rsp_ap ----+
                    +-------------------+                  |

                    +-------------------+                  v
                    | req_monitor       |----> req_ap --> +------------------+
                    +-------------------+                 | scoreboard        |
                                                          |  in-flight table  |
                                                          |  matching by ID   |
                                                          +------------------+
```

Key point: **the scoreboard should not depend on the driver**. It should observe the *actual* handshakes.

---

## Worked example protocol (AXI-like but simplified)

To keep code readable, we’ll assume two independent channels:

- Request channel: `req_valid`, `req_ready`, `req_id`, `req_addr`, `req_data`, `req_is_write`
- Response channel: `rsp_valid`, `rsp_ready`, `rsp_id`, `rsp_status`, `rsp_data`

Rules (reasonable defaults):

1) A request is “accepted” when `req_valid && req_ready`
2) A response is “accepted” when `rsp_valid && rsp_ready`
3) Responses can return out-of-order across IDs
4) **Within the same ID**, responses are returned **in order** (common AXI-ish rule)
5) **ID reuse is allowed**, but you cannot have more than `MAX_PER_ID` outstanding per ID

If your DUT’s rules differ (e.g., no per-ID ordering), the pattern still works—just tweak the checks.

---

## Transaction types (req/rsp)

Keep request and response items distinct; you can also wrap both in a combined “expected” record.

```systemverilog
class proto_req extends uvm_sequence_item;
  rand bit [3:0]  id;
  rand bit        is_write;
  rand bit [31:0] addr;
  rand bit [31:0] data;

  `uvm_object_utils_begin(proto_req)
    `uvm_field_int(id,       UVM_ALL_ON)
    `uvm_field_int(is_write, UVM_ALL_ON)
    `uvm_field_int(addr,     UVM_ALL_ON)
    `uvm_field_int(data,     UVM_ALL_ON)
  `uvm_object_utils_end

  function new(string name="proto_req");
    super.new(name);
  endfunction
endclass

class proto_rsp extends uvm_sequence_item;
  bit [3:0]  id;
  bit [1:0]  status;
  bit [31:0] data;

  `uvm_object_utils_begin(proto_rsp)
    `uvm_field_int(id,     UVM_ALL_ON)
    `uvm_field_int(status, UVM_ALL_ON)
    `uvm_field_int(data,   UVM_ALL_ON)
  `uvm_object_utils_end

  function new(string name="proto_rsp");
    super.new(name);
  endfunction
endclass
```

---

## Monitors: sample only on real handshakes

### Request monitor

```systemverilog
class proto_req_monitor extends uvm_component;
  `uvm_component_utils(proto_req_monitor)

  virtual proto_if vif;
  uvm_analysis_port #(proto_req) ap;

  function new(string name, uvm_component parent);
    super.new(name, parent);
    ap = new("ap", this);
  endfunction

  function void build_phase(uvm_phase phase);
    if (!uvm_config_db#(virtual proto_if)::get(this, "", "vif", vif))
      `uvm_fatal("NOVIF", "vif not set")
  endfunction

  task run_phase(uvm_phase phase);
    proto_req tr;
    forever begin
      @(posedge vif.clk);
      if (!vif.rst_n) continue;

      if (vif.req_valid && vif.req_ready) begin
        tr = proto_req::type_id::create("tr");
        tr.id       = vif.req_id;
        tr.is_write = vif.req_is_write;
        tr.addr     = vif.req_addr;
        tr.data     = vif.req_data;
        ap.write(tr);
      end
    end
  endtask
endclass
```

### Response monitor

```systemverilog
class proto_rsp_monitor extends uvm_component;
  `uvm_component_utils(proto_rsp_monitor)

  virtual proto_if vif;
  uvm_analysis_port #(proto_rsp) ap;

  function new(string name, uvm_component parent);
    super.new(name, parent);
    ap = new("ap", this);
  endfunction

  function void build_phase(uvm_phase phase);
    if (!uvm_config_db#(virtual proto_if)::get(this, "", "vif", vif))
      `uvm_fatal("NOVIF", "vif not set")
  endfunction

  task run_phase(uvm_phase phase);
    proto_rsp tr;
    forever begin
      @(posedge vif.clk);
      if (!vif.rst_n) continue;

      if (vif.rsp_valid && vif.rsp_ready) begin
        tr = proto_rsp::type_id::create("tr");
        tr.id     = vif.rsp_id;
        tr.status = vif.rsp_status;
        tr.data   = vif.rsp_data;
        ap.write(tr);
      end
    end
  endtask
endclass
```

**Tip:** If the protocol has multi-beat bursts, monitors should assemble a complete transaction before publishing (often with a small state machine). The scoreboard pattern stays the same.

---

## Scoreboard: in-flight tracking + reorder buffer matching

Here’s a robust approach:

- Keep per-ID queues of expected completions
- On request accept: push expected entry into `expected_by_id[id]`
- On response: pop front from `expected_by_id[id]`, compare
- Also keep global counters + timeout tracking

### Expected record

```systemverilog
typedef struct {
  proto_req req;
  int unsigned seq_no;   // monotonically increasing for debug
  time accepted_time;
} expected_t;
```

### Scoreboard skeleton

```systemverilog
class proto_scoreboard extends uvm_component;
  `uvm_component_utils(proto_scoreboard)

  // Analysis exports (connect monitors into these)
  uvm_analysis_imp #(proto_req, proto_scoreboard) req_imp;
  uvm_analysis_imp #(proto_rsp, proto_scoreboard) rsp_imp;

  localparam int ID_W = 4;
  localparam int NUM_IDS = 1 << ID_W;
  localparam int MAX_PER_ID = 8;
  localparam time TIMEOUT = 10_000ns;

  // Per-ID FIFO of expected completions
  expected_t expected_by_id[NUM_IDS][$];

  int unsigned next_seq_no = 0;
  int unsigned total_inflight = 0;

  function new(string name, uvm_component parent);
    super.new(name, parent);
    req_imp = new("req_imp", this);
    rsp_imp = new("rsp_imp", this);
  endfunction

  // Called when req monitor writes
  function void write(proto_req t);
    expected_t e;
    int id = t.id;

    if (expected_by_id[id].size() >= MAX_PER_ID) begin
      `uvm_error("SB", $sformatf("Too many outstanding for ID=%0d (size=%0d)", id, expected_by_id[id].size()))
      return;
    end

    e.req = t;
    e.seq_no = next_seq_no++;
    e.accepted_time = $time;

    expected_by_id[id].push_back(e);
    total_inflight++;

    `uvm_info("SB", $sformatf("REQ accept: id=%0d seq=%0d inflight=%0d", id, e.seq_no, total_inflight), UVM_MEDIUM)
  endfunction

  // Called when rsp monitor writes
  function void write(proto_rsp r);
    expected_t e;
    int id = r.id;

    if (expected_by_id[id].size() == 0) begin
      `uvm_error("SB", $sformatf("RSP with no matching REQ: id=%0d status=%0d", id, r.status))
      return;
    end

    e = expected_by_id[id].pop_front();
    total_inflight--;

    // Compare: customize to your protocol
    // Example: for reads, compare data; for writes, maybe only status.
    if (!e.req.is_write) begin
      // placeholder expected model: just an example
      // In real tests, compute expected data from a reference model/memory.
      `uvm_info("SB", $sformatf("READ complete: id=%0d seq=%0d data=0x%08x", id, e.seq_no, r.data), UVM_LOW)
    end

    if (r.status != 0) begin
      `uvm_warning("SB", $sformatf("Non-OK response: id=%0d seq=%0d status=%0d", id, e.seq_no, r.status))
    end

    `uvm_info("SB", $sformatf("RSP match: id=%0d seq=%0d inflight=%0d", id, e.seq_no, total_inflight), UVM_MEDIUM)
  endfunction

  task run_phase(uvm_phase phase);
    // Timeout watchdog
    forever begin
      #(100ns);
      check_timeouts();
    end
  endtask

  function void check_timeouts();
    for (int id = 0; id < NUM_IDS; id++) begin
      if (expected_by_id[id].size() > 0) begin
        expected_t oldest = expected_by_id[id][0];
        if (($time - oldest.accepted_time) > TIMEOUT) begin
          `uvm_error("SB", $sformatf("TIMEOUT waiting for RSP: id=%0d oldest_seq=%0d age=%0t", id, oldest.seq_no, ($time-oldest.accepted_time)))
        end
      end
    end
  endfunction
endclass
```

### What this scoreboard enforces

- **No spurious responses** (response without matching request)
- **Per-ID ordering** (by using per-ID FIFO: responses for an ID must come in order)
- **Max in-flight per ID** (helps catch illegal ID reuse / queue overflow)
- **Timeouts** for missing responses

If your protocol allows out-of-order even within the same ID, replace the per-ID FIFO with a per-ID associative structure keyed by “tag” or sequence number.

---

## Sequences that actually stress pipelining (not just random)

A good pipelining stress sequence should:

- keep **many** requests outstanding
- randomize IDs (with controlled distribution)
- introduce backpressure on response channel (random `rsp_ready` deassertions)
- sometimes reuse IDs aggressively (to test per-ID depth)

Pseudo strategy:

1) Generate `N` requests quickly (driver uses `req_ready` handshake)
2) Continue generating while responses are still coming (don’t “wait for empty” too early)

### Example sequence item generation (sketch)

```systemverilog
class pipelined_ooo_seq extends uvm_sequence #(proto_req);
  `uvm_object_utils(pipelined_ooo_seq)

  rand int unsigned num_reqs = 200;
  rand int unsigned max_id = 15;

  function new(string name="pipelined_ooo_seq");
    super.new(name);
  endfunction

  task body();
    proto_req tr;

    repeat (num_reqs) begin
      tr = proto_req::type_id::create("tr");
      assert(tr.randomize() with {
        id inside {[0:max_id]};
        // Mix reads/writes; skew toward reads if you want data checking
        is_write dist {0:=7, 1:=3};
        addr[1:0] == 0; // word aligned
      });

      start_item(tr);
      finish_item(tr);

      // Small random gaps; keep pipeline full
      if ($urandom_range(0,3) == 0)
        #(10ns);
    end
  endtask
endclass
```

**Note:** Your driver must be written to allow continuous streaming; don’t serialize on response completion.

---

## Debugging tips (this is where time gets saved)

### 1) Log with sequence numbers
Even if your protocol’s ID repeats, having a monotonically increasing `seq_no` in the scoreboard makes it easy to say:

- “REQ seq 147 (ID=3) never completed”

### 2) Keep a “dump in-flight” method
When an error occurs, print the outstanding table:

```
ID=0: [seq=12, seq=44]
ID=3: [seq=101]
ID=7: [seq=77, seq=78, seq=79]
```

This immediately tells you whether you’re stuck behind a missing response for a particular ID.

### 3) Add response-side assertions
In addition to scoreboard checks, consider protocol assertions:

- Response ID must be within range
- No response when reset
- If response is multi-beat, `last` behavior is correct

---

## Common pitfalls (and how to avoid them)

1) **Sampling requests when `valid` is high but not accepted**
   - Always use `valid && ready` for accepted transactions.

2) **Assuming a global order**
   - Out-of-order means you must match by ID (and perhaps tag).

3) **Not modeling per-ID ordering rules**
   - Many protocols guarantee in-order per ID. Enforce it with a per-ID FIFO.

4) **ID reuse corner cases**
   - If DUT reuses an ID too early, the scoreboard will see “too many outstanding per ID” or mismatches.

5) **Timeouts that are too strict**
   - Choose TIMEOUT based on max pipeline depth + worst-case backpressure.

---

## Checklist: ship a solid pipelined/OOO UVM environment

- [ ] Request monitor publishes only on `req_valid && req_ready`
- [ ] Response monitor publishes only on `rsp_valid && rsp_ready`
- [ ] Scoreboard matches responses by ID (and enforces per-ID ordering if applicable)
- [ ] Max outstanding per ID check is implemented
- [ ] Timeout watchdog catches missing responses
- [ ] Sequence keeps multiple in-flight requests (doesn’t serialize)
- [ ] Backpressure is randomized on both channels (as applicable)

---

## Suggested filename slug

`pipelined-protocols-uvm-out-of-order-ids.md`

## Alternate titles

1) “Verifying Out-of-Order, ID-Tagged Pipelines in UVM: A Scoreboard Pattern”
2) “UVM for Pipelined Protocols: In-Flight Tracking and Reorder Buffer Matching”
3) “Practical UVM Architecture for AXI-Like Out-of-Order Responses”
