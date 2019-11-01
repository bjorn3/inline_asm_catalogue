* Reason: Various cpu specific instructions and preventing optimization
* Alternatives:
    * `bkpt`: Use `core::intrinsics::breakpoint`
    * `isb`/`dsb`/`dmb`: Use `core::arch::arm::{__isb, __dsb, __dmb}` instead.
    * Various other `asm!` invocations don't have an alternative.

# Code

https://github.com/rust-embedded/cortex-m/blob/39172497729e827a6bb48f899a1facf5ceb2558b/src/asm.rs

```rust
asm!("bkpt" :::: "volatile");

// delay
asm!("1:
      nop
      subs $0, $$1
      bne.n 1b"
     : "+r"(_n / 4 + 1)
     :
     :
     : "volatile");

asm!("nop" :::: "volatile");
unsafe { asm!("wfe" :::: "volatile");
asm!("wfi" :::: "volatile");
asm!("sev" :::: "volatile");
asm!("isb 0xF" ::: "memory" : "volatile");
asm!("dsb 0xF" ::: "memory" : "volatile");
asm!("dmb 0xF" ::: "memory" : "volatile");
```
