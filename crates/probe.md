* Description: libprobe: Static probes for Rust
* Suggested: https://internals.rust-lang.org/t/should-we-ever-stabilize-inline-assembly/11211/33
* Reason: Create a link section pointing to a nop instruction at the current position, so that SystemTap knows where the probe point is.
* Alternatives:
    * Use a static with `#[link_section]` combined with an `asm!` invoction containing only a nop and a symbol pointing to that nop. (https://internals.rust-lang.org/t/should-we-ever-stabilize-inline-assembly/11211/37)
        * Causes problems when the `asm!` is duplicated to for example inline a function or unroll a loop. (https://internals.rust-lang.org/t/should-we-ever-stabilize-inline-assembly/11211/38)

# Code

https://github.com/cuviper/rust-libprobe/blob/431ac2999eb88e3a8ba5ee15df13557e234d9775/src/platform/systemtap.rs#L162-L185

```rust
asm!(concat!(r#"
990:    nop
        .pushsection .note.stapsdt,"?","note"
        .balign 4
        .4byte 992f-991f, 994f-993f, 3
991:    .asciz "stapsdt"
992:    .balign 4
993:    "#, $addr, r#" 990b
        "#, $addr, r#" _.stapsdt.base
        "#, $addr, r#" 0 // FIXME set semaphore address
        .asciz ""#, stringify!($provider), r#""
        .asciz ""#, stringify!($name), r#""
        .asciz ""#, $argstr, r#""
994:    .balign 4
        .popsection
.ifndef _.stapsdt.base
        .pushsection .stapsdt.base,"aG","progbits",.stapsdt.base,comdat
        .weak _.stapsdt.base
        .hidden _.stapsdt.base
_.stapsdt.base: .space 1
        .size _.stapsdt.base, 1
        .popsection
.endif
"#
    )
    : // output operands
    : // input operands
        $("nor"(($arg) as i64)),*
    : // clobbers
    : // options
        "volatile"
)
```
