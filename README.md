# RISC-V Vector Function ABI PoC — VLA Mangling & libmvec

## 1. Background: GCC Middle-End vs. RISC-V ABI Naming

The RISC-V psABI specifies vector math symbols in the form:

    _ZGVr<lmul>N<simdlen>v_<name>

However, GCC's generic `simd_clone` middle-end was originally designed for x86 and AArch64, where the architecture token (`<isa>`) is a **single character** (e.g., `n` for AArch64 NEON, `s` for SVE). It does **not** natively carry an LMUL field, nor does it reserve a channel to pass multi-character LMUL values (`f8`, `f4`, `f2`, `1`, `2`, `4`, `8`) to the mangling hook.

Therefore, a direct mapping from GCC's internal `_ZGV<x>N<y>v` to the RISC-V ABI `_ZGVr<lmul>N<simdlen>v` is not possible without extending the middle-end data structures.

---

## 2. Temporary Implementation (PoC)

To demonstrate that the proposed ABI is implementable **today**, this PoC uses a **backend workaround**:

- **Extension**: We hook `riscv_simd_clone_adjust` in the RISC-V backend to **rewrite** the assembler name after the middle-end generates it.
  - The middle-end still emits its default name (e.g., `exp.simdclone.2`).
  - The backend strips the `.simdclone.N` suffix and prepends the correct `_ZGVr<lmul>Nxv` prefix.
  - LMUL is decoded from `vecsize_mangle` (`c/b/a/1/2/4/8` → `f8/f4/f2/1/2/4/8`).

This is **intentionally a temporary measure**. The proper long-term fix requires upstream GCC changes, such as:
- Adding an LMUL field to `struct cgraph_simd_clone`, **or**
- Introducing a `TARGET_SIMD_CLONE_MANGLE` hook that runs earlier and allows backends to fully control the mangling string.

Once the ABI is finalized, we will submit a cleanup patch to GCC upstream to remove this workaround.

---

## 3. glibc libmvec Implementation

This PoC includes a RISC-V RVV vector math library (`libmvec`) providing `exp` and `log` for `double`:

- **Symbols exported** (verified in `libmvec.so.1`):
  - `_ZGVr1Nxv_exp`, `_ZGVr2Nxv_exp`, `_ZGVr4Nxv_exp`, `_ZGVr8Nxv_exp`
  - `_ZGVr1Nxv_log`, `_ZGVr2Nxv_log`, `_ZGVr4Nxv_log`, `_ZGVr8Nxv_log`

---

## 4. Verification

### 4.1 Symbol Generation
```text
$ riscv64-unknown-linux-gnu-nm libmvec.so.1 | grep ZGV
00000000000006d8 T _ZGVr1Nxv_exp
0000000000000dba T _ZGVr1Nxv_log
00000000000007e2 T _ZGVr2Nxv_exp
000000000000103a T _ZGVr2Nxv_log
00000000000008ec T _ZGVr4Nxv_exp
00000000000012ca T _ZGVr4Nxv_log
0000000000000a9c T _ZGVr8Nxv_exp
0000000000001556 T _ZGVr8Nxv_log

The vectorized libmvec was linked into SPEC CPU 2017 527.cam4_r and executed on a Sophgo SG2044 RISC-V board.
Hotspot profiling confirms _ZGVr2Nxv_exp and _ZGVr2Nxv_log are actively called at runtime.
