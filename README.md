# KTRR (as of 2024)
Originally posted: https://gist.github.com/kienhho/9c3e55c44b07836478f120a6bf4b118a
## Background
1. Apple Memory and Cache Controller (AMCC) is separate from the CPU and controls cache and memory. All memory or cache read and write is controlled by the AMCC. Important values are stored in MMIO registers, which is a part of DRAM.
2. Memory Management Unit (MMU) is a CPU part, deal with translating  virtual address (VA) to physical address (PA) and also manages memory permission. Since the data are stored in CPU register, get lost when core sleeps
3. Virtual Memory System Architecture (VMSA) are CPU registers that dictates the virtual memory processes. Of them, 2  registers of our concerns are:
* TTBR: Kernel page table, MMU uses information here to map the correct virtual address to physical address
* SCTLR: Security policy register, the least significant bit (bit 0) decides whether MMU is on or off

## KTRR

**Overview:**

KTRR protection is implemented on both MMU and AMCC. The MMU protects CPU programs from writing to the protected range and making EL1 (kernel) code outside of the range executable. However, this would not be enough, as any coprocessors with direct memory access (DMA) would be out of MMU protection domain. Therefore, KTRR protection must also be implemented in the AMCC to  protect read-only region from being written to and panic when a write operation happens.

**Details:**

To mark the range that is protected under KTRR, AMCC stores the RoRgn begin, end, and lock values in the MMIO registers, while MMU store the KTRR begin, end, and lock values in the CPU registers. During the boot process, iBoot computes the kernel start and end address, writing it to the MMIO register and also stashes it in the kernel ``_DATA_const_`` region. Once the kernel has been fully bootstraped, the AMCC lock is set to 1 and the MMU takes the value from the stashed address in ``_DATA_const_`` and also proceed to locking it down. Once the lock is set to a nonzero value, the three fields itself are also protected. For AMCC, since the registers are stored on memory, they will stay locked during the device runtime, whereas for MMU, the registers value will get lost when core sleeps and to be locked down again using the stashed value every time the core resets. Fortunately, the instruction to initialize KTRR protection happens the first thing when core starts and has returns or branching to redirect the control flow.

Aside from implementing KTRR, additional protections must also be done to VMSA registers to prevent an attacker from working around KTRR, as controlling TTBR could allow an attackter to remap the RoRgn region to an address not protected by AMCC, while controlling SCTLR allows disabling the MMU and thus, its execution region protection. To still allow the CPU to set these registers during core resets, but prevent users from ROPping into those instructions, they are stored in the ``_LAST_pinst_`` section, which has instructions that are marked non-executable when MMU is on, but is executable during the reset vector as MMU has not yet started.

Therefore, it can be concluded that the MMU KTRR range has to include one less page than the AMCC KTRR range, omitting the ``_LAST_pinst_`` page.

## Past attacks:
Since the CPU register values are lost and get reset frequently, this is the most likely attack surface. Both past public bypasses couldn’t bypass KTRR on AMCC, but instead modified system registers to bypass KTRR:
* iOS 10.1.1 KTRR bypass rely on an off-by-one error in the ``_LAST_pinst_`` page range, where the instruction ``msr TTBR1_EL1, x0`` is accidentially left executable, allowing for a ROP chain into that instruction to change the root page table and remap ``_DATA_const_`` into a writable PA region.
* A11 chip KTRR bypass rely on the ``dbgwrap`` debug register to control the reset vector. When debugged, the reset vector is modified, allowing the KTRR on MMU lock to be skipped, and TTBR1 to be modified and redirecting ``_TEXT_exec_`` page be to a in a writable physical mapping.

## A12 hardenings:
A12 chip introduced an update to KTRR, now called Configurable Text Read-only Region (CTRR). Unlike KTRR, which protected the VMSA registers by making instructions modifying such registers non-executable, with CTRR, the register itself is now made read-only. After the initial write needed during core reset, the register holding VMSA write permission, ``S3_4_c15_c1_2``, allegedly called the ``VMSA_LOCK``, will be modified to disallow write to itself and the VMSA registers. With the stronger garuntee that this mechanism brings, the instructions to modify these registers has been moved out of the ``_LAST_pinst_`` section and made executable again. However, for attackers, this means that the only way to modify any of these registers would have to be gaining code execution on core reset, before the lock is applied.

## Possible methods:
1. CPU debug register ``dbgwrap``:

The debug register was used in KTRW for the A11 KTRR bypass, but PPL prevented the register from being mapped on A12 or later. It could probably be used again if there is a PPL bypass and the register is not stripped of its functionaility. There are many capabilities to the debug register, but Project Zero used it to halt the core on CPU reset and write custom instructions during the reset vector. It involved disabling KTRR on MMU protection and modifying the root page table to map ``_TEXT_exec_`` to a writable region.

2. Operation Triangulation DMA:

This exploit allow writing to any location in memory, without MMU protection. The capability comes from an MMIO register in the GPU stack. Unfortunately, while this DMA capability allows us to bypass MMU write protection, AMCC is still at play here, protecting the memory region, causing a panic when a write operation happens. It is also known that it is while AMCC protects the regions in memory from being modified, if the write is to the cache, it won't trigger any problem until a write back happens. Therefore, an idea like this code work:
* During a core sleep routine, get the reset vector to be loaded from memory to the system-level cache and patch it with the primitive. THe SLC is shared accross the entire SoC, so the CPU can access the instructions that the GPU has written without it having to first be written back to DRAM.
* From the CPU core before entering sleep, inflate the number of memory access to move the data to the core's own L1 cache. This depends on the assumption that the L1 cache isn't flushed while the core is in sleep mode.

3. Other MMIO that allows controlling core reset code execution

MMIO has a lot of interesting hidden functions, so it could be the case that there is a different MMIO register (probably PPL restricted) that can be used to automatically gain code execution when the core resets.
