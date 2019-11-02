* Suggested: https://github.com/bjorn3/inline_asm_catalogue/issues/1
* Reason: Using x86 transactional memory

# Code

https://github.com/Amanieu/parking_lot/blob/99e7e469396b53dd13d7e11fa7d0d9748cc9f08e/src/elision.rs

```rust
asm!("xacquire; lock; cmpxchgq $2, $1"
     : "={rax}" (prev), "+*m" (self)
     : "r" (new), "{rax}" (current)
     : "memory"
     : "volatile");

asm!("xrelease; lock; xaddq $2, $1"
     : "=r" (prev), "+*m" (self)
     : "0" (val.wrapping_neg())
     : "memory"
     : "volatile");
```
