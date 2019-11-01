* Reason: A lot of stack fiddling to spill all registers to the stack on an interrupt and restore them again when returning.
  There are also various segment manipulations to support 32bit x86.
  It also uses various interrupt handling instructions like `sti`, `cli` and `cli`.
  Lastly like any x86 OS it needs to read from and write to ports and change the cr3 register pointing to the page table.
* Alternatives:
    * Use the `x86-interrupt` call abi for interrupt handling
        * Doesn't support the segment manipulations.

# Source

https://gitlab.redox-os.org/redox-os/kernel/blob/ad5f3814fa665fe1c7123b02a91feeec80b0d50d/src/arch/x86_64/interrupt/syscall.rs
https://gitlab.redox-os.org/redox-os/kernel/blob/ad5f3814fa665fe1c7123b02a91feeec80b0d50d/src/arch/x86_64/interrupt/mod.rs
