---
title: "Pipelined Protocols in UVM with Out-of-Order (ID-Tagged) Completions"
date: 2026-02-03T21:25:59-0600
draft: false
summary: "A practical, end-to-end UVM pattern for verifying ID-tagged, out-of-order pipelined protocols: handshake-accurate monitors, an in-flight (ROB-style) scoreboard, backpressure stress, and debug hooks." 
tags: ["UVM", "SystemVerilog", "Verification", "Pipelining", "Out-of-Order", "Transaction IDs", "Scoreboard", "Monitors", "AXI", "Testbench Architecture"]
---

## Why pipelining + IDs is a different verification class

Non-pipelined protocols are easy to score: request → response → request → response. A single FIFO in the scoreboard is often enough.

Once you add **pipelining** (multiple outstanding requests) and **ID tagging** (responses can return **out of order**), the verification problem becomes:

> Track **many in-flight requests**, match **responses by ID**, enforce ordering rules (**often per-ID ordering**), and make failures debuggable.

The good news: there’s a clean, scalable UVM pattern for this. Think **reorder buffer (ROB)** + **reference model**.

---

## Protocol model (AXI-like, simplified)

To keep this concrete, assume two decoupled channels with standard valid/ready handshakes.

### Request channel
- `req_valid`, `req_ready`
- `req_id[ID_W-1:0]`
- `req_is_write`, `req_addr`, `req_data`

### Response channel
- `rsp_valid`, `rsp_ready`
- `rsp_id[ID_W-1:0]`
- `rsp_status`, `rsp_data`

### Rules (explicit assumptions)
1) Accepted request: `req_valid && req_ready`
2) Accepted response: `rsp_valid && rsp_ready`
3) Responses can be **out-of-order across IDs**
4) Responses are **in-order within the same ID** (common rule; change if your spec differs)
5) ID reuse is allowed, but bounded (e.g., `MAX_PER_ID` outstanding per ID)

These are intentionally “AXI-flavored” because they show up everywhere (interconnects, DMA engines, fabric protocols), even if your signals differ.

---

## Mental model: in-flight table + per-ID FIFO

When you accept a request, you create an **expected completion** record. When you accept a response, you match it to the oldest expected record for that ID.

```
Time →

REQ  id=3  ─────┐
REQ  id=7  ───┐ │
REQ  id=3  ─┐ │ │
            │ │ │
RSP      id=7 ◀┘ │   (OOO across IDs)
RSP         id=3 ◀──┘ (but FIFO within ID=3)
```

This is a reorder buffer, but “segmented” by ID.

---

## UVM architecture (minimal, robust)

- `driver/sequencer`: produce requests quickly (do **not** wait for completion)
- `req_monitor`: publishes accepted requests
- `rsp_monitor`: publishes accepted responses
- `scoreboard`:
  - in-flight tracking keyed by ID
  - reference model (even a simple memory) for data prediction
  - assertions/checks (timeouts, illegal ID reuse, unexpected responses)

```
               +--------------------+
sequence  ---> | uvm_sequencer      |
               +---------+----------+
                         |
                         v
               +--------------------+
               | uvm_driver         |  drives REQ
               +---------+----------+
                         |
                         v
                      [ DUT ]
                         ^
                         |
               +---------+----------+     +---------------------+
               | rsp_monitor        |---> | scoreboard (ROB)     |
               +--------------------+     |  expected_by_id[]    |
               +--------------------+---> |  ref model + checks  |
               | req_monitor        |     +---------------------+
               +--------------------+
```

**Principle:** the scoreboard should depend on monitors observing *real handshakes*, not on “what the driver tried to do”.

---

## Concrete interface (helps avoid hand-wavey examples)

```systemverilog
interface proto_if #(int ID_W = 4) (input logic clk);
  logic rst_n;

  logic              req_valid;
  logic              req_ready;
  logic [ID_W-1:0]    req_id;
  logic              req_is_write;
  logic [31:0]       req_addr;
  logic [31:0]       req_data;

  logic              rsp_valid;
  logic              rsp_ready;
  logic [ID_W-1:0]    rsp_id;
  logic [1:0]        rsp_status;
  logic [31:0]       rsp_data;
endinterface
```

---

## Transactions

Keep request/response items separate; it keeps monitors and scoreboards clean.

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

## Driver sketch (don’t accidentally serialize the pipeline)

A common anti-pattern is: “send one request, wait for response, send next”. That defeats pipelining.

Instead, your driver should:
- accept sequence items continuously
- drive `req_valid` until `req_ready`
- return immediately to the sequencer for the next item

```systemverilog
class proto_driver extends uvm_driver #(proto_req);
  `uvm_component_utils(proto_driver)

  virtual proto_if vif;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  function void build_phase(uvm_phase phase);
    if (!uvm_config_db#(virtual proto_if)::get(this, "", "vif", vif))
      `uvm_fatal("NOVIF", "vif not set")
  endfunction

  task run_phase(uvm_phase phase);
    proto_req tr;

    // idle defaults
    vif.req_valid <= 0;
    vif.req_id    <= '0;
    vif.req_is_write <= 0;
    vif.req_addr  <= '0;
    vif.req_data  <= '0;

    forever begin
      seq_item_port.get_next_item(tr);

      // drive until accepted
      vif.req_valid    <= 1;
      vif.req_id       <= tr.id;
      vif.req_is_write <= tr.is_write;
      vif.req_addr     <= tr.addr;
      vif.req_data     <= tr.data;

      do @(posedge vif.clk);
      while (!vif.rst_n || !(vif.req_valid && vif.req_ready));

      vif.req_valid <= 0;
      seq_item_port.item_done();
    end
  endtask
endclass
```

You can (and should) add response-side backpressure randomization in the test (toggle `rsp_ready`) to force reordering and queue buildup.

---

## Monitors: sample only accepted transfers

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

---

## Scoreboard: per-ID ROB + simple reference model

A scoreboard that feels “techie” (and saves debug time) typically has:

- **per-ID expected queues** (enforces per-ID ordering naturally)
- **sequence numbers** (global monotonic IDs for logging)
- **timeouts** (catch lost responses)
- a **reference model** (even a tiny one) so comparisons are meaningful

### Data structures

```systemverilog
typedef struct {
  proto_req     req;
  int unsigned  seq_no;
  time          accepted_time;
  bit [31:0]    exp_data;      // predicted for reads
  bit [1:0]     exp_status;    // predicted status
} expected_t;
```

### Minimal reference model: a memory map

For a simple read/write protocol, a memory model is enough to make the scoreboard useful:

- on write request accept: update mem
- on read request accept: compute expected data from mem

```systemverilog
class proto_scoreboard extends uvm_component;
  `uvm_component_utils(proto_scoreboard)

  uvm_analysis_imp #(proto_req, proto_scoreboard) req_imp;
  uvm_analysis_imp #(proto_rsp, proto_scoreboard) rsp_imp;

  localparam int ID_W = 4;
  localparam int NUM_IDS = 1 << ID_W;
  localparam int MAX_PER_ID = 8;
  localparam time TIMEOUT = 50_000ns;

  expected_t expected_by_id[NUM_IDS][$];

  // simple memory reference model
  bit [31:0] mem [bit [31:0]];

  int unsigned next_seq_no = 0;
  int unsigned total_inflight = 0;

  function new(string name, uvm_component parent);
    super.new(name, parent);
    req_imp = new("req_imp", this);
    rsp_imp = new("rsp_imp", this);
  endfunction

  // request path
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
    e.exp_status = 2'b00; // OK (default)

    if (t.is_write) begin
      mem[t.addr] = t.data;
      e.exp_data = '0;
    end else begin
      e.exp_data = mem.exists(t.addr) ? mem[t.addr] : '0;
    end

    expected_by_id[id].push_back(e);
    total_inflight++;

    `uvm_info("SB", $sformatf("REQ+ id=%0d seq=%0d qdepth=%0d inflight=%0d addr=0x%08x %s",
                              id, e.seq_no, expected_by_id[id].size(), total_inflight, t.addr,
                              t.is_write ? "W" : "R"),
              UVM_MEDIUM)
  endfunction

  // response path
  function void write(proto_rsp r);
    expected_t e;
    int id = r.id;

    if (expected_by_id[id].size() == 0) begin
      `uvm_error("SB", $sformatf("RSP with no matching REQ: id=%0d status=%0d data=0x%08x", id, r.status, r.data))
      return;
    end

    // FIFO within ID enforces per-ID ordering
    e = expected_by_id[id].pop_front();
    total_inflight--;

    // Status check
    if (r.status !== e.exp_status) begin
      `uvm_error("SB", $sformatf("STATUS mismatch: id=%0d seq=%0d exp=%0d got=%0d", id, e.seq_no, e.exp_status, r.status))
    end

    // Data check (only for reads)
    if (!e.req.is_write) begin
      if (r.data !== e.exp_data) begin
        `uvm_error("SB", $sformatf("DATA mismatch: id=%0d seq=%0d addr=0x%08x exp=0x%08x got=0x%08x",
                                   id, e.seq_no, e.req.addr, e.exp_data, r.data))
      end
    end

    `uvm_info("SB", $sformatf("RSP- id=%0d seq=%0d inflight=%0d", id, e.seq_no, total_inflight), UVM_MEDIUM)
  endfunction

  task run_phase(uvm_phase phase);
    forever begin
      #(200ns);
      check_timeouts();
    end
  endtask

  function void check_timeouts();
    for (int id = 0; id < NUM_IDS; id++) begin
      if (expected_by_id[id].size() > 0) begin
        expected_t oldest = expected_by_id[id][0];
        if (($time - oldest.accepted_time) > TIMEOUT) begin
          `uvm_error("SB", $sformatf("TIMEOUT: id=%0d oldest_seq=%0d age=%0t qdepth=%0d",
                                     id, oldest.seq_no, ($time-oldest.accepted_time), expected_by_id[id].size()))
        end
      end
    end
  endfunction
endclass
```

### Why this is a good “default” scoreboard

- **OOO across IDs**: handled (separate queues)
- **In-order within ID**: enforced (FIFO pop)
- **ID reuse corner cases**: caught (max depth per ID)
- **Meaningful comparisons**: memory model makes read checks real

If your protocol allows **out-of-order even within the same ID**, replace the per-ID FIFO with a per-ID associative structure keyed by a **tag/sequence** returned by the DUT.

---

## Stimulus that forces real out-of-order behavior

A pipelined/OOO test should:

- keep the request channel saturated
- vary IDs to create interleaving
- introduce response backpressure (`rsp_ready`) so completions bunch up
- occasionally concentrate traffic on a subset of IDs to stress per-ID depth

### Sequence sketch

```systemverilog
class pipelined_ooo_seq extends uvm_sequence #(proto_req);
  `uvm_object_utils(pipelined_ooo_seq)

  rand int unsigned num_reqs = 500;
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
        is_write dist {0:=6, 1:=4};
        addr[1:0] == 0;
        // bias to a few hot IDs occasionally
        if ($urandom_range(0,9) == 0) id inside {3,7,11};
      });

      start_item(tr);
      finish_item(tr);

      // keep it pipelined but not completely zero-gap
      if ($urandom_range(0,4) == 0) #(5ns);
    end
  endtask
endclass
```

### Response backpressure (practical, high value)

In your test, drive `rsp_ready` with randomness:

```systemverilog
// in a test or env component with access to vif
forever begin
  @(posedge vif.clk);
  if (!vif.rst_n) begin
    vif.rsp_ready <= 1'b0;
  end else begin
    // 80% ready, 20% backpressure
    vif.rsp_ready <= ($urandom_range(0,9) < 8);
  end
end
```

This alone often turns “looks fine in directed tests” into “oops, reorder bug”.

---

## Make it debug-friendly (the difference between 10 minutes and 2 days)

### 1) Dump in-flight state on error
Add a helper to print a compact ROB snapshot:

```
ID=3: [seq=41 addr=0x..., seq=58 addr=0x...]
ID=7: [seq=77 addr=0x...]
```

### 2) Track high-water marks
Track maximum `total_inflight` and per-ID depth seen; it’s a cheap “did we really stress it?” metric.

### 3) Add lightweight functional coverage
Cover what matters:

- ID distribution
- per-ID depth reaching max
- backpressure scenarios
- read-after-write hazards

---

## Common pitfalls (seen repeatedly)

1) **Sampling non-accepted transfers**
   - Only sample on `valid && ready`. Otherwise you’ll double-count.

2) **Driver accidentally serializes**
   - If driver waits for response, you’ll never see real reordering.

3) **Not matching the spec’s per-ID ordering rule**
   - Some protocols guarantee per-ID FIFO; others don’t. Score accordingly.

4) **Timeout that’s too small**
   - TIMEOUT must cover worst-case pipeline depth + backpressure + arbitration.

---

## Checklist: ship a solid pipelined/OOO UVM environment

- [ ] Monitors publish only on `valid && ready`
- [ ] Scoreboard tracks expected completions per ID (ROB-by-ID)
- [ ] Max outstanding per ID enforced
- [ ] Timeout watchdog present
- [ ] Reference model exists (even simple memory)
- [ ] Sequence keeps multiple requests in-flight (no serialization)
- [ ] Response backpressure randomized
- [ ] Debug hooks: seq_no, in-flight dump, high-water marks

---

## Suggested filename slug

`pipelined-protocols-uvm-out-of-order-ids.md`

## Alternate titles

1) “Verifying Out-of-Order, ID-Tagged Pipelines in UVM: A Practical Scoreboard Pattern”
2) “UVM for Pipelined Protocols: In-Flight Tracking (ROB) and ID-Based Matching”
3) “A Debuggable UVM Pattern for AXI-Like Out-of-Order Completions”
