ARM GIC Interrupt Flow — Edge vs Level Sensitive
1. Overview
The GIC (Generic Interrupt Controller) provides delivery, prioritization, and life‑cycle management for interrupts. Each interrupt source has a state machine and an edge/level configuration:

Edge‑triggered: Event-driven; transition (rising/falling) latches a PENDING state once.
Level‑sensitive: Active while the input line is asserted; must be cleared at the device before signaling EOI, or the interrupt will re‑pend immediately.
2. GIC Interrupt State Machine
markdown


```text
                 ┌──────────────────────┐
                 │      INACTIVE        │
                 └──────────┬───────────┘
                            │
          Edge/level assert │
                            ▼
                 ┌──────────────────────┐
                 │       PENDING        │
                 └──────────┬───────────┘
                            │ ICC_IAR read
                            ▼
                 ┌──────────────────────┐
                 │       ACTIVE         │
                 └──────────┬───────────┘
                            │ EOIR/DIR
                            ▼
                 ┌──────────────────────┐
                 │ ACTIVE+PENDING (only │
                 │ if edge retriggered) │
                 └──────────┬───────────┘
                            │ EOIR/DIR
                            ▼
                       INACTIVE
```
Behavior summary
Trigger Type	Transition on assert	Re‑trigger allowed	Needs device clear before EOI
Edge	INACTIVE → PENDING (once)	Yes (subsequent edges create PENDING if not ACTIVE)	No
Level	INACTIVE → PENDING (line high)	Continuous (if line remains high)	Yes ✅
3. Interrupt Flow — Edge Triggered
markdown


```text
Device event (edge)
      │
      ▼
GICD: marks INTID as PENDING
      │
      ▼
Redistributor: priority comparison
      │  (P_new < ICC_PMR && GroupEnabled)
      ▼
CPU interface asserts signal (IRQ/FIQ)
      │
      ▼
Core samples IRQ line (PSTATE.I==0)
      │
      ▼
──────────── CPU Exception Entry ────────────
• ELR_ELx ← PC
• SPSR_ELx ← PSTATE
• PSTATE.{I,F,A,D} set (mask IRQ/FIQ/SErr/Dbg)
• PC ← VBAR_ELx + offset
─────────────────────────────────────────────
      │
      ▼
Software handler executes
      │
      ▼
Read pending interrupt:
    INTID ← ICC_IAR1_EL1 (or ICC_IAR0_EL1)
      ↑
      │ Reads + update:
      │  • PENDING → ACTIVE
      │  • ICC_RPR ← INT_PRIORITY
      │  • Removes from list of signaled
      ▼
Acknowledge interrupt
      │
      ▼
Handle device / event
      │
(Edge device usually clears itself)
      ▼
Signal end to GIC:
      ICC_EOIR1_EL1 ← INTID
      │
      └─► ACTIVE → INACTIVE
                   ICC_RPR drops to next active
```
Edge‑Triggered Notes
Once acknowledged (IAR read), that edge event is consumed.
If a new edge arrives while still ACTIVE, the state becomes ACTIVE+PENDING, causing an immediate re‑pend after the first EOI.
If no new edge occurs, line stays inactive and interrupt becomes INACTIVE after EOI.
4. Interrupt Flow — Level Sensitive
markdown


```text
Device asserts level (line HIGH)
      │
      ▼
GICD: sets INTID PENDING (line sampled high)
      │
      ▼
Redistributor passes interrupt (priority comparison)
      │
      ▼
CPU interface asserts IRQ/FIQ signal
      │
      ▼
────── Exception Entry (same as edge) ───────
ELR_ELx ← PC
SPSR_ELx ← PSTATE
PSTATE.I ← 1 (mask IRQ)
PC ← VBAR_ELx + offset
─────────────────────────────────────────────
      │
      ▼
Handler executes
      │
      ▼
Read INTID = ICC_IAR1_EL1
 • Marks PENDING→ACTIVE
 • ICC_RPR updated
      │
      ▼
Service device → **must clear source** (e.g. write to device ISR to lower line)
      │
      ▼
End of interrupt:
  write ICC_EOIR1_EL1 ← INTID
If level still asserted at EOI:
  → GIC samples line HIGH
  → immediately sets ACTIVE→PENDING
  → re‑signals interrupt
  → handler re‑enters
If line LOW at EOI:
  → ACTIVE→INACTIVE (normal completion)
```
Level‑Sensitive Key Point
Device must deassert the level before EOI, otherwise the GIC will see the input still high and automatically re‑pend the interrupt, retriggering the CPU.

5. ARM Exception Entry & Exit Registers
Entry Sequence (IRQ/FIQ)
Operation	Destination	Description
Save PC	ELR_ELx	Return address of interrupted instruction
Save PSTATE	SPSR_ELx	Old PSTATE snapshot (NZCV, D/A/I/F, EL, etc.)
Update PSTATE	Hardware	Sets mask bits and changes EL to handler mode
Change SP	SP ← SP_ELx	Switch to handler stack if configured
Vector fetch	VBAR_ELx + offset	Uses exception type & origin (lower/same EL)
Exit (ERET)
Action	Description
PSTATE ← SPSR_ELx	Restores condition flags, interrupt masks, EL, SPSel
PC ← ELR_ELx	Resumes normal code execution
IRQ line sampled again	If still asserted, new entry occurs
PSTATE Mask Bits (DAIF)
Mask	Meaning	Set on entry	Cleared when
D	Debug	1 (masked)	Debug exception return
A	SError	1	Software unmasks via msr daifclr, #8
I	IRQ	1	Cleared explicitly (nested IRQs)
F	FIQ	1	Cleared explicitly for nested FIQs
6. Registers Used in the Flow
Register	Function	Typical Access Time
ICC_IAR1_EL1 / ICC_IAR0_EL1	Acknowledge highest pending interrupt (returns INTID, clears PENDING→ACTIVE)	Read
ICC_EOIR1_EL1 / ICC_EOIR0_EL1	End of interrupt, drop priority (and deactivate if EOImode=0)	Write
ICC_DIR_EL1	Explicit deactivate (EOImode=1 only)	Write
ICC_RPR_EL1	Running priority (reflects top active interrupt)	Read
ICC_PMR_EL1	Priority mask (interrupts with ≥ value are blocked)	R/W
ICC_BPR1_EL1 / ICC_BPR0_EL1	Binary point, defines preemption group vs subpriority bits	R/W
7. Complete Combined Flow (Edge + Level)
markdown


```text
┌───────────────────────────────────────────────────────────────┐
│                    COMPLETE INTERRUPT FLOW                    │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│ Device asserts                                               │
│  ├─ Edge-triggered: latch rises once                         │
│  └─ Level-triggered: line remains HIGH                       │
│                                                               │
│ Distributor marks INTID PENDING                              │
│ Redistributor performs routing + priority check               │
│   - Priority < ICC_PMR?                                       │
│   - Group enabled? ICC_IGRPENx=1                              │
│                                                               │
│ If passes, nIRQ/nFIQ asserted to CPU                          │
│                                                               │
│ CPU exception entry (hardware):                               │
│   - Save ELR_ELx, SPSR_ELx                                    │
│   - PSTATE.I,F,A,D ← 1                                        │
│   - Switch to appropriate stack (SP_ELx)                      │
│   - Branch to vector table                                    │
│                                                               │
│ Software handler (C code):                                    │
│   intid = ICC_IAR1_EL1                                        │
│   handle_interrupt(intid)                                     │
│                                                               │
│   ┌── Edge: device auto clears                                │
│   │                                                         │
│   └── Level: software clears device (lower line!)             │
│                                                               │
│   ICC_EOIR1_EL1 ← intid                                       │
│   (If EOImode=1, ICC_DIR_EL1 later)                           │
│                                                               │
│   ┌── Edge:                                                   │
│   │    no reassert, INACTIVE                                  │
│   └── Level:                                                  │
│        line high → re‑pend → retaken                          │
│                                                               │
│ CPU executes ERET → resumes                                   │
└───────────────────────────────────────────────────────────────┘
```
8. Timing Intuition & Verification Scenarios
Edge
Device line pulse → GIC samples rising edge → Pending once
If CPU is busy (I masked), remains pending until mask cleared.
No line hold needed; clearing device optional.
Level
Device line high → remains pending until both:
Acked (IAR read ⇒ ACTIVE)
Line deasserted before EOIR
Deassert late ⇒ immediate re‑pend → next IRQ soon after EOIR.
9. Verification Matrix (Expected GIC State Transitions)
Trigger	Before EOI line state	After EOI transition	Next CPU behavior
Edge, no retrigger	Low	ACTIVE→INACTIVE	No reentry
Edge, retrigger during active	Don’t care	ACTIVE+PENDING→PENDING	Immediate second IRQ
Level, cleared	Low	ACTIVE→INACTIVE	Done
Level, not cleared	High	ACTIVE→PENDING	Re‑signaled instantly
10. Example Firmware Skeleton (Edge vs Level)
c


void __irq_handler(void) {
    uint32_t intid = read_sysreg(ICC_IAR1_EL1);
    if (intid >= 1020) return; // spurious
    switch (intid) {
    case EDGE_INT:
        handle_edge_event();
        break;
    case LEVEL_INT:
        handle_level_event();
        clear_device_irq();  // ensure line low before EOIR
        break;
    default:
        default_handler();
        break;
    }
    write_sysreg(ICC_EOIR1_EL1, intid);
}
11. Exception Context Summary (AArch64 IRQ Example)
Register Saved	Source	Purpose
ELR_EL1	Hardware	Return PC after interrupt
SPSR_EL1	Hardware	Old PSTATE
SP_EL1 (or SP_ELx)	Software or HW switch	Stack pointer for handler
General Purpose Registers	Software	Saved by handler/ABI
ICC_RPR_EL1	Hardware updates	Running priority
DAIF (in PSTATE)	Hardware set to 1111	Masks exceptions
ICC_IAR1_EL1 value	Returned by HW	INTID currently active
On ERET, SPSR_EL1→PSTATE and ELR_EL1→PC.

✅ Key Takeaways

Edge interrupts auto‑clear once acknowledged; new edge → new pending.
Level interrupts must be cleared at source before EOIR, or re‑pending occurs.
ELR_ELx and SPSR_ELx always saved/restored by CPU hardware on entry/exit.
ICC_IAR/E0IR/PMR/RPR/BPR orchestrate preemption, masking, and priority flow.
The correct service sequence avoids “lost” interrupts or livelock due to uncleared levels.
