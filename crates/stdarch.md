* Suggested: https://github.com/bjorn3/inline_asm_catalogue/issues/2
* Reason: Implementing platform intrinsics
* Alternatives:
    * Add compiler intrinsics
        * There is a fixed amount of platform intrinsics, so simply matching on the asm from a
          codegen backend should be as easy when inline asm is not supported.

# Code

https://github.com/rust-lang/stdarch/blob/7e498dfcd689ee6be6956a65b1d71762b51825fa/crates/std_detect/src/detect/os/aarch64.rs

```rust
// arm
asm!("mrs $0, ID_<REG_NAME>" : "=r"(<reg_name>));
```

https://github.com/rust-lang/stdarch/blob/3c2984e7177b1ee08f706ee4e48ebe30808ed68d/crates/assert-instr-macro/src/lib.rs#L142

```rust
// Make sure that the shim is not removed by leaking it to unknown
// code:
unsafe { asm!("" : : "r"(#shim_name as usize) : "memory" : "volatile") };
```

https://github.com/rust-lang/stdarch/blob/32af6780080ade6525bf52c53d0bb7e172f70e7a/crates/core_arch/src/x86/eflags.rs#L16

```rust
// x86

// deprecated, because the compiler is allowed to insert arbitrary instructions manipulating eflags
// inbetween calls of __readeflags and __ writeeflags, use inline asm instead
pub unsafe fn __readeflags() -> u32 {
    let eflags: u32;
    asm!("pushfd; popl $0" : "=r"(eflags) : : : "volatile");
    eflags
}
```

https://github.com/rust-lang/stdarch/blob/e8ed0838c972b784dcf2e830665bc92c5ee47c43/crates/core_arch/src/acle/barrier/cp15.rs

```rust
// arm

// dmb
asm!("mcr p15, 0, r0, c7, c10, 5" : : : "memory" : "volatile")
// dsb
asm!("mcr p15, 0, r0, c7, c10, 4" : : : "memory" : "volatile")
// isb
asm!("mcr p15, 0, r0, c7, c5, 4" : : : "memory" : "volatile")
```

https://github.com/rust-lang/stdarch/blob/e8ed0838c972b784dcf2e830665bc92c5ee47c43/crates/core_arch/src/acle/registers/mod.rs

```rust
// arm
asm!(concat!("mrs $0,", stringify!($R)) : "=r"(r) : : : "volatile");
asm!(concat!("msr ", stringify!($R), ",$0") : : "r"(value) : : "volatile");
```

https://github.com/rust-lang/stdarch/blob/ef6b0690192f1cfc753af698695c2ecde0c7b991/crates/core_arch/src/x86/bt.rs

```rust
// x86
asm!("btl $2, $1\n\tsetc ${0:b}"
     : "=r"(r)
     : "*m"(p), "r"(b)
     : "cc", "memory");

asm!("btsl $2, $1\n\tsetc ${0:b}"
     : "=r"(r), "+*m"(p)
     : "r"(b)
     : "cc", "memory");

asm!("btrl $2, $1\n\tsetc ${0:b}"
     : "=r"(r), "+*m"(p)
     : "r"(b)
     : "cc", "memory");

asm!("btcl $2, $1\n\tsetc ${0:b}"
     : "=r"(r), "+*m"(p)
     : "r"(b)
     : "cc", "memory");
```

https://github.com/rust-lang/stdarch/blob/ef6b0690192f1cfc753af698695c2ecde0c7b991/crates/core_arch/src/arm/armclang.rs#L51-L63

```rust
// aarch64
asm!(concat!("BRK ", stringify!($imm8)) : : : : "volatile")

// arm32 / thumb32
asm!(concat!("BKPT ", stringify!($imm8)) : : : : "volatile")
```

https://github.com/rust-lang/stdarch/blob/ef6b0690192f1cfc753af698695c2ecde0c7b991/crates/core_arch/src/x86/xsave.rs

Several `XCR` related x86 platform intrinsics.

https://github.com/rust-lang/stdarch/blob/4791ba85e7645c02146dd416288480943670d1ca/crates/core_arch/src/x86/cpuid.rs

```rust
// x86
asm!("cpuid"
     : "={eax}"(eax), "={ebx}"(ebx), "={ecx}"(ecx), "={edx}"(edx)
     : "{eax}"(leaf), "{ecx}"(sub_leaf)
     : :);

// Detect cpuid existence (only used when SSE is not known to be supported)
asm!(r#"
     # Read eflags into $0 and copy it into $1:
     pushfd
     pop     $0
     mov     $1, $0
     # Flip 21st bit of $0.
     xor     $0, 0x200000
     # Set eflags to the value of $0
     #
     # Bit 21st can only be modified if cpuid is available
     push    $0
     popfd          # A
     # Read eflags into $0:
     pushfd         # B
     pop     $0
     # xor with the original eflags sets the bits that
     # have been modified:
     xor     $0, $1
     "#
     : "=r"(result), "=r"(_temp)
     :
     : "cc", "memory"
     : "intel");
```
