* Author: @Amanieu
* Suggested: https://internals.rust-lang.org/t/pre-rfc-inline-assembly/6443/46
* Reason: Add section pointing to two labels

# Code

```rust
// Common code for interruptible syscalls
macro_rules! asm_interruptible_syscall {
    () => {
        r#"
            # If a signal interrupts us between 0 and 1, the signal handler
            # will rewind the PC back to 0 so that the interrupt flag check is
            # atomic.
            0:
                ldrb ${0:w}, $2
                cbnz ${0:w}, 2f
            1:
               svc #0
            2:

            # Record the range of instructions which should be atomic.
            .section interrupt_restart_list, "aw"
            .quad 0b
            .quad 1b
            .previous
        "#
    };
}

// There are other versions of this function with different numbers of
// arguments, however they all share the same asm code above.
#[inline]
pub unsafe fn interruptible_syscall3(
    interrupt_flag: &AtomicBool,
    nr: usize,
    arg0: usize,
    arg1: usize,
    arg2: usize,
) -> Interruptible<usize> {
    let result;
    let interrupted: u64;
    asm!(
        asm_interruptible_syscall!()
        : "=&r" (interrupted)
          "={x0}" (result)
        : "*m" (interrupt_flag)
          "{x8}" (nr as u64)
          "{x0}" (arg0 as u64)
          "{x1}" (arg1 as u64)
          "{x2}" (arg2 as u64)
        : "x8", "memory"
        : "volatile"
    );
    if interrupted == 0 {
        Ok(result)
    } else {
        Err(Interrupted)
    }
}
```
