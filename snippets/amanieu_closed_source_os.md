* Author: @Amanieu
* Suggested: https://internals.rust-lang.org/t/pre-rfc-inline-assembly/6443/46 and
  https://github.com/bjorn3/inline_asm_catalogue/issues/3
* Reason: several

# Code

Reason: Add section pointing to two labels

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

Reason: syscall

```rust
// thread creation
asm!(
    r#"
        svc #0
        cbnz x0, 0f

        mov x29, #0
        mov x30, #0
        mov x0, sp
        br ${1}

        0:
    "#
    : "={x0}" (result)
    : "r" (func),
        "{x0}" (flags),
        "{x1}" (stack),
        "{x2}" (ptid),
        "{x3}" (tls),
        "{x4}" (ctid),
        "{x8}" (linux::__NR_clone)
    : "memory"
    : "volatile"
);

// thread exit + free stack
asm!(
    r#"
        svc #0

        0:
        mov x8, ${1}
        mov x0, ${2}
        svc #0

        # At this point we don't have a stack anymore, so there really
        # isn't anything we can do except loop back and try again.
        b 0b
    "#
    :
    : "N" (linux::__NR_exit),
        "r" (status),
        "{x0}" (base),
        "{x1}" (len),
        "{x8}" (linux::__NR_munmap)
    : "memory"
    : "volatile"
);
```

Reason: change stack pointer

```rust
// switch stack
asm!(
    r#"
        mov sp, ${0}
        mov x29, #0
        mov x30, #0
        br x0
    "#
    :
    : "r" (initial_sp),
        "{x0}" (func)
    :
    : "volatile"
);
```

Reason: set TLS register

```rust
asm!("msr tpidr_el0, $0" :: "r" (tls_base) : "memory" : "volatile");
```

Reason: misc

```rust
#[naked]
#[no_mangle]
pub unsafe fn _start() -> ! {
    asm!(
        r#"
            # Calculate the load offset by taking the difference between a
            # PC-relative address and a non-relocated address in memory.
            adrp x1, 0f
            ldr x2, [x1, #:lo12:0f]
            add x1, x1, #:lo12:0f
            sub x1, x1, x2

            mov x29, #0
            mov x30, #0
            mov x0, sp
            b ${0}

            .data
            .balign 8
          0:
            .quad 0b
            .previous
        "#
        :
        : "s" (init as usize)
        :
        : "volatile"
    );
    hint::unreachable_unchecked();
}
```
