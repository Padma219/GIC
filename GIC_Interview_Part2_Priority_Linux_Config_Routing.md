# GIC Interview Questions — Part 2
## Priority Ownership, Linux Behavior, Interrupt Configuration & Routing

---

## Topic 5: Interrupt Prioritization Ownership

---

### Q1: Your SoC has UART (INTID 64) and DMA (INTID 72). The system architect wants UART to always preempt DMA. Who programs this? Is it hardcoded in silicon? Walk through the entire chain of responsibility.

**Simple Intuition:**
The GIC hardware provides a *mechanism* (priority registers) but makes NO policy decisions. It's like a highway with speed limit signs — the signs exist (hardware), but who writes the number on them is a policy/software decision.

**Detailed Answer:**

**Responsibility Chain:**

```
Layer 0: ARM Architecture
  └─ Defines: GICD_IPRIORITYR[n] exists, 8-bit field, lower = higher priority
  └─ Does NOT define: what value any INTID should get

Layer 1: SoC Design (Hardware Integrator)
  └─ Defines: How many priority bits are implemented (min 4 in GICv3, min 4 in GICv2)
  └─ Defines: Reset value of priority registers (typically all 0x00 or all 0xF0, implementation-defined)
  └─ Does NOT hardcode: "UART > DMA" — this is NOT in silicon

Layer 2: Firmware / Bootloader (Trusted Firmware, UEFI)
  └─ May program: Group 0 (secure) interrupt priorities
  └─ May program: Initial priority scheme before OS boots
  └─ Example: ARM Trusted Firmware sets secure timer priority to 0x20

Layer 3: OS / Kernel (Linux, RTOS, bare-metal)
  └─ Programs: GICD_IPRIORITYR for all assigned interrupts
  └─ Decides: "UART gets 0x20, DMA gets 0x80"
  └─ THIS is where the policy lives for non-secure interrupts

Layer 4: User-space (indirect)
  └─ Can influence: via irqbalance, /proc/irq/N/smp_affinity
  └─ Cannot change: hardware priority (no user-space interface to GICD_IPRIORITYR)
```

**Concrete Example:**

```
SoC Spec says: "UART is critical for console debug, must be highest priority"
This is a SYSTEM DESIGN REQUIREMENT, not hardware enforcement.

Implementation:
1. Device Tree: interrupts = <GIC_SPI 64 IRQ_TYPE_LEVEL_HIGH>;
   → DT does NOT specify priority (only type + ID)

2. GIC driver at boot: sets ALL SPIs to same priority (0xA0 in Linux)
   → System requirement NOT automatically enforced!

3. To actually enforce UART > DMA:
   Option A: Custom GIC driver patch (rare, fragile)
   Option B: Firmware sets it before Linux boots (common for secure interrupts)
   Option C: Platform-specific driver calls (non-standard)
```

**How many bits are actually implemented?**

```
Architecture allows 4-8 bits of priority:
- If 4 bits implemented: priorities are 0x00, 0x10, 0x20, ... 0xF0 (16 levels)
  (Lower 4 bits read-as-zero, write-ignored)
- If 8 bits implemented: priorities are 0x00-0xFF (256 levels)
- Typical SoC: 5 bits (32 levels) or 8 bits (256 levels)

Detection: Write 0xFF to GICD_IPRIORITYR, read back
  → Implemented bits return 1, others return 0
  → If read returns 0xF8 → 5 bits implemented (bits[7:3])
```

**Why lower value = higher priority?**

Historical ARM convention. Think of it as "priority 0 = most urgent, priority 255 = least urgent." This matches urgency level semantics in most RTOS designs.

**Common Mistakes:**
- Saying "priority is hardcoded in the GIC" (it's software-programmed)
- Thinking Device Tree specifies priority (it doesn't — only interrupt type and number)
- Not knowing how to detect implemented priority bits
- Assuming Linux enforces priority ordering (it typically doesn't — flat priorities)

**What Interviewer Expects:**
- Clear articulation of the hardware/firmware/OS/architecture layering
- Knowledge that GIC provides mechanism, not policy
- Practical understanding that Linux ignores hardware priority for most systems
- Ability to detect priority bit width via read-back

---

### Q2: GICD_IPRIORITYR shows INTID 50 = 0x40 and INTID 51 = 0x40. Both go pending to the same core. INTID 50's device is more time-critical. What's wrong and how do you fix it without changing application software?

**Simple Intuition:**
Equal priorities mean the GIC uses implementation-defined tie-breaking (usually lower INTID wins). If INTID 51 is more time-critical but has a higher INTID number, it'll consistently lose. The fix must be at the GIC priority programming level.

**Detailed Answer:**

**What happens with equal priority:**

```
Both pending, same core, same priority (0x40):

GIC tie-breaking (ARM GIC-600 example): Lower INTID wins
→ INTID 50 ALWAYS acknowledged first
→ INTID 51 waits until INTID 50 is handled

If INTID 50 ISR takes 100μs → INTID 51 has guaranteed 100μs+ latency
```

**Fix options (without changing application software):**

```
Option 1: Change priority values (best)
  → GICD_IPRIORITYR[50] = 0x80 (lower priority)
  → GICD_IPRIORITYR[51] = 0x40 (higher priority, lower value)
  → Where: firmware, kernel platform code, or GIC driver init

Option 2: Route to different cores
  → GICD_IROUTER[50] → Core0
  → GICD_IROUTER[51] → Core1
  → Both handled in parallel, no priority conflict
  → Trade-off: burns a core's time, but eliminates latency dependency

Option 3: Adjust BPR to enable preemption
  → If priorities differ (after Option 1), set BPR so group priorities differ
  → Allows INTID 51 to preempt INTID 50's ISR
  → Trade-off: nested interrupts add stack complexity
```

**Why "without changing application software" matters:**
The interviewer is testing whether you know the fix lives at the platform/firmware/driver level — not in application code. Application code shouldn't know or care about GIC priority registers.

**Common Mistakes:**
- Suggesting "change the interrupt handler to be faster" (doesn't address the priority problem)
- Not knowing that equal priority tie-breaking exists
- Assuming round-robin behavior (most GICs don't do round-robin)
- Suggesting user-space solutions for a hardware scheduling problem

**What Interviewer Expects:**
- Diagnosis: equal priority + implementation-defined tie-breaking = starvation risk
- Multiple fix strategies with trade-offs
- Understanding that priority programming is a platform concern, not application concern

---

## Topic 6: Linux Behavior

---

### Q3: Walk through what the Linux GIC driver does from boot to first interrupt delivery. Does Linux use GIC hardware priorities? Why or why not?

**Simple Intuition:**
Linux treats the GIC as a flat priority, dumb interrupt delivery mechanism. The kernel's own scheduler and softirq system handles "priority" in software. Hardware priority is mostly ignored — and this is intentional.

**Detailed Answer:**

**Boot Sequence (GICv3, `drivers/irqchip/irq-gic-v3.c`):**

```
1. Device Tree parsing:
   compatible = "arm,gic-v3";
   → gic_of_init() called
   
2. Distributor initialization (gic_dist_init):
   - Disable distributor (GICD_CTLR = 0)
   - Set all SPIs to:
     • Priority = 0xA0 (FLAT — same for ALL interrupts)
     • Group 1 Non-secure
     • Level-triggered (default, overridden per DT)
     • Target = no core (routing set later)
   - Enable distributor (GICD_CTLR.ARE_NS=1, EnableGrp1NS=1)

3. Redistributor initialization (per-CPU, gic_cpu_init):
   - Wake redistributor (GICR_WAKER.ProcessorSleep = 0)
   - Wait for ChildrenAsleep = 0
   - Set SGI/PPI priorities to 0xA0 (FLAT)
   - Enable PPIs/SGIs as needed
   
4. CPU interface initialization (per-CPU):
   - ICC_PMR_EL1 = 0xF0 (allow priorities 0x00-0xE0)
   - ICC_BPR1_EL1 = 0 (no sub-priority, all bits are group priority)
   - ICC_CTLR_EL1.EOImode = 0 (combined priority drop + deactivate)
   - ICC_IGRPEN1_EL1 = 1 (enable Group 1 interrupts)
   - PSTATE.I = 0 (unmask IRQ via local_irq_enable)

5. Per-interrupt setup (when driver calls request_irq):
   - irq_set_affinity() → writes GICD_IROUTER
   - irq_set_type() → writes GICD_ICFGR (edge/level from DT)
   - GICD_ISENABLER → enable the specific INTID
   - Priority remains 0xA0 (NOT changed per driver)
```

**Does Linux use hardware priorities?**

**NO (with caveats).**

```
Why not:
1. Linux INTENTIONALLY sets all interrupts to 0xA0
2. ICC_PMR is used as a binary on/off switch (0xF0 = all pass, 0x00 = all masked)
3. No kernel API exposes "set hardware priority" to drivers
4. Scheduling priority is handled by:
   - Threaded IRQ: kernel thread priority (SCHED_FIFO, nice value)
   - Top-half (hardirq): runs to completion, no preemption by other interrupts
   - Softirq/tasklet: processed in order queued

Exception: Some RT (PREEMPT_RT) patches and specialized embedded kernels DO use
hardware priority for interrupt nesting, but mainline Linux does not.
```

**Why flat priorities?**

```
1. Simplicity: Nested interrupts add stack overflow risk and complexity
2. Determinism: Software scheduling is more predictable than hardware arbitration
3. Portability: Not all platforms implement same number of priority bits
4. History: Linux interrupt model predates GIC — designed for simpler controllers
```

**What actually determines handling order in Linux?**

```
Simultaneous pending:
→ GIC delivers one first (implementation-defined tie-breaking, usually lower INTID)
→ Second is delivered after first's top-half completes

Non-simultaneous:
→ First-come-first-served — whichever goes Pending first

After top-half:
→ Threaded IRQs compete via kernel scheduler priorities
→ Softirqs run in registration order
```

**Key insight for interviews:**
"GIC hardware priority" and "Linux interrupt priority" are **completely different concepts.** Hardware priority only matters for simultaneous arrival tie-breaking (and even then, Linux doesn't differentiate since all are 0xA0).

**Common Mistakes:**
- Assuming Linux programs different priorities for different interrupts
- Thinking request_irq() priority parameter exists (it doesn't)
- Confusing kernel thread priority (threaded IRQ) with GIC hardware priority
- Believing higher-INTID interrupts can preempt lower-INTID in Linux (they can't — flat priority, no nesting)

**What Interviewer Expects:**
- Honest answer: "Linux mostly ignores hardware priority"
- Knowledge of actual boot sequence (at least the key registers)
- Understanding of why (complexity, portability, design philosophy)
- Distinction between hardware priority (GIC) and software priority (scheduler)

---

### Q4: A developer reports: "My high-priority UART interrupt is being delayed by 2ms because a USB interrupt handler runs first." On a Linux system with default GIC configuration, explain why this happens and list all options to fix it.

**Simple Intuition:**
With flat GIC priorities, "first pending wins." If USB handler is a slow top-half, it blocks UART delivery on that core. This isn't a GIC bug — it's Linux's non-preemptive hardirq model.

**Detailed Answer:**

**Root Cause Analysis:**

```
1. GIC state: Both UART and USB are priority 0xA0 (flat)
2. USB fires first → Core0 reads ICC_IAR → USB goes Active
3. Core0 enters USB hardirq handler (PSTATE.I=1 on some paths, or
   interrupts re-enabled depending on kernel config)
4. USB handler takes 2ms (doing too much work in hardirq context)
5. UART fires during USB handling:
   - Case A (IRQ masked): UART goes Pending, waits for USB handler to finish
   - Case B (IRQ unmasked): Same priority → no preemption, still waits
6. After USB handler returns → UART acknowledged → 2ms late
```

**Why preemption doesn't help here:**
Even if PSTATE.I is cleared during USB handler:
- GIC sees both at same priority (0xA0)
- Same group priority → NO preemption (requires strictly lower value)
- UART stays Pending until USB handler finishes and does EOI

**Fix Options (ordered by invasiveness):**

```
Fix 1: Move USB work to threaded IRQ (BEST practice)
  → request_threaded_irq(usb_irq, quick_top_half, usb_thread_fn, ...)
  → Top-half: acknowledge device, return IRQ_WAKE_THREAD (fast, <10μs)
  → Thread function: do the 2ms of actual work (preemptible by UART hardirq)
  → Result: UART hardirq can interrupt USB thread

Fix 2: Set different GIC priorities (rare in Linux, but possible)
  → Patch GIC driver or platform code:
     writel(0x20, gicd_base + GICD_IPRIORITYR + uart_intid);  // high
     writel(0xC0, gicd_base + GICD_IPRIORITYR + usb_intid);   // low
  → Set ICC_BPR so group priorities differ
  → Enable IRQ nesting in kernel (non-trivial, risk of stack overflow)
  → Trade-off: breaks kernel assumptions, not upstreamable

Fix 3: Route to different cores
  → echo 2 > /proc/irq/<uart_irq>/smp_affinity
  → echo 4 > /proc/irq/<usb_irq>/smp_affinity
  → UART on Core1, USB on Core2 → no contention
  → Trade-off: affinity pinning limits load balancing

Fix 4: Use PREEMPT_RT kernel
  → Most handlers become threads automatically
  → Real priority-based scheduling of interrupt threads
  → Trade-off: throughput penalty, different kernel behavior

Fix 5: Fix the USB driver (root cause fix)
  → 2ms in hardirq is too long — driver is doing work it shouldn't
  → Move bulk processing to bottom-half (tasklet, workqueue)
  → This is the CORRECT fix per kernel coding standards
```

**Device Tree snippet for threaded IRQ:**
```dts
/* DT doesn't control threading — that's a driver choice */
uart0: serial@f000000 {
    interrupts = <GIC_SPI 64 IRQ_TYPE_LEVEL_HIGH>;
    /* Priority not specified in DT — all handled by GIC driver as 0xA0 */
};
```

**Driver code (fix 1):**
```c
// Before (slow hardirq):
request_irq(usb_irq, usb_heavy_handler, IRQF_SHARED, "usb", dev);

// After (threaded):
request_threaded_irq(usb_irq, usb_quick_ack, usb_thread_fn, 
                     IRQF_ONESHOT, "usb", dev);
```

**Common Mistakes:**
- Blaming GIC for "not prioritizing correctly" (it IS following the rules — flat priority)
- Suggesting changing interrupt priority in Device Tree (DT doesn't have priority field)
- Not knowing about threaded IRQs as the mainline solution
- Suggesting IRQ nesting without understanding stack overflow risk

**What Interviewer Expects:**
- Correct root cause: flat priority + non-preemptive hardirq model
- Multiple solutions with trade-offs
- Recommendation of threaded IRQ or driver fix as best practices
- Understanding that this is a SOFTWARE design problem, not a GIC problem

---

## Topic 7: Interrupt Configuration (Edge vs Level)

---

### Q5: A device uses level-triggered signaling, but the Device Tree specifies edge-triggered. The system boots fine but after 3-4 hours under load, the device stops receiving interrupts. Explain the exact failure mode.

**Simple Intuition:**
A level device holds its line HIGH until software clears it. If GIC thinks it's edge-triggered, it only records the initial rising edge. If that edge arrives at a bad time (like during handler execution), the GIC won't see it again because it's waiting for a NEW edge — but the device is just holding the line high, not generating new edges.

**Detailed Answer:**

**The Setup:**

```
Device Tree (WRONG):
  interrupts = <GIC_SPI 100 IRQ_TYPE_EDGE_RISING>;  // Should be LEVEL_HIGH

Actual hardware: Device asserts line HIGH until status register cleared

GICD_ICFGR configured: Edge-triggered for INTID 100
```

**Why it works initially:**

```
Normal case:
T0: Device raises line LOW→HIGH (this IS a rising edge)
T1: GIC detects edge → Pending
T2: CPU reads IAR → Active
T3: ISR clears device status → line goes LOW
T4: EOI → Inactive
T5: Next event: line LOW→HIGH again → new edge → works fine

For hours, the pattern is clean: each event is a distinct LOW→HIGH transition.
GIC sees valid edges. System works.
```

**Why it fails under load (the subtle race):**

```
Under load, events arrive faster. Eventually:

T0: Device asserts line (LOW→HIGH) → GIC: edge detected → Pending
T1: CPU reads IAR → Active
T2: DURING ISR execution, device fires AGAIN (new event from hardware)
    Device line: still HIGH (never went low — device re-asserted before ISR cleared it)
    OR: Device line briefly went LOW then HIGH again, but GIC missed the edge
    
    Case A: Line never went low
    → No new LOW→HIGH transition → GIC doesn't see an edge
    → GIC state: Active (no Active+Pending, because no edge detected)
    
T3: ISR clears device status → line goes LOW
T4: EOI → Inactive
    
    THE SECOND EVENT IS LOST.
    Device already consumed/cleared the event internally (or merged it)
    No new edge will come unless device has a new event.
    
T5: If device has a pending status bit, line stays LOW (already cleared)
    → No new edge → interrupt NEVER fires again for that device
    → Device appears "hung"
```

**Why this is intermittent (takes 3-4 hours):**

The race window is tiny — event must arrive while:
1. Previous interrupt is Active (between IAR and device clear)
2. AND the line doesn't produce a clean new LOW→HIGH edge

Under low load: events are spaced far apart → line always goes LOW between events → clean edges.
Under high load: events overlap → line stays HIGH → edge lost.

**The correct configuration:**

```dts
/* Correct Device Tree */
device@addr {
    interrupts = <GIC_SPI 100 IRQ_TYPE_LEVEL_HIGH>;
};
```

With level-triggered:
- GIC continuously samples the line
- Active state + line HIGH → Active+Pending
- After EOI → Pending (because line still high) → re-delivered
- **NEVER loses events** as long as device holds line

**Debug checklist to identify this bug:**

```
1. cat /proc/interrupts → interrupt count stopped incrementing
2. devmem2 GICD_ISPENDR → bit NOT set (no pending state)
3. devmem2 GICD_ISACTIVER → bit NOT set (not stuck active)
4. devmem2 device_status_reg → device has pending work!
5. devmem2 GICD_ICFGR → check configuration bits
   → Bits show edge-triggered, but SoC TRM says device is level
   → ROOT CAUSE FOUND
6. Check Device Tree vs hardware documentation
```

**Recovery without reboot:**
```bash
# Force-set pending to recover (temporary workaround)
devmem2 GICD_ISPENDR_offset 0x00000010  # Set pending for INTID 100
# Then fix DT and rebuild
```

**Common Mistakes:**
- Saying "edge vs level doesn't matter if the driver is fast enough" (the race always exists under load)
- Not understanding that level-triggered RESAMPLES while edge-triggered LATCHES
- Thinking GICD_ICFGR alone decides behavior (DT tells the driver, driver programs ICFGR)
- Missing that this is a TIME-DEPENDENT bug (works for hours before failing)

**What Interviewer Expects:**
- Clear explanation of WHY it works initially and WHY it fails under load
- Understanding of the timing race window
- Systematic debug approach (devmem2 to check GIC registers vs device state)
- Knowledge that the authority on edge/level is the HARDWARE DESIGN, not software choice

---

### Q6: Conversely: an edge-triggered device is configured as level-triggered in GICD_ICFGR. What symptom do you observe? How is this different from Q5?

**Simple Intuition:**
Edge device produces short pulses. If GIC is configured as level, it will sample the line and may find it LOW (pulse already gone). Result: immediate interrupt storm OR missed interrupts depending on timing.

**Detailed Answer:**

**The Setup:**

```
Device: Produces 10ns pulse (edge, rising) per event
GICD_ICFGR: Configured as level-triggered (WRONG)
```

**Scenario A: Interrupt Storm**

```
T0: Device pulses HIGH for 10ns then goes LOW
T1: GIC samples line during the 10ns pulse → sees HIGH → Pending
T2: CPU reads IAR → Active
T3: ISR runs. Tries to "clear" the device source.
    But edge devices often don't have a "clear source" mechanism
    (the pulse is already gone — there's nothing to clear)
T4: ISR finishes, writes EOI → Active → ???

    GIC re-samples line:
    - If line is LOW: → Inactive. OK, works once.
    - If there's ANY electrical noise or ringing: → sees HIGH → Pending again
    
    BUT THE REAL PROBLEM:
    Some GIC implementations sample at EOI time.
    If line happens to be HIGH (glitch, bounce, or new event) → Pending → STORM
```

**Scenario B: Missed Interrupts (more common)**

```
T0: Device pulses HIGH for 10ns
T1: GIC samples line at its sample point
    - If sample point misses the 10ns window → line is LOW → no Pending
    → INTERRUPT COMPLETELY MISSED

This is more likely because:
- GIC sampling rate is implementation-defined
- 10ns pulse may fall between sample points
- Level-triggered expects the line to STAY asserted until software clears it
```

**Comparison with Q5:**

| Aspect | Level device + Edge config (Q5) | Edge device + Level config (Q6) |
|--------|-------------------------------|-------------------------------|
| Failure mode | Lost interrupt (under load) | Missed interrupt OR storm |
| Timing | Fails after hours (race) | May fail from first event |
| Recovery | Force ISPENDR, fix DT | Fix DT, check line integrity |
| Severity | Subtle, intermittent | Often caught early (storm is obvious) |
| Root cause | GIC ignores re-assertion | GIC sampling misses short pulse |

**How to detect:**

```
Storm scenario:
- /proc/interrupts shows millions of counts in seconds
- System becomes unresponsive (CPU stuck in interrupt handlers)
- dmesg: "irq XX: nobody cared" (Linux disables the IRQ after too many unhandled)

Missed scenario:
- /proc/interrupts count doesn't increment
- Device has pending work but no interrupt recorded
- GICD_ISPENDR shows no pending for that INTID
```

**Who decides edge vs level?**

```
Authority chain:
1. HARDWARE DESIGN (ultimate truth) — the device's electrical interface
2. SoC Integration spec — documents which interrupt is which type
3. Device Tree — communicates this to the OS
4. GIC driver — programs GICD_ICFGR based on DT

The Device Tree does NOT "choose" — it DESCRIBES what the hardware IS.
A wrong DT entry is a BUG, not a configuration choice.
```

**Common Mistakes:**
- Thinking software can freely choose edge or level (it MUST match hardware)
- Not understanding that level requires persistent assertion
- Missing that interrupt storms are the obvious symptom (often caught in testing)
- Believing the GIC will "adapt" to pulse width (it won't)

**What Interviewer Expects:**
- Clear distinction between the two misconfiguration directions
- Understanding that hardware electrical behavior is the ground truth
- Practical debug approach (check /proc/interrupts, check GICD_ICFGR vs TRM)
- Knowledge that DT describes, not decides

---

## Topic 8: Interrupt Types and Routing

---

### Q7: Explain the full path of a PCIe MSI interrupt from device to CPU in a GICv3 system with ITS. What happens at each stage?

**Simple Intuition:**
PCIe MSI (Message Signaled Interrupt) is fundamentally different from wire-based interrupts. Instead of asserting a physical line, the PCIe device performs a memory WRITE to a specific address. The ITS (Interrupt Translation Service) catches this write and translates it into a GIC LPI (Locality-specific Peripheral Interrupt) targeted at a specific core.

**Detailed Answer:**

**Full Path:**

```
┌──────────┐    PCIe write    ┌─────────────┐   LPI    ┌───────────────┐
│ PCIe Dev │───────────────→ │     ITS      │────────→│ Redistributor │
│          │  (to GITS_      │ (Translation │         │   (per-CPU)    │
│          │   TRANSLATER)   │   Service)   │         │               │
└──────────┘                 └─────────────┘         └───────┬───────┘
                                                              │
                                                              ▼
                                                      ┌──────────────┐
                                                      │ CPU Interface │
                                                      │ (ICC_IAR)    │
                                                      └──────────────┘
```

**Step-by-step:**

```
Step 1: PCIe device wants to interrupt
  → Device writes a specific data value (EventID) to GITS_TRANSLATER register
  → This is a normal memory-mapped write on the PCIe bus
  → Address: ITS base + GITS_TRANSLATER offset
  → Data: EventID (identifies which interrupt from this device)

Step 2: ITS receives the write
  → ITS uses DeviceID (from PCIe BDF — Bus/Device/Function) + EventID
  → Looks up in Device Table: DeviceID → Interrupt Translation Table (ITT)
  → Looks up in ITT: EventID → INTID (LPI number) + Collection
  → Looks up Collection Table: Collection → Target Redistributor (MPIDR)

  Translation: (DeviceID, EventID) → (INTID, Target CPU)

Step 3: ITS forwards to target Redistributor
  → ITS sends the LPI INTID to the target Redistributor
  → Redistributor checks:
     • Is this LPI enabled? (GICR_ISENABLER or config table)
     • What's the priority? (LPI priority table in memory, pointed by GICR_PROPBASER)
     • Is it pending? Updates pending table (memory, pointed by GICR_PENDBASER)
  → Sets LPI as Pending

Step 4: Redistributor signals CPU Interface
  → Highest pending priority evaluated
  → If LPI priority < ICC_PMR: signal IRQ to CPU
  → CPU takes IRQ exception

Step 5: Software reads ICC_IAR1_EL1
  → Returns LPI INTID (8192+ range)
  → LPI goes Active (or pending cleared, implementation varies)
  → Software handles interrupt

Step 6: EOI
  → ICC_EOIR1_EL1 written
  → LPI deactivated
```

**Key differences from SPI:**

| Aspect | SPI (wire) | LPI (MSI via ITS) |
|--------|-----------|-------------------|
| Trigger | Physical wire | Memory write |
| Config storage | GICD registers | Memory tables (GICR_PROPBASER/PENDBASER) |
| Number range | 32-1019 | 8192+ (potentially millions) |
| Routing | GICD_IROUTER | ITS Collection Table |
| State machine | Full 4-state | Simplified (Pending/not-pending, no Active in some impls) |
| Enable | GICD_ISENABLER | LPI config table in memory |
| Priority | GICD_IPRIORITYR | LPI priority table in memory |

**ITS Commands (programmed by software):**

```
MAPD: Map DeviceID → ITT base address (device table entry)
MAPI: Map EventID → INTID + collection (ITT entry)
MAPC: Map Collection → Target PE (MPIDR)
INV:  Invalidate cached translation
SYNC: Ensure all prior commands complete
```

**Why ITS exists:**
- SPIs max at ~1020 — not enough for hundreds of PCIe devices with multiple MSI vectors
- LPIs scale to millions (INTID space: 8192 to 2^32)
- ITS provides indirection: can remap interrupts between CPUs without reprogramming devices

**Common Mistakes:**
- Thinking PCIe MSI uses a physical wire (it's a memory write)
- Not knowing that LPI config is in MEMORY, not GIC registers
- Confusing DeviceID (PCIe BDF) with EventID (per-device interrupt index) with INTID (GIC-wide)
- Forgetting the three-level translation: Device Table → ITT → Collection

**What Interviewer Expects:**
- Complete end-to-end path from PCIe device to CPU
- Understanding of the ITS translation mechanism
- Knowledge that LPIs are fundamentally different from SPIs (memory-backed, scalable)
- Awareness of ITS commands for setup/reconfiguration

---

### Q8: Core0 sends an SGI (interrupt 3) to Core1 and Core2 simultaneously. Walk through what happens. How does GICv3 differ from GICv2 in SGI handling?

**Simple Intuition:**
SGI (Software Generated Interrupt) is inter-core communication — one CPU poking another. Think IPI (Inter-Processor Interrupt). GICv2 and GICv3 handle this very differently internally, though the visible behavior is similar.

**Detailed Answer:**

**SGI Generation (GICv3):**

```
Core0 executes:
  MSR ICC_SGI1_EL1, X0

Where X0 encodes:
  - INTID = 3 (SGI number, 0-15)
  - Target list = Core1 + Core2 (affinity-based routing)
  - IRM = 0 (specific targets, not all-except-self)
```

**GICv3 Flow:**

```
T0: Core0 writes ICC_SGI1_EL1
    → GIC generates SGI 3 as EDGE-TRIGGERED event
    → Sent to Core1's Redistributor AND Core2's Redistributor

T1: Core1 Redistributor:
    → SGI 3 goes Pending (GICR_ISPENDR0, bit 3)
    → Priority checked against ICC_PMR
    → If passes: signal IRQ to Core1

T2: Core2 Redistributor:
    → Same as T1, independently
    → SGI 3 goes Pending for Core2

T3: Core1 reads ICC_IAR1_EL1:
    → Returns INTID 3
    → SGI 3 on Core1: Pending → Active

T4: Core2 reads ICC_IAR1_EL1:
    → Returns INTID 3
    → SGI 3 on Core2: Pending → Active
    (Both cores handle independently)

T5: Each core writes ICC_EOIR1_EL1 = 3
    → SGI 3 → Inactive on each core
```

**GICv2 vs GICv3 — Critical Differences:**

```
GICv2 SGI:
┌─────────────────────────────────────────────────┐
│ • Generated by writing GICD_SGIR               │
│ • Source CPU ID encoded in IAR return value!    │
│   GICC_IAR returns: [12:10]=Source CPU, [9:0]=INTID │
│ • Pending state is PER-SOURCE:                  │
│   - SGI 3 from Core0 is DIFFERENT pending bit   │
│     than SGI 3 from Core2                       │
│ • Must EOI with source CPU in GICC_EOIR        │
│ • GICD_CPENDSGIR / GICD_SPENDSGIR track per-source │
└─────────────────────────────────────────────────┘

GICv3 SGI:
┌─────────────────────────────────────────────────┐
│ • Generated by writing ICC_SGI1_EL1 (sys reg)  │
│ • Source CPU ID NOT in IAR return value         │
│   ICC_IAR returns: just INTID (0-15)           │
│ • Pending state is NOT per-source:             │
│   - SGI 3 is SGI 3, regardless of who sent it  │
│   - If Core0 and Core2 both send SGI 3 to      │
│     Core1 before Core1 reads IAR → ONE event   │
│ • EOI just uses INTID (no source encoding)     │
│ • Edge-triggered: each write is a new event    │
└─────────────────────────────────────────────────┘
```

**Critical GICv3 implication:**

```
Scenario: Core0 and Core2 both send SGI 3 to Core1:

GICv2: Core1 sees TWO interrupts (different source IDs)
  → IAR read 1: returns SGI 3 from Core0
  → IAR read 2: returns SGI 3 from Core2

GICv3: Core1 may see ONE or TWO interrupts
  → If both arrive before IAR read: MERGED into single Pending
  → Core1 reads IAR once → Active. Second is LOST.
  → If one arrives, gets acknowledged, then second arrives: two events
  → TIMING DEPENDENT
```

**Why this matters for Linux IPI:**

Linux uses SGIs for:
- SGI 0: Rescheduling IPI
- SGI 1: Call function IPI
- SGI 2: IRQ work IPI
- etc.

With GICv3, the kernel doesn't rely on per-source tracking. The IPI handler is designed to be idempotent — it checks what work needs doing regardless of how many signals arrived.

**Common Mistakes:**
- Assuming GICv3 SGIs carry source information (they don't)
- Not knowing about the merge behavior in GICv3
- Thinking SGIs go through the Distributor in GICv3 (they go to Redistributor directly)
- Forgetting that SGIs are always edge-triggered in GICv3

**What Interviewer Expects:**
- Complete GICv2 vs GICv3 comparison for SGI
- Understanding of the merge/loss scenario in GICv3
- Knowledge that Linux IPI design accounts for this
- Register-level knowledge (ICC_SGI1_EL1 encoding, GICD_SGIR for v2)

---

### Q9: GICD_IROUTER for INTID 80 is set to `Interrupt_Routing_Mode = 1` (1-of-N). Cores 0-3 are online. Under what conditions might the interrupt ALWAYS go to Core0 and never to the others? Is this a bug?

**Simple Intuition:**
1-of-N routing means "deliver to ANY one eligible core — GIC picks." The architecture says which core is picked is IMPLEMENTATION DEFINED. If the implementation always picks Core0, that's architecturally LEGAL — but may be a system performance problem.

**Detailed Answer:**

**1-of-N Routing Rules:**

```
GICD_IROUTER[80]:
  Interrupt_Routing_Mode = 1 (1-of-N)
  Aff3.Aff2.Aff1.Aff0 = ignored (mode=1 means "any PE")

GIC must deliver to ONE PE that:
  1. Has the interrupt enabled (GICD_ISENABLER)
  2. Has the interrupt group enabled (ICC_IGRPEN1_EL1)
  3. Has sufficient priority mask (ICC_PMR allows it)
  4. Is awake (GICR_WAKER.ProcessorSleep = 0)
  5. Is participating in 1-of-N (implementation-defined criteria)
```

**Why Core0 always wins (implementation-defined behaviors):**

```
Reason 1: "Lowest MPIDR wins" implementation
  → Some GICs always pick lowest-numbered eligible core
  → If Core0 is always eligible → always gets it
  → Legal per architecture. Documented in GIC TRM.

Reason 2: "Last successful target" caching
  → Some implementations cache the last core that acknowledged
  → Keeps sending there until that core is busy/asleep
  → If Core0 always acknowledges quickly → sticky to Core0

Reason 3: Core affinity in interconnect
  → Physical proximity on bus → lower latency → GIC picks it
  → Implementation optimization, not architecturally visible

Reason 4: Other cores are "blocked"
  → Core1-3 have PSTATE.I=1 (IRQ masked) — e.g., in critical sections
  → Core1-3 have ICC_PMR=0x00 (all priorities masked) — e.g., during local_irq_save
  → Core1-3 have GICR_WAKER.ProcessorSleep=1 — e.g., in cpuidle
  → Only Core0 is eligible → always gets it
```

**Is this a bug?**

```
Architecturally: NO — implementation is free to pick any eligible core
Practically:    MAYBE — if it causes load imbalance

How Linux handles this:
  → irqbalance daemon monitors /proc/interrupts
  → Detects Core0 handling disproportionate interrupts
  → Changes GICD_IROUTER to specific targeting (mode=0, pin to other core)
  → Effectively overrides 1-of-N with explicit routing

When it IS a real bug:
  → If Core0 is overloaded and latency SLA missed
  → If Core0 goes to sleep and interrupt is lost (GICR_WAKER not checked)
  → If SoC TRM claims "round-robin" but implementation doesn't do it
```

**Verification angle:**

```
What to test:
1. All cores eligible → verify at least one gets it (liveness)
2. Only one core eligible → verify that core gets it
3. Target core goes to sleep → verify re-routing to another core
4. All cores blocked → verify interrupt stays Pending (not lost)
5. If TRM claims fairness → measure distribution over N events

Coverage hole commonly missed:
  → Power transition during pending: core receiving goes to WFI
    between GIC routing decision and actual delivery
```

**Common Mistakes:**
- Expecting 1-of-N to be round-robin (architecture doesn't guarantee this)
- Thinking this is always a bug (it's architecturally valid)
- Not checking if other cores are actually eligible (PMR, sleep, enable state)
- Confusing 1-of-N for SPIs with broadcast for SGIs

**What Interviewer Expects:**
- Knowledge that 1-of-N target selection is IMPLEMENTATION DEFINED
- Multiple plausible reasons for Core0 affinity
- Understanding of when this is acceptable vs. problematic
- Awareness of Linux irqbalance as the software mitigation

---

## Verification Perspective

---

### Q10: Design a verification plan for GICD_ICFGR (interrupt configuration register). What are the critical scenarios that must be covered, and what bugs would they catch?

**Simple Intuition:**
GICD_ICFGR controls edge vs level configuration. The dangerous scenarios are: changing config while interrupt is active/pending, config mismatch with hardware, and boundary conditions between edge and level state machine behavior.

**Detailed Answer:**

**Register Background:**

```
GICD_ICFGR[n]: 2 bits per interrupt
  Bit[1] = 0: Level-sensitive
  Bit[1] = 1: Edge-triggered
  Bit[0] = Reserved (RAZ/WI in GICv3)
  
  Each register covers 16 interrupts
  SPIs: GICD_ICFGR2 onwards (ICFGR0/1 for SGIs/PPIs — may be fixed)
```

**Verification Plan:**

**Category 1: Basic Functionality**

```
Test 1.1: Edge config + rising edge → Pending
  Stimulus: Set ICFGR=edge, drive LOW→HIGH
  Check:    State transitions to Pending
  Bug found: Logic not detecting rising edge correctly

Test 1.2: Level config + line HIGH → Pending  
  Stimulus: Set ICFGR=level, drive HIGH
  Check:    State transitions to Pending, STAYS pending while line HIGH
  Bug found: Pending not sustained for level-triggered

Test 1.3: Edge config + line HIGH (no edge) → No Pending
  Stimulus: Set ICFGR=edge, line already HIGH (no transition)
  Check:    State stays Inactive
  Bug found: Level leaking into edge detection logic

Test 1.4: Level config + line LOW → No Pending
  Stimulus: Set ICFGR=level, line LOW
  Check:    State stays Inactive (or Pending clears)
  Bug found: Stuck pending on de-assertion
```

**Category 2: Dynamic Reconfiguration (BUG MAGNETS)**

```
Test 2.1: Change config while interrupt is Pending
  Stimulus: Edge → Pending → change ICFGR to Level → line goes LOW
  Check:    What happens to Pending state?
  Expected: UNPREDICTABLE per architecture — verify implementation matches TRM
  Bug found: State corruption, Pending stuck

Test 2.2: Change config while interrupt is Active
  Stimulus: Level → Pending → IAR → Active → change ICFGR to Edge
  Check:    Active state preserved? EOI behavior correct?
  Expected: Implementation should not corrupt Active state
  Bug found: Active bit cleared by config change (data loss)

Test 2.3: Change config between Pending and IAR read
  Stimulus: Edge → Pending → change to Level → IAR read
  Check:    Does IAR still return the INTID? State transition correct?
  Bug found: Race in config vs. arbitration logic
```

**Category 3: Timing Races**

```
Test 3.1: Edge arrival same cycle as ICFGR write
  Stimulus: Rising edge arrives same cycle as ICFGR being written
  Check:    Deterministic outcome (either old or new config applies)
  Bug found: Metastability in config vs. detection

Test 3.2: Short pulse on edge-configured interrupt
  Stimulus: Pulse width = 1 clock cycle
  Check:    Edge is detected
  Bug found: Minimum pulse width violation

Test 3.3: Level line toggles faster than GIC sample rate
  Stimulus: Line oscillates at high frequency
  Check:    No spurious pending/clearing
  Bug found: Aliasing in level sampling logic
```

**Category 4: Interaction with Enable/Disable**

```
Test 4.1: Edge with interrupt disabled → enable after edge gone
  Stimulus: ICENABLER=0 → edge fires → ISENABLER=1 → check state
  Expected: Remains Inactive (edge lost)
  Check:    Verify no spurious pending on enable
  Bug found: Enable logic incorrectly samples line as "edge"

Test 4.2: Level with interrupt disabled → line HIGH → enable
  Stimulus: ICENABLER=0 → line HIGH → ISENABLER=1
  Expected: Transitions to Pending (line is still HIGH)
  Bug found: Enable doesn't re-evaluate level for pending
```

**Category 5: Read-back and Access**

```
Test 5.1: Write edge, read back → verify correct value
Test 5.2: Write to reserved bit → verify RAZ/WI
Test 5.3: Non-secure access to secure interrupt ICFGR → verify blocked/RAZ
Test 5.4: SGI/PPI ICFGR bits → verify some are read-only (implementation-defined)
```

**Coverage Model:**

```systemverilog
covergroup icfgr_coverage;
  // Config state
  config_type: coverpoint icfgr_bit {bins edge = {1}; bins level = {0};}
  
  // Interrupt state when config changes
  state_at_change: coverpoint int_state {
    bins inactive = {INACTIVE};
    bins pending  = {PENDING};
    bins active   = {ACTIVE};
    bins act_pend = {ACTIVE_PENDING};
  }
  
  // Cross: config change during each state
  cross config_type, state_at_change;
  
  // Stimulus type
  hw_stimulus: coverpoint stimulus {
    bins rising_edge = {EDGE_RISE};
    bins falling_edge = {EDGE_FALL};
    bins level_high = {LEVEL_H};
    bins level_low = {LEVEL_L};
    bins short_pulse = {PULSE};
  }
  
  // Cross: stimulus vs config (catch mismatches)
  cross config_type, hw_stimulus;
endgroup
```

**Assertions:**

```systemverilog
// Level-triggered: Pending must track line state (when enabled + not active)
assert_level_tracks_line: assert property (
  @(posedge clk) 
  (icfgr_is_level && enabled && !active) |-> 
    (pending == hw_line)
);

// Edge-triggered: Pending sets only on rising edge
assert_edge_on_rise: assert property (
  @(posedge clk)
  (icfgr_is_edge && enabled && !active && !pending) |->
    (pending == $rose(hw_line))
);

// Config change must not corrupt Active state
assert_active_stable_on_reconfig: assert property (
  @(posedge clk)
  (icfgr_write && active) |=> active  // Active survives config change
);
```

**Common Verification Mistakes:**
- Only testing "config before event" — missing dynamic reconfiguration
- Not covering the edge+disabled lost-interrupt scenario
- Missing multi-interrupt interactions (changing one INTID's config affecting neighbor due to shared register)
- Not verifying read-only behavior for fixed-config interrupts (SGI 0-15)

**What Interviewer Expects:**
- Systematic approach: basic → dynamic → races → interactions
- Knowledge of architecturally UNPREDICTABLE cases and how to handle them
- Concrete assertions (not just "I'd write some checks")
- Understanding that ICFGR bugs manifest as real silicon escapes (lost interrupts, storms)

---

## Summary of Key Principles (Part 2)

1. **Priority is software-programmed, not hardcoded** — architecture provides mechanism, system architect defines policy
2. **Linux uses flat priority (0xA0 for all)** — hardware priority is effectively unused in mainline
3. **Edge/level is dictated by hardware** — DT describes, software programs, but the DEVICE decides the electrical behavior
4. **Misconfiguration direction matters** — level-as-edge = lost interrupts under load; edge-as-level = storm or missed
5. **1-of-N routing is implementation-defined** — don't assume round-robin
6. **LPIs (ITS) are memory-backed** — fundamentally different from SPIs (register-based)
7. **GICv3 SGIs lose source tracking** — software must be idempotent
8. **Verification must cover dynamic reconfiguration** — config change during active state is a bug magnet
