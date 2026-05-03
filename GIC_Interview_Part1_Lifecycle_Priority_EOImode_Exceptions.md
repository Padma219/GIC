# GIC Interview Questions — Part 1
## Interrupt Lifecycle, Priority, EOImode, Exceptions & Security

---

## Topic 1: Interrupt Lifecycle and GIC State Machine

---

### Q1: An edge-triggered SPI fires while the same interrupt is already ACTIVE. Walk through the GIC state transitions. What happens if software issues EOI only once?

**Simple Intuition:**
Think of edge-triggered like a doorbell — each press is a distinct event. The GIC "remembers" that a second ring happened even while you're answering the first one. But if you only acknowledge once, you lose one ring.

**Detailed Answer:**

**Initial State:** INTID 100 is Inactive, edge-triggered, enabled, routed to Core0.

**Flow:**

```
Time T0: Device asserts rising edge → INTID 100
         GIC State: Inactive → Pending
         GIC signals IRQ to Core0

Time T1: Core0 reads ICC_IAR (GICv3) / GICC_IAR (GICv2) → returns 100
         GIC State: Pending → Active
         Core0 enters ISR, starts handling

Time T2: Device fires SECOND rising edge (while INTID 100 is Active)
         GIC State: Active → Active+Pending
         (GIC latches the new edge — it's a NEW event)

Time T3: Core0 writes ICC_EOIR1 = 100 (EOImode=0)
         GIC State: Active+Pending → Pending
         (Priority drop + deactivate for the FIRST instance)
         GIC immediately re-signals IRQ to Core0

Time T4: Core0 reads ICC_IAR → returns 100 again
         GIC State: Pending → Active
         Core0 handles the second event

Time T5: Core0 writes ICC_EOIR1 = 100
         GIC State: Active → Inactive
```

**What if software only does ONE EOI at T3 and ignores the re-signaled IRQ?**

- GIC remains in Pending state
- If PSTATE.I is set (interrupts masked) and never cleared, the second interrupt is lost from software's perspective
- GIC continues to assert the IRQ line, but if software never acknowledges → interrupt stuck Pending forever

**Critical Detail — Edge vs Level here:**

| Aspect | Edge-triggered | Level-triggered |
|--------|---------------|-----------------|
| Active → Active+Pending | Only on NEW edge detection | As long as HW line stays asserted |
| After EOI | Pending (from latched edge) | Pending (if line still high) |
| Lost interrupt risk | If edge arrives in Inactive state with enable=0 | Lower risk — line stays asserted |

**GICv2 vs GICv3 difference:**
- GICv2: `GICC_IAR` read acknowledges, `GICC_EOIR` write completes
- GICv3: `ICC_IAR1_EL1` (Group 1) or `ICC_IAR0_EL1` (Group 0), `ICC_EOIR1_EL1`/`ICC_EOIR0_EL1`
- State machine behavior is identical — only register interface differs

**Common Mistakes:**
- Believing edge-triggered can't go to Active+Pending
- Thinking a single EOI clears both events
- Confusing "edge is latched" with "level is re-sampled"

**What Interviewer Expects:**
- Clear state machine walk-through with exact transitions
- Understanding that edge events are *latched* not *re-sampled*
- Knowledge that Active+Pending → Pending on EOI (not → Inactive)

---

### Q2: A level-sensitive interrupt source de-asserts its line DURING the ISR, before EOI. What state does the GIC transition to after EOI? What if it de-asserts AFTER ICC_IAR read but BEFORE the ISR clears the device?

**Simple Intuition:**
Level-triggered is like a smoke alarm — it keeps screaming as long as there's smoke. The GIC continuously *samples* the line. If smoke clears (line de-asserts) while you're handling it, the GIC notices.

**Detailed Answer:**

**Scenario A: Line de-asserts BEFORE EOI**

```
T0: Device asserts line → GIC: Inactive → Pending → CPU reads IAR → Active
T1: ISR clears device register → device de-asserts line
    GIC samples line: LOW
    GIC State remains: Active (not Active+Pending, because line is low)
T2: CPU writes EOI
    GIC State: Active → Inactive (NOT Pending, because line is low at sample point)
    No re-trigger. Clean completion.
```

**Scenario B: Line stays asserted THROUGH EOI**

```
T0: Device asserts line → GIC: Inactive → Pending → CPU reads IAR → Active
T1: ISR does NOT clear device (bug or intentional)
    GIC continuously samples line: HIGH
    GIC State: Active+Pending (line still high → re-pended)
T2: CPU writes EOI
    GIC State: Active+Pending → Pending (line still high)
    GIC immediately re-signals IRQ → INTERRUPT STORM if ISR never clears device
```

**Scenario C (the tricky one): Line de-asserts AFTER IAR read but BEFORE ISR clears device**

This is actually impossible to distinguish from Scenario A from the GIC's perspective. The GIC samples the line:
- If line is LOW when GIC samples → state is just Active
- If line is HIGH when GIC samples → state is Active+Pending

The "when" of de-assertion relative to IAR doesn't matter — what matters is the line level at GIC's sampling points.

**Key Architectural Point:**
> For level-sensitive interrupts, the Pending state is determined by the **current level** of the signal, not by a latched event. The GIC re-evaluates continuously (or at defined sample points, implementation-specific).

**Common Mistakes:**
- Thinking level-sensitive interrupts "latch" like edge (they don't)
- Not understanding that Active+Pending for level means "line is still high"
- Missing the interrupt storm scenario when device isn't cleared before EOI

**What Interviewer Expects:**
- Clear distinction between edge (latched) and level (sampled) Pending behavior
- Understanding that interrupt storm root cause is "EOI without clearing device"
- Ability to reason about timing: what the GIC "sees" at its sample point

---

### Q3: You have an edge-triggered interrupt. The device fires an edge, but at that exact moment the interrupt is disabled in GICD_ISENABLER. What happens? Is the interrupt lost forever?

**Simple Intuition:**
The GIC's enable bit is like a gate. For edge-triggered: if the gate is closed when the doorbell rings, the ring is **never recorded**. Unlike level (where the alarm keeps screaming and you'll hear it when you open the gate).

**Detailed Answer:**

**Edge-triggered, interrupt disabled:**

```
T0: GICD_ICENABLER has INTID 50 disabled (bit = 0)
T1: Device fires rising edge on INTID 50
    GIC behavior: Edge is DISCARDED. No state transition.
    State remains: Inactive

T2: Software writes GICD_ISENABLER to enable INTID 50
    State: Still Inactive — the edge is gone, never latched

    → INTERRUPT LOST PERMANENTLY
```

**Level-triggered, interrupt disabled:**

```
T0: GICD_ICENABLER has INTID 50 disabled
T1: Device asserts line (goes HIGH)
    GIC: Does not transition to Pending (disabled)

T2: Software writes GICD_ISENABLER to enable INTID 50
    GIC samples line: still HIGH
    GIC State: Inactive → Pending
    → Interrupt NOT lost — picked up when enabled
```

**This is THE classic lost interrupt scenario in real SoCs:**

Real-world trigger:
1. Driver calls `disable_irq()` to mask during critical section
2. Device fires edge during masked window
3. Driver calls `enable_irq()` 
4. Interrupt never arrives → device hangs

**Mitigation Strategies:**
- Design edge devices with a "status register" that software polls after re-enable
- Use level-triggered for devices that can hold their line
- Some SoCs add a "missed edge" sticky bit (non-standard, implementation-specific)

**GICv2 vs GICv3:** Behavior is identical — this is architectural.

**Common Mistakes:**
- Assuming disabled just delays delivery (true for level, false for edge)
- Not understanding the fundamental asymmetry between edge and level regarding enable/disable
- Believing GICD_ISPENDR can help (you'd have to *know* you missed an edge — chicken-and-egg)

**What Interviewer Expects:**
- Immediate recognition that edge + disabled = lost
- Contrast with level behavior
- Real-world mitigation strategies (polling status register, design choice of level vs edge)
- Understanding this drives real SoC bugs in production

---

## Topic 2: Priority and Preemption

---

### Q4: Core0 is handling INTID 32 (priority 0x40). INTID 33 (priority 0x20) goes Pending. ICC_BPR1 = 3. Does preemption occur? Show the math.

**Simple Intuition:**
BPR (Binary Point Register) defines how many bits of the priority field form the "group priority" that can preempt. Think of it as: "How different does the new interrupt need to be to interrupt what I'm already doing?"

Higher BPR = fewer bits for group priority = harder to preempt (coarser grouping).

**Detailed Answer:**

**Setup:**
- Active interrupt: INTID 32, priority = 0x40 (binary: 0100 0000)
- Pending interrupt: INTID 33, priority = 0x20 (binary: 0010 0000)
- ICC_BPR1 = 3

**Step 1: Determine group priority field**

BPR value defines the split point. For GICv3 with 8-bit priority:

```
BPR = 3 means: Group priority = bits[7:4], Subpriority = bits[3:0]
(Formula: group priority bits = 8 - BPR, but minimum group = bits[7:N+1] where N=BPR)

Actually, the ARM spec defines:
- BPR = 0: [7:1] group, [0] sub  → 7 bits group
- BPR = 1: [7:2] group, [1:0] sub → 6 bits group
- BPR = 2: [7:3] group, [2:0] sub → 5 bits group
- BPR = 3: [7:4] group, [3:0] sub → 4 bits group
```

**Step 2: Calculate group priorities**

```
Running priority (INTID 32): 0x40 = 0100 0000
  Group priority (bits[7:4]): 0100 = 4

Pending priority (INTID 33): 0x20 = 0010 0000
  Group priority (bits[7:4]): 0010 = 2
```

**Step 3: Preemption check**

Rule: Preemption occurs if `pending_group_priority < running_group_priority`
(Lower numeric value = higher priority)

```
Pending group (2) < Running group (4)?  → YES → PREEMPTION OCCURS
```

**Step 4: What happens**

```
1. GIC signals IRQ to Core0 (preemption signal)
2. Core0 takes IRQ exception (if PSTATE.I = 0)
3. Software reads ICC_IAR1 → returns 33
4. Running priority (ICC_RPR) updates to 0x20
5. INTID 32 remains Active, INTID 33 now also Active
6. Software handles INTID 33, writes EOI for 33
7. Running priority drops back to 0x40
8. Software returns, continues handling INTID 32, writes EOI for 32
```

**What if BPR = 7?**

```
BPR = 7: Group priority = bit[7] only, subpriority = bits[6:0]
Running:  0x40 → bit[7] = 0
Pending:  0x20 → bit[7] = 0
Same group priority → NO PREEMPTION (even though 0x20 is "higher priority")
```

**Key Insight:** BPR doesn't change which interrupt is higher priority — it changes whether that priority difference is *enough* to preempt.

**Common Mistakes:**
- Confusing BPR with PMR (PMR masks, BPR controls preemption granularity)
- Getting the split formula wrong
- Forgetting that subpriority differences NEVER cause preemption
- Thinking BPR affects delivery order of non-preempting interrupts (it doesn't — full priority is used for ordering)

**What Interviewer Expects:**
- Correct BPR math with binary breakdown
- Clear distinction: "group priority preempts, subpriority orders within same group"
- Understanding that BPR is the software knob for nesting depth control

---

### Q5: Two SPIs (INTID 60 and INTID 61) are configured with EQUAL priority (both 0x80). Both go Pending simultaneously. Who gets served first? Who decides this?

**Simple Intuition:**
GIC must break the tie somehow — two interrupts can't both be "first." The architecture says: **this is IMPLEMENTATION DEFINED.** The GIC vendor (ARM's IP, or Qualcomm's custom, etc.) decides the tie-breaking rule.

**Detailed Answer:**

**What the ARM Architecture Spec says:**
> "If two pending interrupts have the same priority, it is IMPLEMENTATION DEFINED which is presented first."

**Common implementation choices (vendor-specific):**

| Implementation | Tie-breaking rule |
|----------------|-------------------|
| ARM GIC-500 | Lower INTID wins |
| ARM GIC-600 | Lower INTID wins |
| Some custom GICs | Round-robin among equal priority |
| Others | Fixed priority by INTID number |

**For the scenario:**

```
INTID 60 priority = 0x80
INTID 61 priority = 0x80
Both pending on Core0

Most ARM reference implementations: INTID 60 wins (lower INTID)
```

**But who decides the PRIORITY VALUE itself (0x80)?**

This is a layered answer:

```
Layer 1: Architecture defines the MECHANISM
  → GICD_IPRIORITYR[n] register holds priority for INTID n
  → 8-bit field (or fewer implemented bits, min 4 in GICv3)
  → Lower value = higher priority

Layer 2: SoC integrator defines RESET VALUES
  → Reset value is typically 0x00 or implementation-defined
  → SoC spec may define "INTID 32 must be priority 0x10" — this is a SYSTEM DESIGN choice

Layer 3: Software (firmware/OS) programs RUNTIME values
  → Trusted Firmware / bootloader may set Group 0 priorities
  → Linux GIC driver typically sets ALL interrupts to the SAME priority (0xA0)
  → Effectively: Linux does NOT use hardware priority for scheduling
```

**Why does Linux set flat priorities?**

Because the kernel's softirq/threaded-irq mechanism handles scheduling in software. Hardware priority would complicate kernel assumptions about interrupt handling order.

**Real Interview Follow-up:** "So if Linux sets all priorities equal, what actually determines interrupt handling order?"
- Answer: Implementation-defined tie-breaking (usually INTID order) for simultaneous arrival
- For non-simultaneous: first-come-first-served (whichever goes Pending first gets acknowledged first)

**Common Mistakes:**
- Saying "lower INTID always wins" without qualifying it as implementation-specific
- Not knowing that Linux flattens priorities
- Confusing "who programs GICD_IPRIORITYR" (software) with "who decides the policy" (system architect)

**What Interviewer Expects:**
- Clear separation of architecture (mechanism) vs implementation (tie-breaking) vs software (policy)
- Knowledge that priority assignment is a system design decision, not hardcoded in silicon
- Understanding that equal priority is common in Linux and the implications

---

### Q6: Explain late arrival in GIC context. INTID 40 (priority 0x80) just went Pending and Core0 is about to read ICC_IAR. Before the read completes, INTID 41 (priority 0x10) goes Pending. What does ICC_IAR return?

**Simple Intuition:**
"Late arrival" means a higher-priority interrupt sneaks in after a lower-priority one triggered the CPU exception but before software actually reads IAR. The GIC always returns the HIGHEST priority pending interrupt at the moment of the read — not necessarily the one that originally caused the IRQ.

**Detailed Answer:**

**Flow:**

```
T0: INTID 40 (prio 0x80) goes Pending
    GIC asserts IRQ to Core0
    
T1: Core0 takes IRQ exception, enters vector handler
    (CPU saves context, jumps to IRQ vector)
    
T2: INTID 41 (prio 0x10) goes Pending  ← LATE ARRIVAL
    (Before software reads ICC_IAR)
    
T3: Software reads ICC_IAR1_EL1
    GIC evaluates: highest pending = INTID 41 (0x10 < 0x80)
    Returns: INTID 41
    GIC State: INTID 41 → Active, INTID 40 → remains Pending
    
T4: Software handles INTID 41, writes EOI
    GIC State: INTID 41 → Inactive
    Running priority drops → INTID 40 now can signal
    GIC re-asserts IRQ for INTID 40
    
T5: Software reads ICC_IAR → returns 40
    Normal handling continues
```

**Key Architectural Guarantee:**
> ICC_IAR always returns the highest priority pending interrupt at the moment of the read. There is no "lock-in" from the original signaling event.

**Why this matters for verification:**
- You must test races between new interrupts arriving and IAR reads
- The assertion signal and the IAR read are not atomic — anything can change in between

**GICv2 identical behavior:** GICC_IAR exhibits the same late-arrival semantics.

**Common Mistakes:**
- Assuming the interrupt that caused the exception is guaranteed to be returned by IAR
- Not handling the case where IAR returns a different INTID than expected
- Missing that INTID 40 will cause a second IRQ after INTID 41's EOI

**What Interviewer Expects:**
- Understanding that IRQ assertion and IAR read are decoupled events
- The GIC is a dynamic priority arbiter, not a FIFO
- Correct sequencing of what stays Pending vs what goes Active
- Verification angle: this is a critical race to cover in testbench

---
📘 Key Idea
The CPU exception entry is hardware‑driven and happens as soon as the IRQ signal is recognized and not masked.
At that moment, the CPU hasn’t yet interacted with the GIC registers — it just reacts to the assertion of the IRQ line.

The IAR read happens later, inside software, after the CPU has already saved state and jumped to the vector.

🔹 Stage-by-Stage Timeline


Initial:    PSTATE.I = 0   (IRQ unmasked)
Time	Event	Hardware action
T0	INTID 40 (prio 0x80) becomes pending	GIC asserts IRQ to core
T1	CPU samples IRQ=1 → begins exception entry	
• Saves PC → ELR_EL1	
• Saves PSTATE → SPSR_EL1	
• Updates PSTATE bits (I=1,F=1,A=1,D=1) → mask everything	
• Loads PC ← IRQ vector address	
T2	(Still finishing those micro‑ops) another interrupt arrives — INTID 41 with prio 0x10	GIC sees new higher priority pending; IRQ line already high
(The core is already committed to enter the IRQ vector.)	
T3	CPU starts executing vector code — handlers entry instructions	Software not yet read ICC_IAR1_EL1
T4	Handler executes intid = MRS ICC_IAR1_EL1	GIC chooses the highest‑priority pending at this instant.
INTID 41 has higher priority → GIC returns 41; marks it ACTIVE, leaves 40 pending.
T5	Software handles INTID 41, writes EOI	GIC drops 41 → inactive, RPR updated, finds next pending (INTID 40) → re‑asserts IRQ
T6	CPU, still in IRQ mode, sees IRQ line re‑asserted once PSTATE.I cleared → takes next interrupt, IAR returns 40.	
🔹 Why This Works Even Though PSTATE.I=1 During Entry
When the CPU starts exception entry, hardware sets PSTATE.I=1 to mask further IRQs until software deliberately unmasks.
But that mask only affects further exception entries, not what ICC_IAR reads.
At the moment you read IAR:

The CPU is already inside the IRQ handler (so IRQ masking doesn’t block the IAR read).
GIC’s logic decides, “Which pending interrupt should I hand to the CPU now?”
It doesn’t “remember” which one first caused the entry.
It simply returns whichever has the highest priority among current pending ones in the system.
🔹 What Happens to State (SPSR/ELR)
During exception entry:

Register	Set by	Contains
ELR_EL1	Hardware	Return address (instruction after the one interrupted)
SPSR_EL1	Hardware	Copy of prior PSTATE (NZCV, SPSEL, DAIF, EL, etc.)
PSTATE	Hardware updates	DAIF all set → exceptions masked; EL bits change to handler EL
These happen before any instruction executes in software.
Neither ELR_EL1 nor SPSR_EL1 depend on which interrupt will finally be reported by the IAR.
They only describe the context you were in when the IRQ line was taken.

🔹 The “Late Arrival” Summary in Human Words
The CPU reacts to “IRQ line high” — not to a specific interrupt source.
The GIC later tells it, at IAR read time, which interrupt currently deserves service.
If a higher‑priority one sneaks in before the read, the GIC upgrades the answer.

🔹 Mental Model Diagram
markdown


```text
T0   T1         T2          T3             T4            T5
│    │          │           │              │             │
│ INT40 pend    │           │              │             │
│ IRQ asserted  │           │              │             │
│──────────────▶│ Exception │              │             │
│               │ entry     │              │             │
│               │ (save ELR,│              │             │
│               │  SPSR)    │              │             │
│               │           │ INT41 pend   │             │
│               │           │ (higher prio)│             │
│               │           │────────────▶ │             │
│               │           │              │ IAR read →41│
│               │           │              │             │
│               │           │              │ EOI(41)     │
│               │           │              │             │
│               │           │              │ IRQ re‑assert│
│               │           │              │ IAR read →40 │
│               │           │              │             │
```
✅ So yes:

Late arrivals can be reported instead of the original trigger.
The CPU doesn’t need to re‑enable interrupts for that to happen; it will get the new INTID from the first IAR read.
Once that higher‑priority interrupt finishes and is EOId, the earlier one (still pending) will be reported next.

## Topic 3: EOImode Behavior

---

### Q7: Explain EOImode=0 vs EOImode=1 with exact GIC state transitions. Then explain why a hypervisor MUST use EOImode=1.

**Simple Intuition:**

EOI (End of Interrupt) does two things conceptually:
1. **Priority drop** — "I'm no longer occupied with this priority level" (allows preemption by same/lower priority)
2. **Deactivate** — "This interrupt is truly done" (moves from Active → Inactive)

- **EOImode=0:** Both happen in one write (simple, used by bare-metal/OS)
- **EOImode=1:** Split into two writes — EOIR does priority drop only, DIR does deactivate (needed for hypervisors)

**Detailed Answer:**

**EOImode=0 (combined):**

```
Register: ICC_EOIR1_EL1 write (GICv3) / GICC_EOIR (GICv2)
Effect:   Priority drop + Deactivate in single operation

State: Active → Inactive (one step)
       Active+Pending → Pending (one step)

Running priority immediately drops to next active or idle priority.
```

**EOImode=1 (split):**

```
Step 1: ICC_EOIR1_EL1 write
Effect: Priority drop ONLY
State:  Active → Active (still active! just priority dropped)
        Running priority drops

Step 2: ICC_DIR_EL1 write  
Effect: Deactivate ONLY
State:  Active → Inactive
```

**Why hypervisors NEED EOImode=1:**

```
Scenario without split (EOImode=0) — THE PROBLEM:

1. Physical INTID 100 fires → Hypervisor (EL2) gets IRQ
2. Hypervisor reads ICC_IAR → INTID 100 goes Active
3. Hypervisor injects VIRTUAL interrupt to guest via List Register
4. Hypervisor writes ICC_EOIR → INTID 100 goes Inactive ← PROBLEM!
   
   Now: Physical INTID 100 is Inactive
   But:  Guest hasn't handled it yet!
   
   If device fires AGAIN → GIC takes it as NEW interrupt
   → Double delivery / state corruption
```

```
Scenario with split (EOImode=1) — THE SOLUTION:

1. Physical INTID 100 fires → Hypervisor (EL2) gets IRQ
2. Hypervisor reads ICC_IAR → INTID 100 goes Active
3. Hypervisor writes ICC_EOIR → Priority drop (can handle other interrupts)
   Physical INTID 100 is STILL Active (not yet deactivated)
4. Hypervisor injects virtual interrupt to guest via LR
5. Guest handles virtual interrupt, writes virtual ICC_EOIR
6. This traps to hypervisor (or HW bit in LR handles it)
7. NOW hypervisor writes ICC_DIR → Physical INTID 100 → Inactive

   Physical interrupt stays Active until guest truly finishes
   → No double delivery, no state corruption
```

**ASCII State Diagram:**

```
EOImode=0:                     EOImode=1:
                               
Active ──EOIR──→ Inactive      Active ──EOIR──→ Active (prio dropped)
                                        │
                                        ──DIR──→ Inactive
```

**Register Summary:**

| Mode | Priority Drop | Deactivate | Use Case |
|------|--------------|------------|----------|
| EOImode=0 | ICC_EOIR | ICC_EOIR (same write) | OS, bare-metal |
| EOImode=1 | ICC_EOIR | ICC_DIR (separate write) | Hypervisor |

**GICv2 vs GICv3:**
- GICv2: GICC_CTLR.EOImode, GICC_EOIR, GICC_DIR
- GICv3: ICC_CTLR_EL1.EOImode, ICC_EOIR1_EL1, ICC_DIR_EL1
- Semantics identical, only register names/access mechanism differ

**Common Mistakes:**
- Thinking EOImode=1 means "don't drop priority" (wrong — EOIR still drops priority)
- Not understanding why keeping the interrupt Active matters for virtualization
- Confusing virtual EOI (guest writes) with physical EOI (hypervisor writes)
- Forgetting that Active state blocks same-interrupt re-delivery

**What Interviewer Expects:**
- Full state transition with both modes
- Clear articulation of the virtualization problem
- Understanding that "Active" means "blocked from re-signaling"
- Knowing which register does what in each mode

---

## Topic 4: Exception Handling vs Interrupts

---

### Q8: An SError (async abort) and an IRQ arrive at the same cycle. Does SError go through GIC? Which one does the CPU take first? What does software see?

**Simple Intuition:**

Think of it this way:
- **IRQ/FIQ** = "Someone rang the doorbell" → GIC manages this
- **SError** = "The house is on fire" → Nothing to do with the doorbell system (GIC)

SError is an asynchronous *abort*, not an *interrupt*. It doesn't go through GIC at all. It's generated by the memory system (bus errors, ECC failures, parity errors) and delivered directly to the CPU via the SError exception vector.

**Detailed Answer:**

**Does SError use GIC?**
**NO.** Absolutely not.

```
SError source: Memory system / interconnect / slave device
SError path:   Bus → CPU core directly → SError exception vector
GIC involvement: ZERO

IRQ source:    Peripheral device
IRQ path:      Device → GIC (Distributor → Redistributor → CPU Interface) → CPU IRQ signal
```

**Will ICC_IAR be read for SError?**
**NO.** ICC_IAR is only for GIC-managed interrupts (IRQ/FIQ). SError has no INTID, no priority, no GIC state.

**What if both arrive simultaneously?**

ARM architecture defines exception priority (from ARM ARM):

```
Priority (highest first):
1. SError (asynchronous abort)
2. IRQ
3. FIQ  
(Note: this is the synchronization priority — which gets "taken" first)

Wait — actually the precise rule is more nuanced:
```

**Precise Architecture Rule:**

Both SError and IRQ are asynchronous exceptions. When both are pending:

```
1. CPU checks PSTATE.A (SError mask) and PSTATE.I (IRQ mask)
2. If both unmasked: SError is taken FIRST (higher priority)
3. After SError handler entry:
   - PSTATE.I is SET (IRQs masked by default on exception entry)
   - PSTATE.A is SET (SError masked)
   - IRQ remains pending at GIC level
4. When SError handler unmasks IRQ (clears PSTATE.I): IRQ is taken
```

**What software sees:**

```
Cycle N: Both SError + IRQ arrive
Cycle N+x: CPU takes SError exception
  → Vector: VBAR + 0x380 (SError, current EL, SPx) [AArch64]
  → ESR_EL1/ESR_EL2: syndrome for async abort
  → NO ICC_IAR read needed or possible
  → IRQ is pending but masked (PSTATE.I=1)

After SError handler completes (ERET):
  → PSTATE restored, if PSTATE.I=0 → IRQ taken immediately
  → CPU enters IRQ vector
  → NOW ICC_IAR is read → returns INTID
```

**Critical Point for SoC Debug:**
SError is often **imprecise** — by the time you see it, the offending instruction is long past. The ESR syndrome may give limited info. This makes SError + IRQ scenarios particularly nasty to debug because:
- SError handler may corrupt state needed by IRQ handler
- SError may indicate the *device* that was supposed to IRQ is broken
- The IRQ that follows may itself be related to the bus error

**Async Abort vs SError:**
In ARMv8/AArch64, SError IS the async abort. They're the same thing (terminology unified from ARMv7 where "async abort" was the name).

**Common Mistakes:**
- Thinking SError goes through GIC (it doesn't)
- Trying to read ICC_IAR in an SError handler
- Believing SError is FIQ (it's not — FIQ is GIC Group 0)
- Not understanding that SError masks on exception entry affect subsequent IRQ delivery

**What Interviewer Expects:**
- Immediate clarity: "SError has nothing to do with GIC"
- Correct exception priority ordering
- Understanding of PSTATE masking on exception entry
- Real-world appreciation that SError + IRQ combination is a debug nightmare

---

### Q9: In a TrustZone system, a Group 0 interrupt fires while the CPU is executing Non-secure code at EL1. Walk through the full delivery path. What happens if Non-secure OS tries to read ICC_IAR0?

**Simple Intuition:**

Group 0 = "Secure world's doorbell." If it rings while you're in Non-secure world, you get yanked into Secure world (EL3) via FIQ. The Non-secure OS never sees it, never touches it.

**Detailed Answer:**

**System configuration:**
- GICv3, ARE=1 (affinity routing enabled)
- SCR_EL3.FIQ=1 (FIQ routed to EL3)
- SCR_EL3.IRQ=0 (IRQ handled at current EL)
- INTID 32 = Group 0, priority 0x00

**Delivery Flow:**

```
T0: Secure device fires → INTID 32 goes Pending (Group 0)
    GIC evaluates: Group 0 → signal as FIQ

T1: CPU is at NS-EL1, PSTATE.F=0 (FIQ unmasked)
    CPU takes FIQ exception → TARGETS EL3 (due to SCR_EL3.FIQ=1)

T2: Exception entry to EL3:
    - SPSR_EL3 saves NS-EL1 state
    - ELR_EL3 saves return address
    - SCR_EL3.NS examined to determine source was Non-secure
    - PSTATE: DAIF all set (all exceptions masked)
    - Vector: VBAR_EL3 + 0x480 (FIQ, lower EL, AArch64)

T3: EL3 firmware (ATF/TF-A) reads ICC_IAR0_EL1
    → Returns INTID 32
    → GIC State: Pending → Active
    → GIC switches to Secure access mode for this CPU interface

T4: EL3 handles or delegates to S-EL1 (Secure OS like OP-TEE)

T5: Handler writes ICC_EOIR0_EL1 = 32
    → GIC State: Active → Inactive
    → ERET back to NS-EL1 (original code resumes)
```

**What if Non-secure OS tries to read ICC_IAR0_EL1?**

```
Scenario: NS-EL1 code executes: MRS X0, ICC_IAR0_EL1

Result depends on ICC_SRE_EL3 and SCR_EL3 configuration:

Option A (typical): Access TRAPS to EL3
  → EL3 firmware gets Secure register access trap
  → Can inject an abort or return 1023

Option B (if accessible): Returns 1023 (spurious)
  → Group 0 interrupts are not visible to NS world
  → GIC returns special INTID 1023 = no valid interrupt

Option C (architecturally): IMPLEMENTATION DEFINED / UNDEFINED
  → Some implementations may return 0 or RAZ/WI
```

**Security Guarantee:**
The architecture ensures Non-secure software **cannot**:
- Read Group 0 interrupt state
- Acknowledge Group 0 interrupts
- Modify Group 0 interrupt configuration (GICD registers are banked/filtered)

**Group Mapping Summary (GICv3 with EL3):**

| Group | Exception | Target EL | Register |
|-------|-----------|-----------|----------|
| Group 0 | FIQ | EL3 | ICC_IAR0_EL1 |
| Group 1 Secure | IRQ (at S-EL1) or FIQ (at NS) | S-EL1 or EL3 | ICC_IAR1_EL1 |
| Group 1 Non-secure | IRQ | NS-EL1/EL2 | ICC_IAR1_EL1 |

**Common Mistakes:**
- Thinking Group 0 → IRQ (it maps to FIQ in GICv3)
- Not knowing that FIQ routes to EL3 when SCR_EL3.FIQ=1
- Believing NS software can access Group 0 via ICC_IAR0
- Confusing GICv2 (Group 0 → IRQ or FIQ, configurable) with GICv3 (Group 0 → always FIQ)

**GICv2 vs GICv3 Difference (important!):**
- GICv2: Group 0 can be signaled as IRQ OR FIQ (controlled by GICC_CTLR.FIQEn)
- GICv3: Group 0 → ALWAYS FIQ, Group 1 → ALWAYS IRQ (hardwired mapping)

**What Interviewer Expects:**
- Complete exception routing knowledge (SCR_EL3 bits)
- Understanding that Group 0 is the "secure interrupt" mechanism
- GICv2 vs GICv3 group-to-signal mapping difference
- Security implications of register access from wrong world

---

## Topic 5: Verification Perspective

---

### Q10: You're designing a verification testbench for GIC interrupt state machine coverage. What are the critical state transitions you MUST cover, and what corner cases would catch most silicon bugs?

**Simple Intuition:**
The GIC state machine has 4 states and multiple transitions between them. The interesting bugs live in the *edges* (transitions), especially ones triggered by unlikely timing combinations.

**Detailed Answer:**

**State Machine to Cover:**

```
           ┌─────────────────────────────────────────┐
           │                                         │
           ▼                                         │
     ┌──────────┐    IAR read     ┌────────┐      EOI (EOImode=0)
     │ INACTIVE │───────────────→ │ ACTIVE │────────┘
     └──────────┘                 └────────┘
           │                          │  ▲
           │  Signal assert           │  │  New edge/level while active
           │  (edge or level=HIGH)    │  │
           ▼                          ▼  │
     ┌──────────┐    IAR read     ┌─────────────────┐
     │ PENDING  │───────────────→ │ ACTIVE+PENDING  │
     └──────────┘                 └─────────────────┘
           ▲                              │
           │          EOI (EOImode=0)     │
           └──────────────────────────────┘
```

**Critical Transitions to Cover:**

| # | Transition | Trigger | Why critical |
|---|-----------|---------|--------------|
| 1 | Inactive → Pending | Edge/Level | Basic functionality |
| 2 | Pending → Active | IAR read | Basic acknowledge |
| 3 | Active → Inactive | EOI (mode=0) | Normal completion |
| 4 | Active → Active+Pending | New event while active | Re-trigger |
| 5 | Active+Pending → Pending | EOI (mode=0) | Must re-signal IRQ |
| 6 | Active+Pending → Active | Level de-asserts while active | Level-specific |
| 7 | Pending → Inactive | Disable (ICENABLER) | Interrupt disabled while pending |
| 8 | Active → Active (prio drop) | EOIR (mode=1) | Split EOI |
| 9 | Active → Inactive | DIR (mode=1) | Deactivate only |

**Corner Cases That Catch Bugs:**

**1. IAR read during state change (race):**
```
Stimulus: Assert edge → same cycle as IAR read
Check:    Does IAR return INTID or 1023?
Bug found: Off-by-one cycle in pending latch vs arbitration
```

**2. Disable during Active+Pending:**
```
Stimulus: INTID in Active+Pending → write GICD_ICENABLER
Check:    Active state preserved? Pending cleared?
Expected: Active portion remains, Pending cleared for edge; 
          for level, Pending re-evaluated on re-enable
Bug found: Some implementations incorrectly clear Active on disable
```

**3. Priority change while Pending:**
```
Stimulus: INTID pending at prio 0x80 → software writes GICD_IPRIORITYR to 0x10
Check:    Does arbitration use new priority?
Expected: Architecture says priority change on pending is UNPREDICTABLE
Bug found: Implementations that update vs. those that don't — verify YOUR impl matches spec
```

**4. EOI for wrong INTID:**
```
Stimulus: INTID 32 Active → software writes ICC_EOIR1 = 33 (wrong!)
Check:    What happens?
Expected: Architecturally UNPREDICTABLE, but verify impl doesn't corrupt state
Bug found: State machine corruption leaking to other INTIDs
```

**5. Simultaneous SGI from multiple cores:**
```
Stimulus: Core1 and Core2 both send SGI 5 to Core0 in same cycle
Check:    Does Core0 see it once or twice?
Expected: SGI in GICv3 is edge-triggered per source, so potentially twice
          (GICv2: different — SGI pending is per-source with GICD_CPENDSGIR)
Bug found: Source tracking logic under simultaneous write
```

**6. 1023 (spurious) scenarios:**
```
Must verify 1023 returned when:
- PMR masks all pending interrupts
- Pending interrupt is disabled between signal and IAR read
- Pending interrupt is moved to another core between signal and IAR read (SPI)
- No interrupt pending (signal was a glitch)
```

**Coverage Model Recommendation:**

```systemverilog
// Functional coverage bins
covergroup gic_state_transitions;
  state: coverpoint current_state {
    bins inactive = {INACTIVE};
    bins pending  = {PENDING};
    bins active   = {ACTIVE};
    bins act_pend = {ACTIVE_PENDING};
  }
  
  transition: coverpoint state_transition {
    bins all_valid[] = (INACTIVE => PENDING),
                       (PENDING => ACTIVE),
                       (ACTIVE => INACTIVE),
                       (ACTIVE => ACTIVE_PENDING),
                       (ACTIVE_PENDING => PENDING),
                       (ACTIVE_PENDING => ACTIVE),
                       (PENDING => INACTIVE);
  }
  
  // Cross with trigger type
  cross transition, trigger_type; // {edge, level}
  
  // Cross with EOImode  
  cross transition, eoi_mode; // {0, 1}
  
  // Cross with concurrent events
  cross transition, concurrent_event; // {none, new_edge, level_change, disable, priority_change}
endgroup
```

**Assertions to Write:**

```systemverilog
// Active count must never exceed 1 per INTID per CPU
assert_active_limit: assert property (
  @(posedge clk) active_count[intid][cpu] <= 1
);

// EOI must only affect the intended INTID
assert_eoi_no_side_effect: assert property (
  @(posedge clk) (eoi_write && eoi_intid == X) |=> 
    $stable(state[Y]) // for all Y != X
);

// Spurious (1023) must not change any state
assert_spurious_no_state_change: assert property (
  @(posedge clk) (iar_read && iar_return == 1023) |=>
    $stable(all_intid_states)
);
```

**Common Mistakes in Verification:**
- Only covering "sunny day" transitions (Inactive→Pending→Active→Inactive)
- Not covering disable/enable during Active state
- Missing multi-core races for SPIs
- Not verifying what happens with architecturally UNPREDICTABLE inputs

**What Interviewer Expects:**
- Systematic coverage model, not ad-hoc tests
- Understanding of which transitions are "dangerous" (race-prone)
- Concrete assertions for protocol compliance
- Knowledge of UNPREDICTABLE cases and how to verify implementation-specific behavior
- Appreciation for multi-core timing scenarios

---

## Quick Reference: Key Register Map

| Function | GICv2 | GICv3 |
|----------|-------|-------|
| Acknowledge | GICC_IAR | ICC_IAR0/1_EL1 |
| EOI | GICC_EOIR | ICC_EOIR0/1_EL1 |
| Deactivate | GICC_DIR | ICC_DIR_EL1 |
| Priority Mask | GICC_PMR | ICC_PMR_EL1 |
| Binary Point | GICC_BPR | ICC_BPR0/1_EL1 |
| Running Priority | GICC_RPR | ICC_RPR_EL1 |
| EOImode control | GICC_CTLR.EOImode | ICC_CTLR_EL1.EOImode |
| Enable group | GICD_CTLR | ICC_IGRPEN0/1_EL1 |

---

## Summary of Key Principles

1. **Edge = latched, Level = sampled** — fundamental to understanding lost interrupts
2. **BPR splits priority into group (preempts) and subpriority (orders only)**
3. **EOImode=1 exists for virtualization** — keeps physical interrupt Active until guest is done
4. **SError ≠ GIC** — completely separate path, no INTID, no IAR
5. **Group 0 → FIQ → EL3** (GICv3), Non-secure world cannot touch it
6. **Equal priority → implementation-defined** — never assume INTID order in portable code
7. **ICC_IAR returns highest priority at read time** — not at signal time (late arrival)
