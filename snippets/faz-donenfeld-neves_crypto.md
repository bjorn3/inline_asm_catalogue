* Authors: Armando Faz, Jason A. Donenfeld, Samuel Neves
* Reason: side-channel resistance required + performance critical
* Language: C
* Source: https://github.com/armfazh/rfc7748_precomputed
* License: [BSD-3-clause](https://github.com/armfazh/rfc7748_precomputed/blob/master/LICENSE)

# Code

This snippet is probably a good example of how inline assembly directives
are used in the cryptographic engineering community.

This snippet is part of a larger implementation of Curve25519 and Curve448,
two popular elliptic curves used for cryptography (specified by [RFC7748]).
This implementation has been put to use in multiple practical applications,
most notably in [wireguard][zinc].

```c
// Add two elements in GF(2^255 - 19) at `a` and `b` and put the result at `c`.
//
// Field elements are stored in radix 2^64 packed representation.  I.e. every
// element spans 4 limbs.

inline void add_EltFp25519_1w_x64(uint64_t *const c, uint64_t *const a, uint64_t *const b) {
#ifdef __ADX__
  __asm__ __volatile__(
    "mov     $38, %%eax ;"
    "xorl  %%ecx, %%ecx ;"
    "movq   (%2),  %%r8 ;"  "adcx   (%1),  %%r8 ;"
    "movq  8(%2),  %%r9 ;"  "adcx  8(%1),  %%r9 ;"
    "movq 16(%2), %%r10 ;"  "adcx 16(%1), %%r10 ;"
    "movq 24(%2), %%r11 ;"  "adcx 24(%1), %%r11 ;"
    "cmovc %%eax, %%ecx ;"
    "xorl %%eax, %%eax  ;"
    "adcx %%rcx,  %%r8  ;"
    "adcx %%rax,  %%r9  ;"  "movq  %%r9,  8(%0) ;"
    "adcx %%rax, %%r10  ;"  "movq %%r10, 16(%0) ;"
    "adcx %%rax, %%r11  ;"  "movq %%r11, 24(%0) ;"
    "mov     $38, %%ecx ;"
    "cmovc %%ecx, %%eax ;"
    "addq %%rax,  %%r8  ;"  "movq  %%r8,   (%0) ;"
  :
  : "r" (c), "r" (a), "r" (b)
  : "memory", "cc", "%rax", "%rcx", "%r8", "%r9", "%r10", "%r11"
  );
#else
  __asm__ __volatile__(
    "mov     $38, %%eax ;"
    "movq   (%2),  %%r8 ;"  "addq   (%1),  %%r8 ;"
    "movq  8(%2),  %%r9 ;"  "adcq  8(%1),  %%r9 ;"
    "movq 16(%2), %%r10 ;"  "adcq 16(%1), %%r10 ;"
    "movq 24(%2), %%r11 ;"  "adcq 24(%1), %%r11 ;"
    "mov      $0, %%ecx ;"
    "cmovc %%eax, %%ecx ;"
    "addq %%rcx,  %%r8  ;"
    "adcq    $0,  %%r9  ;"  "movq  %%r9,  8(%0) ;"
    "adcq    $0, %%r10  ;"  "movq %%r10, 16(%0) ;"
    "adcq    $0, %%r11  ;"  "movq %%r11, 24(%0) ;"
    "mov     $0, %%ecx  ;"
    "cmovc %%eax, %%ecx ;"
    "addq %%rcx,  %%r8  ;"  "movq  %%r8,   (%0) ;"
  :
  : "r" (c), "r" (a), "r" (b)
  : "memory", "cc", "%rax", "%rcx", "%r8", "%r9", "%r10", "%r11"
  );
#endif
}
```

[RFC7748]: https://tools.ietf.org/html/rfc7748
[zinc]: https://git.zx2c4.com/WireGuard/tree/src/crypto/zinc/curve25519/curve25519-x86_64.c?id=21df5a545df65ec58a73e3af6bfb67f93b70feb9
