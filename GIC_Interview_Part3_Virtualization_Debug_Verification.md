# GIC Interview Questions — Part 3
## Virtualization, Debug Scenarios & Verification

---

## Topic 9: Virtualization

---

### Q1: A hypervisor receives a physical interrupt (INTID 100, SPI) and must inject it as a virtual interrupt to a guest VM. Walk through the complete flow from physical assertion to guest handling and back, with EOImode=1. Show what happens at each layer.

**Simple Intuition:**
Think of the hypervisor as a mailroom. A physical letter (interrupt) arrives at the building (GIC). The mailroom (hypervisor) receives it, logs that it's holding something (keeps physical interrupt Active), then puts a note in the guest's mailbox (List Register). The guest picks up the note (virtual IAR), handles it, says "done" (virtual EOIR) — only THEN does the mailroom release the original letter (physical DIR/deactivate).

**Detailed Answer:**

**System Configuration:**
```
- GICv3, EL2 hypervisor
- Physical: ICC_CTLR_EL2.EOImode = 1 (SPLIT: priority drop ≠ deactivate)
- Virtual:  ICH_VMCR_EL2.VEOIM = 0 (guest uses combined EOI — it doesn't know it's virtualized)
- INTID 100 = Group 1 NS, routed to this PE
```

**Complete Flow:**

```
═══════════════════════════════════════════════════════════
PHASE 1: Physical interrupt delivery to Hypervisor
═══════════════════════════════════════════════════════════

T0: Device asserts → INTID 100 goes Pending in Distributor
    GICD → Redistributor → CPU Interface signals IRQ

T1: CPU is running Guest (EL1) → IRQ trapped to EL2 (HCR_EL2.IMO=1)
    CPU: EL1 → EL2 exception entry
    - SPSR_EL2 saves guest state
    - ELR_EL2 saves guest return address
    - PSTATE.DAIF all set

T2: Hypervisor reads ICC_IAR1_EL1 (physical acknowledge)
    → Returns INTID 100
    → Physical GIC state: Pending → Active
    → Running priority = priority of INTID 100

═══════════════════════════════════════════════════════════
PHASE 2: Priority drop (EOImode=1 enables this)
═══════════════════════════════════════════════════════════

T3: Hypervisor writes ICC_EOIR1_EL1 = 100 (priority drop ONLY)
    → Physical GIC: Running priority drops back to idle
    → Physical INTID 100: STILL ACTIVE (not deactivated!)
    → Hypervisor can now receive other physical interrupts
    
    WHY THIS MATTERS:
    If another higher-priority interrupt fires, hypervisor can handle it
    Without EOImode=1, hypervisor would be stuck at INTID 100's priority
    until guest finishes (could be milliseconds)

═══════════════════════════════════════════════════════════
PHASE 3: Virtual interrupt injection
═══════════════════════════════════════════════════════════

T4: Hypervisor writes a List Register (LR):
    ICH_LR<n>_EL2 = {
      vINTID   = 100,        // Virtual INTID guest will see
      pINTID   = 100,        // Physical INTID (used with HW bit)
      HW       = 1,          // Hardware-backed: auto-deactivate physical on vEOI
      Group    = 1,          // Group 1 (IRQ to guest)
      Priority = 0xA0,       // Virtual priority presented to guest
      State    = 01 (Pending) // Inject as Pending
    }

T5: Hypervisor does ERET → returns to Guest (EL1)
    Virtual CPU Interface evaluates LRs:
    → LR with State=Pending, priority passes virtual PMR
    → Virtual IRQ signaled to guest

═══════════════════════════════════════════════════════════
PHASE 4: Guest handles virtual interrupt
═══════════════════════════════════════════════════════════

T6: Guest takes IRQ exception (within EL1)
    Guest reads ICC_IAR1_EL1 → TRAPPED to EL2? NO!
    (ICH_HCR_EL2 can allow direct virtual access without trap)
    
    Virtual CPU Interface returns: INTID 100 (from LR)
    LR State: Pending → Active
    Guest sees: normal interrupt, doesn't know it's virtual

T7: Guest runs ISR, handles the interrupt
    (May access device via MMIO — hypervisor may need to emulate or passthrough)

T8: Guest writes ICC_EOIR1_EL1 = 100 (virtual EOI)
    Virtual CPU Interface processes:
    → LR State: Active → Inactive (LR freed)
    → Since HW=1: AUTOMATICALLY writes ICC_DIR_EL1 for pINTID 100
    → Physical GIC: Active → Inactive (DEACTIVATED)
    
    This is the magic of HW bit:
    Guest EOI → physical deactivation happens without hypervisor intervention

═══════════════════════════════════════════════════════════
PHASE 5: Cleanup
═══════════════════════════════════════════════════════════

T9: LR is now free (State=Invalid/Inactive)
    Physical INTID 100 is Inactive
    System is clean, ready for next interrupt
```

**What if HW=0 (software-only virtual interrupt)?**

```
HW=0 means: virtual interrupt has NO physical counterpart
Use case: virtual timer, inter-VM notification, emulated device

Difference at T8:
→ LR State: Active → Inactive
→ NO automatic physical deactivation (no pINTID)
→ If hypervisor held a physical interrupt Active, it must MANUALLY
  write ICC_DIR_EL1 (via maintenance interrupt or trap)
```

**ASCII Summary:**

```
Physical world:          Virtual world (guest's view):
                         
Pending ──IAR──→ Active  
              │          Hypervisor writes LR
Active ──EOIR─→ Active   Pending (in LR)
(prio dropped)    │      
                  │      Guest IAR
                  │      Active (in LR)
                  │      
                  │      Guest EOIR
Active ──DIR──→ Inactive Inactive (LR freed)
   (HW=1 auto)
```

**Common Mistakes:**
- Forgetting that EOImode=1 means EOIR only drops priority (interrupt stays Active)
- Not understanding HW bit automates the physical deactivation
- Thinking guest EOIR always traps to hypervisor (with HW=1 it doesn't need to)
- Confusing physical Active state with virtual LR state

**What Interviewer Expects:**
- End-to-end flow with both physical and virtual state machines
- Clear explanation of WHY EOImode=1 is needed (priority drop enables other interrupts)
- Understanding of HW bit optimization (avoids trap on guest EOI)
- Knowledge of which registers are trapped vs. directly accessed by guest

---

### Q2: Explain the List Register (LR) structure in GICv3. The hypervisor has 4 LRs (ICH_LR0-3) but the guest has 10 interrupts pending. What happens? What is the maintenance interrupt?

**Simple Intuition:**
LRs are the "mailbox slots" — limited slots to hold virtual interrupts for the guest. If you have more interrupts than slots, the hypervisor must manage overflow. The maintenance interrupt is the "slot freed" notification that tells the hypervisor "a slot opened up, inject the next queued interrupt."

**Detailed Answer:**

**LR Structure (GICv3, ICH_LR<n>_EL2, 64-bit):**

```
Bits [63:62] - State:
  00 = Invalid (LR empty/free)
  01 = Pending
  10 = Active
  11 = Active+Pending

Bits [61]    - HW:
  0 = Software-only virtual interrupt
  1 = Hardware-backed (pINTID valid, auto-deactivate)

Bits [60]    - Group:
  0 = Group 0 (virtual FIQ)
  1 = Group 1 (virtual IRQ)

Bits [55:48] - Priority: Virtual priority for this interrupt

Bits [44:32] - pINTID: Physical INTID (only valid when HW=1)

Bits [31:0]  - vINTID: Virtual INTID presented to guest
```

**Number of LRs:**
```
- Implementation-defined: 1 to 16 LRs
- Typical: 4 (mobile SoCs), 8 or 16 (server SoCs)
- Queried via: ICH_VTR_EL2.ListRegs field
- More LRs = fewer traps to hypervisor = better performance
- Fewer LRs = smaller silicon area = cheaper
```

**Overflow Scenario: 4 LRs, 10 pending virtual interrupts:**

```
State: LR0-LR3 all occupied (State=Pending or Active)
       6 more virtual interrupts queued in hypervisor's software list

Problem: Can't inject interrupt #5-#10 — no free LRs

Solution: Maintenance Interrupt
```

**Maintenance Interrupt Flow:**

```
Step 1: Hypervisor configures ICH_HCR_EL2:
  - UIE = 1 (Underflow Interrupt Enable)
    → Fires when fewer than 2 LRs have State != Invalid
  OR
  - LRENPIE = 1 (List Register Entry Not Present IE)
    → Fires when guest acknowledges and no pending LR available
  OR
  - NpIE = 1 (No Pending Interrupt Enable)
    → Fires when no LR has Pending state

Step 2: Guest processes interrupts, writes virtual EOIR:
  → LR State: Active → Invalid (freed)
  → When enough LRs freed → Maintenance interrupt condition met

Step 3: Maintenance interrupt fires (physical interrupt to hypervisor):
  → Hypervisor enters EL2
  → Checks software queue: "I have 6 more to inject"
  → Fills freed LRs with next pending virtual interrupts
  → Returns to guest (ERET)
  → Guest sees new virtual interrupts

Step 4: Repeat until software queue is empty
```

**Detailed timeline:**

```
T0: Hypervisor has 10 interrupts for guest
    Fills LR0-LR3 (highest priority 4)
    Sets ICH_HCR_EL2.UIE = 1
    Queues remaining 6 in software list
    ERET to guest

T1: Guest handles vINTID in LR0, writes EOIR → LR0 = Invalid
T2: Guest handles vINTID in LR1, writes EOIR → LR1 = Invalid
T3: Guest handles vINTID in LR2, writes EOIR → LR2 = Invalid
    
    Now: only LR3 has State ≠ Invalid (1 valid < 2)
    UIE condition met → maintenance interrupt fires

T4: Hypervisor enters EL2 (maintenance IRQ)
    Reads ICH_MISR_EL2 to determine cause (underflow)
    Fills LR0-LR2 with next 3 from software queue (items 5,6,7)
    3 remaining in queue
    ERET to guest

T5: Guest continues handling... cycle repeats
```

**Performance Impact:**

```
Each maintenance interrupt = context switch (EL1→EL2→EL1)
Cost: ~1-5μs per switch (varies by implementation)

4 LRs, 10 interrupts = at least 2 maintenance interrupts
16 LRs, 10 interrupts = zero maintenance interrupts (all fit)

This is why server GICs have more LRs — virtualization-heavy workloads
```

**Common Mistakes:**
- Thinking all virtual interrupts must fit in LRs simultaneously
- Not knowing about maintenance interrupt as the overflow mechanism
- Confusing maintenance interrupt with the guest's interrupt (it's physical, to hypervisor)
- Assuming LR count = max virtual interrupts (no — it's a window, with software queue backing)

**What Interviewer Expects:**
- LR field-level knowledge (State, HW, Priority, vINTID, pINTID)
- Understanding of overflow handling via maintenance interrupt
- Performance awareness (LR count vs. trap frequency trade-off)
- Knowledge of ICH_HCR_EL2 enables for maintenance conditions

---

### Q3: What is GICv4 vPE scheduling and doorbell interrupts? Why does GICv4 exist when GICv3 already supports virtualization?

**Simple Intuition:**
GICv3 virtualization requires the hypervisor to manually inject every virtual interrupt via LRs. GICv4 lets the hardware DIRECTLY deliver virtual interrupts to a guest without hypervisor intervention — but only when the guest's vCPU is actually running. If the vCPU is not scheduled, a "doorbell" interrupt wakes up the hypervisor to schedule it.

**Detailed Answer:**

**The Problem GICv4 Solves:**

```
GICv3 virtual interrupt path:
  Device → Physical SPI → Hypervisor trap → Read IAR → Write LR → ERET → Guest

Every single device interrupt for a VM = hypervisor trap
Cost: ~2-5μs per interrupt
With 100K interrupts/sec (network-heavy VM): 200-500ms of CPU time just for injection!
```

**GICv4 Direct Injection:**

```
GICv4 path (when vCPU is running):
  Device → ITS → LPI → GIC directly injects to vCPU via LR hardware
  
  NO hypervisor trap. Zero software overhead.
  The GIC hardware knows which vCPU is currently scheduled.
```

**How it works — vPE (virtual PE) concept:**

```
vPE = Virtual Processing Element = a vCPU
Each vPE has:
  - A virtual LPI pending table (in memory, like physical LPIs)
  - A virtual LPI config table
  - A scheduling state (resident on a physical PE, or not)

ITS extended commands for GICv4:
  VMAPP:  Map vPE to a physical Redistributor (when vCPU is scheduled)
  VMOVI:  Move virtual interrupt between vPEs  
  VMAPTI: Map (DeviceID, EventID) → (vINTID, vPE)
  VINVALL: Invalidate cached virtual interrupt state
```

**vPE Scheduling Flow:**

```
═══ When hypervisor schedules vCPU onto physical Core2: ═══

1. Hypervisor writes ITS VMAPP command:
   "vPE #7 is now resident on Redistributor for Core2"
   
2. GIC hardware maps vPE's virtual pending table to Core2's Redistributor

3. Any pending virtual LPIs in vPE #7's table:
   → Directly delivered to Core2 as virtual interrupts
   → NO hypervisor involvement

4. New device interrupts targeting vPE #7:
   → ITS translates: (DeviceID, EventID) → (vINTID, vPE #7)
   → vPE #7 resident on Core2 → inject directly to Core2's virtual interface
   → Guest sees interrupt immediately

═══ When hypervisor deschedules vCPU (vPE #7 no longer resident): ═══

5. Hypervisor writes ITS VMAPP with V=0 (not resident)

6. New device interrupts targeting vPE #7:
   → ITS translates, finds vPE not resident
   → Records interrupt in vPE's virtual pending table (memory)
   → Generates DOORBELL interrupt to hypervisor!

7. Doorbell interrupt tells hypervisor:
   "vPE #7 has a pending interrupt — consider scheduling it"
   
8. Hypervisor scheduling decision:
   → Schedule vPE #7 on some core → VMAPP → pending interrupts delivered
   → Or defer (guest can wait) — doorbell just provides information
```

**Doorbell Interrupt Details:**

```
- Each vPE has an associated doorbell INTID (physical LPI)
- Doorbell fires when: vPE is not resident AND gets a new virtual interrupt
- Hypervisor uses doorbell to make scheduling decisions
- Can be enabled/disabled per vPE
- Coalesced: multiple pending virtual interrupts → single doorbell
```

**GICv4.1 Improvements over GICv4.0:**

```
GICv4.0 limitations:
- Only LPIs (MSI) can be directly injected
- SGIs and SPIs still require hypervisor LR injection

GICv4.1 additions:
- Direct injection of virtual SGIs (vSGI)
- Improved doorbell handling (selective enable)
- Better vPE scheduling performance
- Virtual SGI pending state saved/restored without hypervisor intervention
```

**Performance Impact (real numbers):**

```
Network-heavy VM, 100K interrupts/sec:

GICv3: 100K hypervisor traps/sec → ~10-20% CPU overhead for injection
GICv4: 0 traps for direct-injected LPIs → <1% overhead
        (doorbell only fires when vCPU not scheduled)

This is the difference between viable 10GbE VM networking and not.
```

**Common Mistakes:**
- Thinking GICv4 eliminates ALL hypervisor involvement (only for LPIs when vCPU is running)
- Not knowing about doorbell (thinking interrupts to non-resident vPE are lost)
- Confusing GICv4 direct injection with device passthrough (GICv4 works with shared devices too, via ITS)
- Believing GICv4 replaces LRs (LRs still used for SPIs, SGIs, and non-direct interrupts)

**What Interviewer Expects:**
- Clear articulation of the problem GICv4 solves (hypervisor trap overhead)
- Understanding of vPE concept and resident/non-resident scheduling
- Knowledge of doorbell as the "wake up the scheduler" mechanism
- Performance awareness (why this matters for data-center workloads)

---

## Topic 10: Debug Scenarios

---

### Q4: SYMPTOM: A network device works for exactly 1 packet after boot, then never receives another interrupt. The interrupt count in /proc/interrupts stays at 1. Physical network link is fine. What went wrong?

**Simple Intuition:**
"Works once then dies" = interrupt got stuck somewhere after the first handling. Classic: interrupt stuck in ACTIVE state because EOI was never written, or device source not cleared so no new edge generated.

**Detailed Answer:**

**Systematic Root Cause Analysis:**

```
Symptom: count = 1 → first interrupt delivered and handled
         count stays 1 → no subsequent delivery
         
Two possible stuck points:
A) GIC: interrupt stuck ACTIVE (EOI missing) → blocks re-delivery
B) Device: source not cleared → no new event generated (edge) or storm absorbed (level)
C) Routing: interrupt moved/disabled after first handling
```

**Debug Step 1: Check GIC state**

```bash
# Read GICD_ISACTIVER for the INTID (say INTID 80)
devmem2 0x08000300 w   # GICD_ISACTIVER2 (INTIDs 64-95)
# If bit 16 is SET (INTID 80 = offset 16 in this register): STUCK ACTIVE
```

**If ACTIVE bit is set → Root Cause A: Missed EOI**

```
What happened:
T0: Network device fires INTID 80 → Pending
T1: CPU reads ICC_IAR → returns 80 → Active
T2: ISR runs... but NEVER writes ICC_EOIR1_EL1 = 80

Why EOI was missed:
1. Driver bug: early return path in ISR skips EOI
2. ISR crashed (exception during handler, longjmp, panic)
3. Wrong IRQ handler registered (returned IRQ_NONE without EOI)
4. EOImode=1 but DIR never written (hypervisor bug)
5. ISR wrote EOIR with wrong INTID value (e.g., wrote 0 instead of 80)
```

```
Why this blocks ALL future interrupts for INTID 80:
- GIC rule: Active interrupt cannot go Pending again (edge-triggered)
  OR: Active interrupt IS Active+Pending but priority doesn't allow
  re-delivery because running priority wasn't dropped (EOImode=0 case)
- Actually, without EOI, running priority stays elevated
  → Even OTHER lower-priority interrupts may be blocked!
```

**If ACTIVE bit is NOT set → Root Cause B: Device not re-arming**

```
What happened:
T0: Network device fires edge (INTID 80 is edge-triggered)
T1: ISR runs, writes EOI ← this part works
T2: ISR clears the device interrupt status register
    BUT: device expects a "re-arm" or "enable next interrupt" command
T3: Device has a new packet → but interrupt generation is disabled at device level
    → No new edge → GIC never sees new event

Common with:
- Intel NICs (e1000e): must write ICR or IMS to re-enable interrupt
- DMA controllers: must re-enable channel interrupt after completion
- Any device with "interrupt coalescing" that needs explicit re-arm
```

**If neither → Root Cause C: Masking/routing changed**

```
Check:
- GICD_ISENABLER: still enabled?
- ICC_PMR: still allows this priority?
- GICD_IROUTER: still pointing to this core?
- GICR_WAKER: core still awake?
- irqbalance moved affinity to offline core?
```

**Recovery:**

```
For stuck ACTIVE (without reboot):
1. If EOImode=0: write GICD_ICACTIVER to force-clear Active (DANGEROUS)
   → May corrupt driver state
2. If EOImode=1: write ICC_DIR_EL1 = 80 (deactivate)
3. Then trigger a new interrupt: write GICD_ISPENDR (force-pend)

For device not re-armed:
1. Reset device interrupt logic
2. Write device-specific "re-enable interrupt" register
3. Fix driver to properly re-arm after handling
```

**Linux-specific clue:**

```
If you see in dmesg:
  "irq 80: nobody cared (try booting with the 'irqpoll' option)"
  
This means: handler returned IRQ_NONE → Linux suspects unhandled
After 100K unhandled → Linux DISABLES the interrupt
→ Different failure mode but similar symptom (count stops)
```

**Common Mistakes:**
- Jumping to "device is broken" without checking GIC state
- Not knowing how to read GICD_ISACTIVER to diagnose stuck ACTIVE
- Forgetting that Active state blocks re-delivery
- Missing the device-side "re-arm" requirement (especially for network NICs)

**What Interviewer Expects:**
- Systematic approach: GIC state → Device state → Routing state
- Knowledge of exact registers to check (ISACTIVER, ISPENDR, ISENABLER)
- Understanding of WHY Active blocks re-delivery
- Multiple hypotheses with ability to distinguish via register reads

---

### Q5: SYMPTOM: Your interrupt counter shows 500,000 interrupts per second for a device that should fire at most 1,000/sec. System is barely responsive. CPU shows 99% in interrupt context. Diagnose.

**Simple Intuition:**
Interrupt storm. The GIC keeps delivering the same interrupt immediately after EOI. Most common cause: level-triggered interrupt where the device source is never properly cleared, so the GIC sees the line HIGH after every EOI and immediately re-pends.

**Detailed Answer:**

**Immediate Triage:**

```bash
# Identify the offending interrupt
cat /proc/interrupts  # Look for rapidly incrementing counter
watch -n1 cat /proc/interrupts  # See which one is growing at 500K/sec

# Check current state
cat /proc/irq/80/spurious  # Shows count of unhandled invocations
```

**Root Cause Tree:**

```
Interrupt Storm Causes:
│
├── Level-triggered + device source not cleared
│   └── ISR acknowledges GIC but doesn't clear device status register
│       → After EOI, line still HIGH → immediate re-pending → STORM
│
├── Edge misconfiguration (actually level device)
│   └── Device holds line HIGH → GIC configured as edge
│       → BUT: GIC implementation still detects "level" on some revisions
│       → Or: initial level-high at boot keeps re-triggering
│
├── Shared interrupt line (IRQF_SHARED) 
│   └── Device A and Device B share INTID
│       → Device A's handler runs, clears A's source
│       → Device B still asserting → line stays HIGH → re-trigger
│       → Device B's handler not registered or returns IRQ_NONE
│
├── Hardware glitch / electrical noise
│   └── Floating interrupt line, no pull-down
│       → Noise causes continuous edge detection or level-high
│
└── EOImode=1 without DIR (hypervisor bug)
    └── Priority dropped but interrupt still Active
        → If implementation allows Pending while Active (Active+Pending)
        → Level re-asserts → Active+Pending after each EOIR
        → But this shouldn't storm... unless implementation bug
```

**Most Likely Cause (90% of cases): EOI without clearing device**

```
The flow:

T0: Device asserts line HIGH (level-triggered, INTID 80)
T1: GIC: Pending → IAR read → Active
T2: ISR runs:
    a) Does some work
    b) Writes ICC_EOIR1_EL1 = 80 ← EOI done
    c) But FORGOT to write device's interrupt-clear register!
    
T3: GIC: Active → checks line → LINE STILL HIGH → Pending
T4: GIC immediately signals IRQ → CPU takes exception
T5: ICC_IAR → Active → ISR → EOI → line still HIGH → Pending
    ... infinite loop at hardware speed

Time between T2 and T4: ~10-50 nanoseconds
Result: 500K+ interrupts/sec = CPU completely consumed
```

**Fix:**

```c
// BROKEN handler:
irqreturn_t broken_handler(int irq, void *dev) {
    data = read_device_data(dev);
    process(data);
    return IRQ_HANDLED;  // ← Never cleared device interrupt source!
}

// FIXED handler:
irqreturn_t fixed_handler(int irq, void *dev) {
    data = read_device_data(dev);
    process(data);
    writel(INT_CLEAR, dev->base + INT_STATUS_REG);  // ← Clear device source
    return IRQ_HANDLED;
}
```

**Debug sequence to confirm root cause:**

```bash
# Step 1: Disable the interrupt to stop the storm
echo 0 > /proc/irq/80/control  # or: devmem2 GICD_ICENABLER

# Step 2: Check device interrupt status register
devmem2 <device_int_status_addr>  # Is source bit still set? → YES = root cause

# Step 3: Check GICD_ICFGR for correct trigger type
devmem2 <GICD_ICFGR_addr>  # Edge or level? Matches device?

# Step 4: Manually clear device source
devmem2 <device_int_clear_addr> 0x1

# Step 5: Re-enable GIC interrupt
# If storm stops → confirmed: driver bug (not clearing source)
```

**Shared interrupt scenario debug:**

```bash
# If IRQF_SHARED:
cat /proc/irq/80/  # Lists all handlers registered

# Check each device's status register individually
# The one still asserting after all handlers ran = the culprit
```

**Common Mistakes:**
- Blaming the GIC ("GIC is broken") when it's doing exactly what it should
- Not checking the device-side interrupt status
- Forgetting about shared interrupts (line stays high because ONE device isn't handled)
- Trying to fix with irqpoll or edge-trigger hack instead of fixing the driver

**What Interviewer Expects:**
- Immediate identification: storm = level-trigger + source not cleared (90% case)
- Systematic elimination of other causes
- Hands-on debug commands (devmem2, /proc/interrupts)
- Clear understanding that GIC is CORRECT — the bug is in software/driver

---

### Q6: SYMPTOM: After hotplugging Core2 back online, interrupts routed to Core2 are never delivered. Other cores receive interrupts fine. GICD_IROUTER for affected INTIDs correctly points to Core2's MPIDR. What's wrong?

**Simple Intuition:**
The GIC's Redistributor for Core2 is still in "sleep" mode. When a core powers down, its Redistributor enters ProcessorSleep state. Software must wake it up before the core can receive interrupts.

**Detailed Answer:**

**Root Cause: GICR_WAKER.ProcessorSleep not cleared**

```
Power-down sequence (what happened when Core2 went offline):
1. Linux CPU hotplug calls gic_cpu_disable()
2. ICC_IGRPEN1_EL1 = 0 (disable Group 1)
3. Core2 enters WFI → power controller powers it down
4. Redistributor enters sleep: GICR_WAKER.ProcessorSleep = 1
   → GICR_WAKER.ChildrenAsleep = 1 (set by hardware)
   → Redistributor stops forwarding interrupts to this CPU interface

Power-up sequence (what SHOULD happen):
1. Power controller powers Core2 back on
2. Core2 wakes, firmware runs
3. Linux CPU hotplug calls gic_cpu_init() which MUST:
   a) GICR_WAKER.ProcessorSleep = 0  ← THE CRITICAL STEP
   b) Poll until GICR_WAKER.ChildrenAsleep = 0 (HW acknowledges wake)
   c) ICC_PMR_EL1 = 0xF0
   d) ICC_IGRPEN1_EL1 = 1
```

**The Bug:**

```
If step 3a is missed (or fails silently):
- GICR_WAKER.ProcessorSleep = 1 (still sleeping)
- GICR_WAKER.ChildrenAsleep = 1
- Redistributor DISCARDS all interrupts targeted at Core2
- GICD_IROUTER says "send to Core2" → Redistributor says "I'm asleep, drop it"
- Interrupt stays Pending in Distributor but never signals CPU
```

**Debug steps:**

```bash
# Step 1: Read GICR_WAKER for Core2
# GICR base for Core2 = GICR_base + (core2_index * 0x20000)
devmem2 <GICR_WAKER_core2> w

# Check bits:
# Bit[1] ProcessorSleep: 1 = sleeping (BAD)
# Bit[2] ChildrenAsleep: 1 = children asleep (confirms sleep state)

# Step 2: Verify interrupt IS pending at Distributor
devmem2 <GICD_ISPENDR> w  # Is the interrupt Pending? YES → confirms GIC has it

# Step 3: Check CPU interface
# On Core2, read ICC_IGRPEN1_EL1 — is Group 1 enabled?
# Check ICC_PMR_EL1 — is priority mask allowing delivery?
```

**Fix:**

```c
// In CPU hotplug online callback:
void gic_cpu_online(int cpu) {
    void __iomem *rbase = gic_redistributor_base(cpu);
    
    // Wake redistributor
    u32 val = readl(rbase + GICR_WAKER);
    val &= ~GICR_WAKER_ProcessorSleep;  // Clear sleep bit
    writel(val, rbase + GICR_WAKER);
    
    // MUST wait for hardware acknowledgment
    while (readl(rbase + GICR_WAKER) & GICR_WAKER_ChildrenAsleep)
        cpu_relax();  // Spin until HW confirms wake
    
    // Now configure CPU interface
    write_sysreg(0xF0, ICC_PMR_EL1);
    write_sysreg(1, ICC_IGRPEN1_EL1);
    isb();
}
```

**Other possible causes (less common):**

```
1. ARE (Affinity Routing Enable) not set for Core2's context
   → GICD_CTLR.ARE_NS = 0 → GICv2 compatibility mode → IROUTER ignored
   → Check: GICD_CTLR

2. Core2 came up in wrong security state
   → NS-EL1 trying to receive Group 1S interrupts
   → Check: interrupt group assignment in GICD_IGROUPR

3. ICC_SRE_EL1 not set on Core2
   → System register interface not enabled → CPU interface inaccessible
   → GICv3 requirement: ICC_SRE_EL1.SRE must be 1

4. Power domain issue: core is ON but Redistributor power domain is OFF
   → Some SoCs have separate power for GIC Redistributor
   → Check SoC power controller status
```

**GICv2 equivalent issue:**
In GICv2, the equivalent is GICC not enabled (GICC_CTLR.EnableGrp1 = 0) after hotplug. No GICR_WAKER concept (no Redistributor).

**Common Mistakes:**
- Assuming GICD_IROUTER is sufficient (routing register is set, but Redistributor is asleep)
- Not knowing about GICR_WAKER (GICv3-specific)
- Forgetting to poll ChildrenAsleep (write ProcessorSleep=0 but don't wait for HW ack)
- Not checking ICC_IGRPEN1_EL1 on the newly online core

**What Interviewer Expects:**
- Immediate identification: GICR_WAKER as the prime suspect
- Knowledge of the wake sequence (clear ProcessorSleep, poll ChildrenAsleep)
- Understanding of Redistributor's role as gatekeeper per-PE
- Awareness of other hotplug-related issues (ARE, SRE, PMR)

---

### Q7: SYMPTOM: ICC_IAR read returns 1023 (spurious) but you're certain an interrupt was pending 10ns ago. No software disabled it. What happened?

**Simple Intuition:**
1023 means "nothing for you" at the exact moment you read. The interrupt was pending but something changed between the GIC signaling IRQ and your IAR read. This is a LEGITIMATE race condition, not a bug.

**Detailed Answer:**

**When GIC returns 1023 (architecturally defined):**

```
ICC_IAR returns 1023 when AT THE MOMENT OF THE READ:
1. No interrupt is pending with sufficient priority (below ICC_PMR)
2. No interrupt is pending for this CPU's group (group disabled)
3. No interrupt passes the BPR preemption check against running priority
4. The highest-priority pending interrupt was just taken by another core (1-of-N race)
```

**Scenario: The 10ns Race (1-of-N SPI)**

```
Setup: INTID 80 (SPI), 1-of-N routing, Core0 and Core1 both eligible

T0:     Device fires → INTID 80 Pending
T0+1ns: GIC signals IRQ to BOTH Core0 and Core1
        (GIC is evaluating which core to deliver to)
        
T0+5ns: Core0 reads ICC_IAR
        → GIC awards INTID 80 to Core0
        → Returns 80 to Core0
        → INTID 80: Pending → Active (on Core0)
        
T0+10ns: Core1 reads ICC_IAR
         → INTID 80 is already Active on Core0
         → No other interrupt pending for Core1
         → Returns 1023 (spurious)
         → Core1's IRQ signal was a valid signal at T0+1ns
            but the interrupt was "stolen" by Core0 at T0+5ns
```

**Other valid 1023 scenarios:**

```
Scenario B: Priority mask change
  T0: INTID pending, priority 0x80, PMR=0xF0 → qualifies → IRQ signaled
  T1: Another core writes ICC_PMR for this core = 0x00 (via IPI-triggered handler)
      (unlikely but architecturally possible via memory-mapped access)
  T2: This core reads IAR → priority now masked → 1023

Scenario C: Interrupt disabled between signal and read
  T0: INTID 80 pending, enabled → IRQ signaled to core
  T1: Another core disables INTID 80 (GICD_ICENABLER write)
      → GIC de-asserts pending, withdraws signal
  T2: Core reads IAR → 1023 (signal was valid when sent, invalid now)

Scenario D: Level-triggered line de-asserted
  T0: Level-sensitive device asserts → Pending → IRQ signaled
  T1: Device autonomously de-asserts (hardware timeout, DMA completion)
  T2: Core reads IAR → line is LOW → no longer Pending → 1023
```

**Correct Software Handling:**

```c
void irq_handler(void) {
    uint32_t intid = read_sysreg(ICC_IAR1_EL1);
    
    if (intid == 1023) {
        // SPURIOUS — legitimate, not an error
        // Do NOT write EOIR for 1023!
        // Simply return from exception
        return;
    }
    
    if (intid == 1022) {
        // GICv2 only: Group 1 interrupt when in Group 0 handler
        // Special handling needed
        return;
    }
    
    // Valid INTID: handle normally
    handle_interrupt(intid);
    write_sysreg(intid, ICC_EOIR1_EL1);
}
```

**Critical Rule: NEVER write EOIR for 1023**

```
If software writes ICC_EOIR1_EL1 = 1023:
→ Architecture says: write is IGNORED (no effect)
→ But some implementations may misbehave
→ Best practice: always check before EOIR write
```

**How frequent should spurious interrupts be?**

```
Normal system: very rare (one per million interrupts)
  → Occasional race with 1-of-N routing

Frequent spurious (>1% of total): indicates a problem
  → Device rapidly asserting/de-asserting line
  → Priority mask being modified during delivery
  → SPI routing configuration unstable
  
Linux tracks this: cat /proc/irq/<N>/spurious
```

**Common Mistakes:**
- Treating 1023 as a bug (it's architecturally expected)
- Writing EOIR for 1023 (harmless but incorrect practice)
- Not handling 1023 in ISR (causing crash on null dereference of handler table)
- Blaming GIC hardware for "losing interrupts" when it's a legitimate race

**What Interviewer Expects:**
- Immediate recognition: 1023 is valid, expected, and race-related
- At least 2 concrete scenarios that produce 1023
- Correct software handling (check before EOIR, just return)
- Understanding that 1-of-N routing is the most common cause in multi-core

---

### Q8: SYMPTOM: In a system with EOImode=1, after 16 interrupts, no more interrupts are delivered to ANY core. System is live but interrupt-dead. What happened?

**Simple Intuition:**
EOImode=1 splits priority drop (EOIR) from deactivation (DIR). If software does EOIR but forgets DIR, the interrupt stays ACTIVE forever. After enough interrupts accumulate in ACTIVE state, all priority levels are consumed and nothing new can preempt or be delivered.

**Detailed Answer:**

**Root Cause: ICC_DIR never written — Active list full**

```
The GIC has a maximum number of simultaneously Active interrupts per CPU.
This is limited by the Active Priority Registers (ICC_APR<n>_EL1).
Typical: 4 APR registers × 32 bits = 128 priority levels trackable
But ACTUALLY: limited by the number of unique group-priority levels

With 4 implemented priority bits (16 levels):
- After 16 interrupts go Active without deactivation
- All priority levels have an Active entry
- Running priority = lowest possible (0xF0)
- No new interrupt can have LOWER value than running priority
- ALL interrupts are blocked
```

**Detailed mechanism:**

```
T0:  INTID 32 (prio 0x00) → IAR → Active → EOIR (prio drop) → NO DIR
     Active state: INTID 32 still Active, but running prio dropped
     
T1:  INTID 33 (prio 0x10) → IAR → Active → EOIR → NO DIR
T2:  INTID 34 (prio 0x20) → IAR → Active → EOIR → NO DIR
...
T15: INTID 47 (prio 0xF0) → IAR → Active → EOIR → NO DIR

State after T15:
- 16 interrupts stuck in ACTIVE state
- All priority levels occupied in Active Priority Registers
- Running priority after EOIR drops, BUT...
- GIC tracks that priorities 0x00-0xF0 ALL have active interrupts
- New interrupt at ANY priority: blocked because it would need to
  be higher than ALL active priorities combined (no room)

Actually, the precise rule:
- Even though EOIR drops the running priority, the GIC still tracks
  active priorities in ICC_AP1R<n>_EL1 registers
- With all priority groups having an active entry:
  ICC_RPR_EL1 shows the "minimum active priority"
  New interrupts need priority < ICC_RPR to signal
  But NOTHING is lower than 0x00 (already Active)
  → Complete deadlock
```

**Wait — doesn't EOIR drop the running priority?**

```
YES, but there's a subtlety:

ICC_RPR (Running Priority Register) reflects:
- EOImode=0: highest priority among Active interrupts on this CPU
- EOImode=1: highest priority among NON-priority-dropped Active interrupts

After EOIR (priority drop):
- That specific entry is "priority dropped" — doesn't contribute to RPR
- BUT the Active state remains in the AP registers
- The AP register bit is NOT cleared by EOIR in mode=1

When AP registers are full:
- Architecture implementations vary, but many cannot track more Active
  interrupts than their AP register capacity
- Some implementations: IAR returns 1023 when AP bits exhausted
- Others: hang condition

The exact "16 interrupt" limit = implementation-specific
(Depends on priority bits implemented and AP register count)
```

**Debug:**

```bash
# Check Active Priority Registers
# GICv3: ICC_AP1R0_EL1 through ICC_AP1R3_EL1
# If all bits set → Active list overflow

# Check ISACTIVER for all INTIDs
devmem2 <GICD_ISACTIVER0>  # Check which INTIDs are stuck Active
devmem2 <GICD_ISACTIVER1>
# Multiple bits set = confirmation of leak

# Check if DIR is being called
# Trace/instrument the interrupt exit path
```

**Fix:**

```c
// BROKEN (hypervisor forgot DIR):
void interrupt_handler(void) {
    uint32_t intid = read_sysreg(ICC_IAR1_EL1);
    write_sysreg(intid, ICC_EOIR1_EL1);  // Priority drop ✓
    inject_virtual(intid);  // Inject to guest
    // ← MISSING: eventually call write_sysreg(intid, ICC_DIR_EL1)
}

// FIXED (with HW=1 in LR):
// Physical DIR happens automatically when guest writes virtual EOIR
// Hypervisor only needs to set HW=1 in the List Register
// If HW=0: hypervisor must track and manually issue DIR later
```

**Recovery (without reboot):**

```bash
# For each stuck-Active INTID, write ICC_DIR:
# OR: Write GICD_ICACTIVER to force-clear Active (DANGEROUS)
devmem2 <GICD_ICACTIVER_offset> <bitmask>
# This violates protocol but recovers from deadlock
```

**Common Mistakes:**
- Not understanding that EOImode=1 requires BOTH EOIR and DIR
- Thinking EOIR in mode=1 fully completes the interrupt
- Not knowing about Active Priority Register capacity limits
- Missing that this is a GRADUAL failure (works for first N interrupts, then dies)

**What Interviewer Expects:**
- Understanding of the split EOI model (EOIR ≠ full completion in mode=1)
- Knowledge that Active state accumulation is the mechanism
- Awareness that this is the #1 hypervisor GIC bug (forgetting DIR)
- Understanding of AP registers and their capacity

---

### Q9: SYMPTOM: INTID 50 (priority 0x20) fires AFTER INTID 51 (priority 0x80) even though INTID 50 should be higher priority (lower value = higher). Both targeted at Core0. /proc/interrupts shows INTID 51 handled at timestamp T, INTID 50 at T+100μs. Explain all possible causes.

**Simple Intuition:**
Higher priority should fire first — but there are several reasons it might not: they didn't go pending simultaneously, priority registers aren't what you think, or they went to different cores.

**Detailed Answer:**

**Possible Causes (ordered by likelihood):**

**Cause 1: Non-simultaneous pending (MOST LIKELY)**

```
What people ASSUME:
  "Both interrupts are pending, GIC should pick higher priority first"

What ACTUALLY happened:
  T0: INTID 51 goes Pending (device 51 fires first)
  T1: Core0 reads IAR → returns 51 (only pending interrupt)
  T2: INTID 50 goes Pending (device 50 fires later)
      → But Core0 is already handling INTID 51
      → Same priority group (BPR) → NO PREEMPTION
  T3: Core0 finishes INTID 51, EOI
  T4: Core0 reads IAR → returns 50
  
  Result: 51 handled before 50 even though 50 is "higher priority"
  Reason: 50 wasn't pending when the decision was made
  
  This is NOT a bug — correct GIC behavior.
```

**Cause 2: BPR preventing preemption**

```
Setup:
  INTID 50 priority = 0x20 → group priority (BPR=4): bits[7:4] = 0x2
  INTID 51 priority = 0x80 → group priority (BPR=4): bits[7:4] = 0x8
  
  IF both are pending simultaneously:
  → 0x2 < 0x8 → INTID 50 should win ✓
  
  BUT if INTID 51 is Active and INTID 50 arrives:
  → Running priority = 0x80, group priority of running = 0x8
  → INTID 50 group priority = 0x2, which IS lower → SHOULD preempt
  → If BPR is set such that group priorities are EQUAL: no preemption
  
  Check: BPR=0 → group bits = [7:1] → groups clearly different → should preempt
  Check: BPR=7 → group bits = [7] only → 0x20 bit[7]=0, 0x80 bit[7]=0 → SAME GROUP
  
  BPR=7 → no preemption even with different numeric priorities!
```

**Cause 3: Priority register not programmed as expected**

```
What you THINK:
  GICD_IPRIORITYR[50] = 0x20
  GICD_IPRIORITYR[51] = 0x80

What ACTUALLY may be true:
  Linux set both to 0xA0 (flat priority — default!)
  You're reading stale/wrong values
  
  Debug: devmem2 <GICD_IPRIORITYR for INTID 50>
         devmem2 <GICD_IPRIORITYR for INTID 51>
  → Confirm actual values match expectations
```

**Cause 4: Different cores (1-of-N routing)**

```
If both use 1-of-N routing:
  → INTID 51 went to Core0 (delivered instantly)
  → INTID 50 went to Core1 (delivered at same time)
  → But you're only looking at Core0's /proc/interrupts
  
  Or: INTID 50 went to Core1 which is in WFI
  → Takes time to wake → appears 100μs "late"
  
  Debug: cat /proc/interrupts (check per-CPU columns)
```

**Cause 5: ICC_PMR was temporarily masking**

```
If during INTID 51's handler:
  → Software temporarily sets ICC_PMR = 0x00 (mask all)
  → INTID 50 arrives → blocked by PMR
  → Handler restores PMR → 50 now delivered
  → Appears as "delayed"
  
  This happens with Linux's local_irq_save/restore within handlers.
```

**Cause 6: INTID 50 is in wrong group**

```
If INTID 50 is Group 0 and INTID 51 is Group 1:
  → ICC_IAR1 only returns Group 1 interrupts
  → INTID 50 requires ICC_IAR0 (or EL3 handling)
  → If software only reads IAR1 → INTID 50 never acknowledged via that path
  
  Debug: check GICD_IGROUPR, GICD_IGRPMODR
```

**Systematic Debug Checklist:**

```
1. Were they SIMULTANEOUSLY pending? → Check timestamps of device events
2. Read actual GICD_IPRIORITYR → Confirm priorities are what you expect
3. Read ICC_BPR1 → Calculate group priority split
4. Check GICD_IROUTER → Same core or different?
5. Check GICD_IGROUPR → Same group?
6. Check ICC_PMR → Was it temporarily restricted?
7. Trace IAR reads with timestamps → What was returned when?
```

**Common Mistakes:**
- Assuming simultaneous pending when devices fire at different times
- Not checking actual register values (assuming priorities are programmed)
- Ignoring BPR's effect on preemption granularity
- Forgetting that Linux uses flat priorities by default

**What Interviewer Expects:**
- "First, were they simultaneously pending?" (most common explanation)
- Multiple hypotheses with discrimination strategy
- Knowledge of BPR's role in preventing preemption
- Practical debug approach (check registers, don't assume)

---

## Topic 11: Verification

---

### Q10: Design a verification strategy to prove that ICC_IAR correctly returns 1023 (spurious) under all required conditions. What stimulus do you need and what coverage holes do teams commonly miss?

**Simple Intuition:**
Proving 1023 correctness means proving a NEGATIVE — "there is nothing to deliver." You must show that for every reason an interrupt could be blocked, IAR correctly returns 1023 instead of a valid INTID.

**Detailed Answer:**

**All architecturally-defined 1023 conditions:**

```
IAR must return 1023 when:
C1: No interrupt is Pending (trivial case)
C2: Pending interrupt exists but priority ≥ ICC_PMR (masked)
C3: Pending interrupt exists but group is disabled (ICC_IGRPEN1=0)
C4: Pending interrupt exists but group priority ≤ running priority (can't preempt)
C5: Pending interrupt was Pending but became Inactive between signal and read (race)
C6: Pending interrupt targeted another PE (1-of-N, taken by other core)
C7: Pending interrupt's INTID was disabled (GICD_ICENABLER) between signal and read
```

**Test Plan:**

```
═══════════════════════════════════════════════════
Category 1: Static masking (no race conditions)
═══════════════════════════════════════════════════

Test 1.1 — PMR mask:
  Setup:   INTID 50 Pending, priority = 0x40, ICC_PMR = 0x20
  Stimulus: Read ICC_IAR
  Expected: 1023 (priority 0x40 ≥ PMR 0x20, masked)
  Variants: Test at all priority boundary values

Test 1.2 — Group disabled:
  Setup:   INTID 50 Pending, Group 1, ICC_IGRPEN1 = 0
  Stimulus: Read ICC_IAR
  Expected: 1023

Test 1.3 — Running priority blocks:
  Setup:   INTID 50 Active (prio 0x20), INTID 60 Pending (prio 0x40)
  Stimulus: Read ICC_IAR (while INTID 50 Active)
  Expected: 1023 (0x40 cannot preempt 0x20)
  Variant:  Test with BPR making same-group

Test 1.4 — Interrupt disabled at Distributor:
  Setup:   INTID 50 asserted (line HIGH), but GICD_ISENABLER bit = 0
  Stimulus: Read ICC_IAR
  Expected: 1023

═══════════════════════════════════════════════════
Category 2: Race conditions (dynamic)
═══════════════════════════════════════════════════

Test 2.1 — PMR change between signal and IAR:
  Setup:   INTID 50 Pending, PMR=0xF0 → IRQ signaled to CPU
  Race:    Before IAR read, change PMR to 0x00
  Stimulus: Read ICC_IAR
  Expected: 1023 (current PMR masks it)
  Coverage: Vary timing of PMR write relative to IAR read (cycle-accurate)

Test 2.2 — Interrupt disabled between signal and IAR:
  Setup:   INTID 50 Pending → IRQ signaled
  Race:    Another core writes GICD_ICENABLER[50] = 0
  Stimulus: Read ICC_IAR
  Expected: 1023

Test 2.3 — 1-of-N stolen by other core:
  Setup:   INTID 50 Pending, 1-of-N, Core0 and Core1 eligible
  Race:    Core1 reads IAR at T, Core0 reads IAR at T+1
  Stimulus: Core0's IAR read
  Expected: 1023 (Core1 won the race)
  Coverage: Vary relative timing between the two IAR reads

Test 2.4 — Level de-assertion between signal and IAR:
  Setup:   Level-triggered INTID 50, line HIGH → Pending → IRQ signaled
  Race:    Device de-asserts line (hardware event)
  Stimulus: Read ICC_IAR
  Expected: 1023 (no longer pending)
  Coverage: De-assertion at every cycle between signal and read

Test 2.5 — Group disabled between signal and IAR:
  Setup:   INTID Pending, group enabled, IRQ signaled
  Race:    ICC_IGRPEN1 = 0 written before IAR read
  Expected: 1023

═══════════════════════════════════════════════════
Category 3: Boundary and corner cases
═══════════════════════════════════════════════════

Test 3.1 — PMR boundary (priority = PMR exactly):
  Setup:   INTID priority = 0x80, PMR = 0x80
  Expected: 1023 (equal means masked — priority must be LOWER than PMR)
  WHY: This is a common implementation bug — off-by-one in comparator

Test 3.2 — Multiple pending, ALL masked:
  Setup:   10 interrupts pending, PMR masks all
  Expected: 1023 (not any of them)

Test 3.3 — Interrupt pending on OTHER group:
  Setup:   INTID in Group 0, read ICC_IAR1 (Group 1 register)
  Expected: 1023 (Group 0 interrupt not visible via IAR1)

Test 3.4 — No interrupt pending (quiescent system):
  Setup:   All INTIDs Inactive, nothing pending
  Race:    IRQ line de-asserted before IAR read (glitch or late clear)
  Stimulus: Read ICC_IAR
  Expected: 1023
  WHY: Some implementations may not perfectly deglitch the IRQ signal

Test 3.5 — Active+Pending, same interrupt, no preemption:
  Setup:   INTID 50 Active+Pending, same priority as running
  Stimulus: Read ICC_IAR
  Expected: 1023 (cannot preempt self — same group priority)
```

**Coverage Model:**

```systemverilog
covergroup spurious_1023_coverage;
  // Which condition caused 1023
  cause: coverpoint spurious_cause {
    bins pmr_mask       = {PMR_BLOCKS};
    bins group_disabled = {GROUP_OFF};
    bins running_prio   = {PRIORITY_BLOCKS};
    bins race_stolen    = {OTHER_CORE_WON};
    bins race_disabled  = {DISABLED_DURING};
    bins race_level_low = {LEVEL_DEASSERTED};
    bins race_pmr_change = {PMR_CHANGED};
    bins no_pending      = {NOTHING_PENDING};
    bins wrong_group     = {WRONG_GROUP_IAR};
  }
  
  // Timing of race (cycles between signal and IAR read)
  race_timing: coverpoint cycles_signal_to_iar {
    bins immediate   = {[0:1]};
    bins short_race  = {[2:5]};
    bins medium_race = {[6:20]};
    bins long_race   = {[21:100]};
  }
  
  // Cross: race cause × timing
  cross cause, race_timing {
    ignore_bins non_race = binsof(cause) intersect {PMR_BLOCKS, GROUP_OFF, 
                                                     NOTHING_PENDING} &&
                           binsof(race_timing) intersect {[2:100]};
  }
  
  // Number of pending interrupts when 1023 returned
  pending_count: coverpoint num_pending_at_read {
    bins zero       = {0};
    bins one        = {1};
    bins few        = {[2:5]};
    bins many       = {[6:32]};
  }
endgroup
```

**Assertions:**

```systemverilog
// If IAR returns 1023, NO interrupt should have been deliverable
assert_1023_correct: assert property (
  @(posedge clk) (iar_read && iar_value == 1023) |->
    !exists_deliverable_interrupt(cpu_id)
);

// Helper: "deliverable" means pending + enabled + group_on + prio < pmr + prio < running
function bit exists_deliverable_interrupt(int cpu);
  foreach (intid[i]) begin
    if (is_pending[i] && is_enabled[i] && group_enabled[i] &&
        priority[i] < pmr[cpu] && group_priority[i] < running_priority[cpu] &&
        routed_to(i, cpu))
      return 1;
  end
  return 0;
endfunction

// Converse: if deliverable interrupt exists, IAR must NOT return 1023
assert_no_false_spurious: assert property (
  @(posedge clk) (iar_read && exists_deliverable_interrupt(cpu_id)) |->
    iar_value != 1023
);

// 1023 must NOT change any GIC state
assert_1023_no_side_effect: assert property (
  @(posedge clk) (iar_read && iar_value == 1023) |=>
    $stable({all_active_bits, all_pending_bits, running_priority})
);
```

**Coverage Holes Commonly Missed:**

```
1. PMR boundary (equal priority = masked) — off-by-one bug
2. Race timing sweep (need cycle-granularity variation)
3. Multi-core 1-of-N with >2 cores racing simultaneously
4. 1023 returned while Active+Pending (same interrupt, same priority)
5. Group 0 pending while reading Group 1 IAR (cross-group masking)
6. SecurityState masking (NS reading IAR for Secure-group interrupt)
7. State assertion: proving NO side-effect on 1023 read (no state corruption)
```

**Common Mistakes in Verification:**
- Only testing the "no interrupt pending" case (trivial 1023)
- Not sweeping race timing (test only exact-same-cycle or widely-separated)
- Missing the PMR boundary condition (exact equality)
- Not verifying the CONVERSE (if pending+deliverable, must NOT return 1023)

**What Interviewer Expects:**
- Exhaustive list of 1023 conditions (not just "nothing pending")
- Race-oriented test strategies with timing variation
- Both positive check (1023 correct) and negative check (no false 1023)
- Coverage model that ensures all conditions hit
- Knowledge of common implementation bugs (boundary comparator)

---

### Q11: You need to verify multi-core interrupt arbitration for 1-of-N SPIs. Two cores read ICC_IAR within 1 cycle of each other for the same pending SPI. What are all valid outcomes, and how do you build a checker?

**Simple Intuition:**
Only ONE core can win a given interrupt. The checker must verify: exactly one gets the INTID, the other gets 1023, and the interrupt moves to Active on exactly one core. The hard part: there's no architectural guarantee about WHICH core wins.

**Detailed Answer:**

**Valid Outcomes (architecture):**

```
Setup: INTID 80 Pending, 1-of-N, Core0 and Core1 both eligible
       Both read ICC_IAR within 1 cycle

VALID Outcome A:
  Core0 IAR → 80 (wins)
  Core1 IAR → 1023 (loses)
  INTID 80: Active on Core0

VALID Outcome B:
  Core0 IAR → 1023 (loses)
  Core1 IAR → 80 (wins)
  INTID 80: Active on Core1

INVALID Outcomes:
  ✗ Both return 80 → double-delivery (SILICON BUG)
  ✗ Both return 1023 → interrupt lost (SILICON BUG)
  ✗ One returns 80, interrupt Active on wrong core → state corruption
```

**Checker Architecture:**

```systemverilog
// Multi-core arbiter checker
module gic_1ofn_checker;
  
  // Monitor all IAR reads across all cores
  logic [31:0] iar_return [NUM_CORES];
  logic        iar_valid  [NUM_CORES];
  
  // For each pending SPI with 1-of-N routing:
  always @(posedge clk) begin
    foreach (spi_id[s]) begin
      if (is_pending_1ofn[s]) begin
        
        // Collect all IAR reads this cycle that could claim this SPI
        int claimants = 0;
        int winners = 0;
        int winner_core = -1;
        
        foreach (core[c]) begin
          if (iar_valid[c]) begin
            claimants++;
            if (iar_return[c] == spi_id[s]) begin
              winners++;
              winner_core = c;
            end
          end
        end
        
        // ASSERTION 1: At most one winner
        assert (winners <= 1) 
          else $error("Double delivery: SPI %0d delivered to %0d cores", s, winners);
        
        // ASSERTION 2: If claimants > 0 and SPI was pending, exactly one must win
        // (unless SPI was masked/disabled in between)
        if (claimants > 0 && still_deliverable[s])
          assert (winners == 1)
            else $error("SPI %0d lost: %0d claimants, 0 winners", s, claimants);
        
        // ASSERTION 3: Winner's core ID matches Active state
        if (winners == 1)
          assert (active_on_core[s] == winner_core)
            else $error("Active state mismatch: SPI %0d active on wrong core", s);
      end
    end
  end
  
endmodule
```

**Stimulus Strategy:**

```
Strategy 1: Tight race (hardest to implement, most bug-finding)
  → Force both cores to execute IAR read in same clock cycle
  → Varies by 0, 1, 2, 3 cycles offset
  → Use UVM sequence synchronization or testbench clock control

Strategy 2: Pipeline contention
  → Fill pipeline: multiple SPIs pending, each 1-of-N
  → All cores reading IAR rapidly
  → Tests arbiter under sustained load

Strategy 3: Asymmetric eligibility
  → Core0 has PMR=0xF0 (all pass), Core1 has PMR=0x10 (most blocked)
  → SPI priority = 0x20 (passes Core0's PMR, fails Core1's)
  → Only Core0 should win (ever)
  → Verify Core1 NEVER gets this SPI

Strategy 4: Dynamic eligibility change
  → Both eligible → Core1's PMR changes to 0x00 → only Core0 eligible
  → Time the PMR change relative to arbiter decision
  → Verify no delivery to newly-ineligible core

Strategy 5: Scalability
  → 4, 8, 16 cores all eligible for same SPI
  → Verify exactly-one-winner scales
  → Check no starvation (if implementation claims fairness)
```

**Coverage Model:**

```systemverilog
covergroup one_of_n_arbitration;
  // Number of simultaneous eligible claimants
  num_claimants: coverpoint claimant_count {
    bins two    = {2};
    bins three  = {3};
    bins four   = {4};
    bins many   = {[5:16]};
  }
  
  // Which core won
  winner: coverpoint winner_core_id {
    bins core[] = {[0:NUM_CORES-1]};
  }
  
  // Timing offset between IAR reads
  read_offset: coverpoint cycle_offset_between_reads {
    bins same_cycle = {0};
    bins one_apart  = {1};
    bins two_apart  = {2};
    bins close      = {[3:10]};
  }
  
  // Cross: ensure all cores can win across all timing scenarios
  cross num_claimants, winner, read_offset;
  
  // Fairness (if implementation specifies):
  winner_distribution: coverpoint winner_core_id {
    // After N trials, each core should have won at least once
    // (functional coverage for statistical fairness)
  }
endgroup
```

**Bugs This Catches:**

```
Bug 1: Double-grant (arbiter sends ACK to two cores)
  → Both get valid INTID → state corruption
  → Caught by: assertion "winners <= 1"

Bug 2: Deadlock (arbiter doesn't grant to anyone)
  → Both get 1023 → interrupt stuck Pending forever
  → Caught by: assertion "winners == 1 when deliverable"

Bug 3: Stale routing (arbiter sends to core that became ineligible)
  → Core that just changed PMR still gets delivery
  → Caught by: check eligibility at time of delivery, not request

Bug 4: Priority inversion in arbiter
  → Lower-priority SPI wins over higher-priority during simultaneous read
  → Caught by: ordering assertion (highest priority always acknowledged first per core)
```

**Common Mistakes:**
- Only testing 2-core races (bugs emerge at 3+ cores)
- Not varying timing offset (same-cycle only)
- Checking only "one winner" without checking "exactly one" (miss the loss case)
- Not testing dynamic eligibility changes during arbitration
- Assuming deterministic winner (test must accept either core winning)

**What Interviewer Expects:**
- Clear statement of valid vs. invalid outcomes
- Non-deterministic checker (accepts either winner)
- Race timing as a primary coverage axis
- Scalability awareness (2-core race ≠ 8-core race)
- Both safety (no double-delivery) and liveness (no loss) assertions

---

### Q12: You're writing assertions for EOImode=0 vs EOImode=1 compliance. What properties MUST hold for each mode, and what's the most common silicon bug between the two?

**Simple Intuition:**
EOImode controls whether one write (EOIR) does everything, or two writes (EOIR + DIR) are needed. The assertion challenge: verify that EOIR in mode=1 does ONLY priority drop without state change, and DIR does ONLY deactivation without priority change.

**Detailed Answer:**

**Properties for EOImode=0:**

```
Property P1: EOIR write → Active → Inactive (single transition)
  assert_eoi_mode0_deactivates: assert property (
    @(posedge clk)
    (eoi_mode == 0 && eoir_write && intid_active[eoir_intid] && !intid_pending[eoir_intid])
    |=> !intid_active[eoir_intid]
  );

Property P2: EOIR write → Active+Pending → Pending
  assert_eoi_mode0_actpend: assert property (
    @(posedge clk)
    (eoi_mode == 0 && eoir_write && intid_active[eoir_intid] && intid_pending[eoir_intid])
    |=> (!intid_active[eoir_intid] && intid_pending[eoir_intid])
  );

Property P3: EOIR write → running priority drops
  assert_eoi_mode0_prio_drop: assert property (
    @(posedge clk)
    (eoi_mode == 0 && eoir_write && valid_eoir)
    |=> (running_priority > $past(running_priority) || running_priority == IDLE_PRIORITY)
  );
  // Note: "drops" means numeric value increases (lower urgency) or goes to idle (0xFF)

Property P4: DIR write in mode=0 → no effect (architecturally UNPREDICTABLE, verify impl)
  // Some implementations: DIR is ignored in mode=0
  // Others: UNPREDICTABLE behavior
  // Verify YOUR implementation matches its TRM
```

**Properties for EOImode=1:**

```
Property P5: EOIR write → priority drops but Active state PRESERVED
  assert_eoi_mode1_prio_only: assert property (
    @(posedge clk)
    (eoi_mode == 1 && eoir_write && valid_eoir)
    |=> (intid_active[eoir_intid] &&    // STILL Active!
         running_priority > $past(running_priority))
  );

Property P6: DIR write → Active → Inactive (deactivate only)
  assert_dir_deactivates: assert property (
    @(posedge clk)
    (eoi_mode == 1 && dir_write && intid_active[dir_intid])
    |=> !intid_active[dir_intid]
  );

Property P7: DIR write → running priority UNCHANGED
  assert_dir_no_prio_change: assert property (
    @(posedge clk)
    (eoi_mode == 1 && dir_write)
    |=> $stable(running_priority)
  );
  // DIR doesn't affect priority — it was already dropped by EOIR

Property P8: EOIR without subsequent DIR → Active state persists indefinitely
  assert_mode1_needs_dir: assert property (
    @(posedge clk)
    (eoi_mode == 1 && eoir_write && valid_eoir) |=>
      (intid_active[eoir_intid] throughout 
       (!dir_write || dir_intid != eoir_intid)[*1:$])
  );
  // Active stays until DIR specifically for this INTID
```

**Cross-mode transition properties:**

```
Property P9: Changing EOImode while interrupts are Active
  // Architecture says this is UNPREDICTABLE in most cases
  // Verify implementation doesn't corrupt state
  assert_mode_change_no_corrupt: assert property (
    @(posedge clk)
    (eoi_mode_changed && any_active) |=>
      $stable(active_count)  // At minimum, don't lose or gain active entries
  );
```

**The Most Common Silicon Bug:**

```
═══════════════════════════════════════════════════════════
BUG: EOIR in mode=1 accidentally deactivates (like mode=0)
═══════════════════════════════════════════════════════════

Root cause: Shared RTL path between mode=0 and mode=1
  → Mode-select signal not properly gating the deactivation logic
  → EOIR triggers full completion regardless of mode setting
  
Symptom:
  - Mode=1 appears to work (priority drops)
  - But Active state also clears (it shouldn't)
  - Hypervisor doesn't notice until guest does EOIR for virtual interrupt
  - Physical interrupt already deactivated → device can fire again
  → Possible double-delivery or state confusion
  
Why it's missed in basic testing:
  - If you only check "priority dropped" → passes
  - If you only check "interrupt eventually becomes inactive" → passes
  - Only fails if you specifically assert Active PERSISTS after EOIR in mode=1

This is why Property P5 is THE critical assertion.
```

**Second Most Common Bug:**

```
═══════════════════════════════════════════════════════════
BUG: DIR in mode=1 also drops priority (double-drop)
═══════════════════════════════════════════════════════════

Root cause: DIR handler shares logic with EOIR
  → Priority drop occurs at DIR time (already dropped at EOIR time)
  → Running priority goes lower than it should
  
Symptom:
  - After DIR, an interrupt that shouldn't be deliverable gets through
  - Priority inversion
  
Caught by: Property P7 ($stable(running_priority) after DIR)
```

**Complete Verification Strategy:**

```
1. Test all sequences:
   Mode=0: IAR → EOIR → verify (Inactive + prio drop)
   Mode=1: IAR → EOIR → verify (Active + prio drop) → DIR → verify (Inactive)

2. Interleaving:
   Mode=1: IAR(A) → IAR(B) → EOIR(A) → EOIR(B) → DIR(B) → DIR(A)
   Verify: Each INTID independently tracked

3. Mode switching:
   Start mode=0 → change to mode=1 with Active interrupt → verify behavior
   
4. Error injection:
   EOIR for non-Active INTID → verify no corruption
   DIR for non-Active INTID → verify no corruption
   DIR in mode=0 → verify impl-defined behavior matches TRM

5. Performance:
   Rapid IAR/EOIR/DIR cycles → verify no accumulation bugs
   Saturate AP registers → verify correct blocking/1023
```

**Common Mistakes in Verification:**
- Only asserting the "happy path" (write does what it should) without asserting "doesn't do what it shouldn't"
- Not testing interleaved EOIR/DIR for multiple interrupts
- Missing the mode-change-while-active corner case
- Not checking running_priority independently from Active state
- Treating mode=0 and mode=1 as completely separate tests (they share RTL)

**What Interviewer Expects:**
- Paired assertions: what SHOULD happen AND what SHOULD NOT happen
- Knowledge of the common silicon bug (EOIR deactivating in mode=1)
- Understanding that EOIR and DIR have ORTHOGONAL effects in mode=1
- Cross-mode transition awareness
- Emphasis on P5 as the critical differentiating assertion

---

## Quick Reference: Virtualization Registers

| Register | Purpose | Layer |
|----------|---------|-------|
| ICH_LR<n>_EL2 | List Registers (virtual interrupt state) | Hypervisor |
| ICH_HCR_EL2 | Hypervisor Control (maintenance enables) | Hypervisor |
| ICH_VMCR_EL2 | Virtual Machine Control (virtual PMR/BPR/EOImode) | Hypervisor |
| ICH_VTR_EL2 | Virtualization Type (LR count, priority bits) | Read-only |
| ICH_MISR_EL2 | Maintenance Interrupt Status | Hypervisor |
| ICH_AP1R<n>_EL2 | Virtual Active Priorities | Hypervisor |
| ICC_IAR1_EL1 | Guest reads → virtual CPU interface returns from LR | Guest (trapped or direct) |
| ICC_EOIR1_EL1 | Guest writes → virtual deactivation + HW auto-DIR | Guest |

---

## Summary of Key Principles (Part 3)

1. **EOImode=1 + HW bit** = the optimal virtualization path (no trap on guest EOI)
2. **LRs are limited** — maintenance interrupt is the overflow mechanism
3. **GICv4 eliminates hypervisor traps** for direct-injected LPIs (vPE scheduling)
4. **Stuck ACTIVE = missed EOI/DIR** — always check GICD_ISACTIVER first
5. **1023 is legitimate** — race between signal and IAR read, not a bug
6. **GICR_WAKER** is the #1 post-hotplug failure cause
7. **"Works once then dies"** = stuck ACTIVE or device not re-arming
8. **Interrupt storm = level device + source not cleared** (90% of cases)
9. **Multi-core arbitration verification** needs both safety (no double-grant) and liveness (no loss)
10. **EOImode=1 common bug**: EOIR accidentally deactivates — assert Active PERSISTS after EOIR
