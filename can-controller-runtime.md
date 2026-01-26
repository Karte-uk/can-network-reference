# CAN Controller Internals and Runtime Behavior

*Author: Percival Segui*

*Prepared as an independent technical reference.*

*This document is a personal technical reference documenting validated operating behavior and baseline configuration from a completed CAN network project.*

---

## Purpose:
Establish a correct mental model of bxCAN behavior at runtime.

This document explains what the CAN controller is doing, why certain failure modes occur, and how firmware structure must align with controller behavior.

It is not a bring-up checklist and not a physical-layer guide. Those topics are covered in separate documents.

---

### Scope

This document covers:

* bxCAN transmit and receive architecture
* hardware mailboxes and FIFOs
* interrupt-driven design constraints
* error counters and state transitions
* ACK behavior and its consequences
* runtime recovery mechanisms
* safe firmware patterns for CAN systems

This document does **not** cover:

* CubeMX pin or clock configuration
* CAN electrical wiring or termination
* CAN bit-timing heuristics

---

## Design Principle

> **bxCAN is an autonomous state machine.**
>
> Firmware does not "control" CAN directly; it *services* a controller that already enforces protocol rules.

Most CAN bring-up failures occur when firmware assumptions conflict with this reality.

---

## 1. bxCAN Architectural Overview

bxCAN consists of:

* 3 transmit mailboxes (hardware)
* 2 receive FIFOs (hardware)
* internal arbitration logic
* protocol error detection logic
* autonomous error-state machine

Firmware interacts through:

* register configuration
* interrupts
* message queues

Once enabled, the controller operates continuously and independently of the CPU.

---

## 2. Transmit Path Behavior

### 2.1 Hardware Mailboxes

bxCAN provides exactly **three transmit mailboxes**.

Each mailbox can hold:

* one complete CAN frame
* pending arbitration request

Important characteristics:

* Mailboxes are not FIFOs
* Transmission order is not guaranteed by insertion time
* Arbitration always occurs on the bus

If all three mailboxes are occupied:

* additional transmit requests fail
* HAL transmit calls return busy

---

### 2.2 HAL Transmit Model

`HAL_CAN_AddTxMessage()`:

* selects the first available mailbox
* loads identifier and payload
* sets transmission request

Completion is signaled via:

* `HAL_CAN_TxMailboxXCompleteCallback()`

Only after this callback fires is the mailbox truly free.

---

### 2.3 Critical Rule - Never Block on Transmit

Blocking patterns such as:

* waiting in a loop for mailbox availability
* transmitting inside an ISR

lead to:

* lost scheduling time
* cascading delays
* mailbox starvation

**Correct pattern:**

* enqueue frames into a software TX queue
* drain the queue opportunistically while mailboxes are free

The CAN controller must never be treated as a blocking peripheral.

---

## 3. Receive Path Behavior

### 3.1 Receive FIFOs

bxCAN implements:

* FIFO0
* FIFO1

Each FIFO can buffer multiple received frames.

Filters determine which FIFO a frame is routed into.

---

### 3.2 RX Pending Interrupt

When a frame arrives:

* controller stores frame in FIFO
* RX pending interrupt fires

HAL callback:

* `HAL_CAN_RxFifo0MsgPendingCallback()`

Inside this callback:

* the frame must be read immediately
* FIFO entry is released only after read

---

### 3.3 ISR Design Rule

> **ISRs must capture data and exit immediately.**

Never perform inside a CAN ISR:

* UART prints
* string formatting
* dynamic memory
* delays

Correct pattern:

* copy frame into a ring buffer
* return from ISR
* process in main loop

This preserves determinism and prevents interrupt latency escalation.

---

## 4. ACK Bit Semantics

### 4.1 What the ACK Bit Means

During every CAN frame:

* the transmitter sends the frame
* all receivers validate CRC
* any valid receiver asserts a dominant ACK bit

If **no node asserts ACK**:

* transmitter interprets this as failure
* transmit error counter (TEC) increments

Important:

* ACK does not require application acceptance
* any correctly configured CAN controller can ACK

---

### 4.2 Why ACK Errors Are Common

ACK loss occurs when:

* only one node is present
* transceiver is in standby
* wiring or ground is broken
* bit timing mismatches prevent frame decoding

The transmitter cannot distinguish these causes.

It only observes:

> “No one acknowledged my frame.”

---

## 5. Error Counters and States

### 5.1 Error Counters

bxCAN maintains:

* **TEC** - Transmit Error Counter
* **REC** - Receive Error Counter

Both range from 0 to 255.

---

### 5.2 Error States

| State         | Condition               |
| ------------- | ----------------------- |
| Error Active  | TEC < 128 and REC < 128 |
| Error Passive | TEC ≥ 128 or REC ≥ 128  |
| Bus-Off       | TEC ≥ 256               |

These transitions occur autonomously.

Firmware cannot prevent them.

---

### 5.3 Practical Interpretation

* Rising TEC almost always indicates ACK or physical-layer failure
* REC growth is less common during bring-up
* Bus-Off means the controller has *removed itself from the bus*

At Bus-Off:

* transmission stops
* RX may stop
* controller waits for recovery conditions

---

## 6. Last Error Code (LEC)

The controller records the **most recent protocol error**:

| Code | Meaning             |
| ---- | ------------------- |
| 0    | No error            |
| 1    | Stuff error         |
| 2    | Form error          |
| 3    | Acknowledge error   |
| 4    | Bit recessive error |
| 5    | Bit dominant error  |
| 6    | CRC error           |
| 7    | Software / invalid  |

Important behavior:

* LEC resets after read
* LEC resets on bus-off

**Implication:**

> Error information must be captured immediately.

Delayed logging permanently loses diagnostic signal.

---

## 7. Error Interrupt Handling

When an error occurs:

* `HAL_CAN_ErrorCallback()` is invoked

Inside this callback:

* read error status flags
* read LEC
* enqueue error snapshot

Do not print.

Error handling must follow the same ISR discipline as RX.

---

## 8. Auto Bus-Off Management (ABOM)

### 8.1 Hardware Behavior

When ABOM is enabled:

* controller waits for 128 × 11 recessive bits
* then automatically re-enters Error Active

At 500 kbps, this delay is on the order of milliseconds.

---

### 8.2 Software Layering

In practice:

* ABOM handles protocol-level recovery
* firmware must re-enable notifications and state

Common pattern:

* detect BOF flag
* stop CAN
* restart CAN
* re-enable interrupts

This provides belt-and-suspenders robustness.

---

## 9. Runtime Firmware Architecture Pattern

A safe CAN firmware structure:

```
ISR layer:
  RX interrupt → enqueue frame
  Error interrupt → enqueue error

Main loop:
  drain RX queue
  drain TX queue
  process errors
  print diagnostics
```

Key properties:

* ISRs are short
* UART is foreground only
* CAN controller is never blocked

---

## 10. Common Runtime Failure Patterns

| Symptom                  | Likely Cause             |
| ------------------------ | ------------------------ |
| TX stops, mailboxes full | Missing ACK              |
| TEC rising steadily      | Physical or timing issue |
| No RX interrupts         | Filters or wiring        |
| Immediate bus-off        | Single-node bus          |
| Recovery requires reset  | ABOM disabled            |

These symptoms originate from controller behavior, not firmware bugs.

---

## 11. Key Takeaways

* bxCAN operates autonomously
* firmware must be asynchronous
* mailboxes are scarce resources
* ACK loss dominates bring-up failures
* ISRs must never block
* LEC must be captured immediately
* ABOM is essential for unattended systems

Understanding these rules prevents most CAN debugging dead-ends.

---

### Document Status

This document defines the **runtime mental model** for CAN systems.

It should be read before:

* adding higher-level protocols
* increasing bus load
* optimizing throughput
* diagnosing persistent bus errors

Physical wiring and timing heuristics are intentionally handled elsewhere.

---

End of document.
