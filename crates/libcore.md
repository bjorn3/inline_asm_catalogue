* Reason: Black box

# Code

https://github.com/rust-lang/rust/blob/01e5d91482e3e8fb9f55efabab760db2d50ddaff/src/libcore/hint.rs#L118

```rust
asm!("" : : "r"(&dummy));
```
