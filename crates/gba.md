* Suggested: https://github.com/bjorn3/inline_asm_catalogue/issues/4
* Reason: GBA BIOS calls
* Alternatives:
    * Extern assembly
        * More overhead on not very fast processor

# Source

https://github.com/rust-console/gba/blob/35e9857dcc895053cad2ac7f02b984c0c4e97364/src/bios.rs#L256-L261

```rust
asm!(/* ASM */ "swi 0x06"
    :/* OUT */ "={r0}"(div_out), "={r1}"(rem_out)
    :/* INP */ "{r0}"(numerator), "{r1}"(denominator)
    :/* CLO */ "r3"
    :/* OPT */
);
```

And many more `swi` calls.

Wants to make https://github.com/rust-console/gba/blob/35e9857dcc895053cad2ac7f02b984c0c4e97364/crt0.s#L1 inline.
