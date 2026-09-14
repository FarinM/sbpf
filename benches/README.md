`jit_address_translation` measures execution of verified SBPF V0 programs through the x86-64 JIT. It uses the same nightly `test::Bencher` framework as the other benchmarks, without additional dependencies.

```sh
cargo +nightly bench --locked --bench jit_address_translation -- --nocapture
```

On an Apple Silicon Mac with Rosetta installed, add `--target x86_64-apple-darwin` before `--`. These runs are diagnostics; measure on native validator hardware before drawing conclusions about validator performance.

The execution cases cover:

- Loads, register stores and immediate stores at widths 1, 2, 4 and 8 bytes.
- Readonly input, an unaligned guest address, sequential offsets, fixed stack gaps and mixed input/stack regions.
- Memory accesses mixed with arithmetic, plus an arithmetic-only control.
- Unaligned mappings, which use the original translation helpers.
- Readonly stores, out-of-bounds accesses, missing regions and accesses into a stack gap.
- Different generated-code footprints with the same number of executed loads.

Normal single-operation cases perform 16,384 accesses per invocation. The mixed-region body performs four accesses per repetition; the mixed-arithmetic body performs one access and three arithmetic instructions. Fault cases stop at their first access. Reported `ns/iter` includes VM entry/exit, register and CU-budget reset, and execution of the guest loop. It excludes assembly, verification, JIT compilation, memory allocation and correctness comparisons.

Before timing, each execution case compares the interpreter and JIT with identical starting memory and registers. It checks the expected success/failure, exact result/error, instruction count, remaining CU, final PC and backing-memory contents. The stores are idempotent; final memory and remaining CU are checked again after timing. Instruction metering, immediate sanitization, runtime-pointer encryption and randomized NOP insertion remain enabled. The fixed stack uses 4 KiB frames.

Most programs use 128 body sites. The 64-site and 1,024-site load cases vary code footprint while retaining 16,384 executed loads. The two `capacity_*` cases append unreachable instructions to vary the allocation estimate without adding executed work: 950 instructions fill the current 128 KiB pool block on 16 KiB-page hosts; 1,070 do so on 4 KiB-page hosts. The applicable boundary case uses the original helpers because there is no spare allocation capacity for shared routines. Check the printed code/allocation sizes when using a different allocator or page size. NOP randomization makes code size vary slightly between processes.

`compile_128_sites` and `compile_1024_sites` time actual repeated JIT compilation, including allocation, code emission, sealing and replacement of the old compiled program. They exclude assembly and verification. Keep compilation results separate from execution results.

For a stock-versus-patched comparison, use identical benchmark source, compiler, target, features and build settings in both checkouts. For this branch, the unmodified runtime base is `1be400928a046729cf55595c033933553cf89e06`:

```sh
git worktree add --detach ../sbpf-bench-stock 1be400928a046729cf55595c033933553cf89e06
cp benches/jit_address_translation.rs ../sbpf-bench-stock/benches/

cargo +nightly bench --locked --bench jit_address_translation -- --nocapture
cargo +nightly bench --locked --manifest-path ../sbpf-bench-stock/Cargo.toml \
  --bench jit_address_translation -- --nocapture
```

Pin one nightly date for both builds and record both revisions. Build both targets with `--no-run` before measuring, then alternate executions of the benchmark binaries in separate processes. Repeat runs to account for randomized JIT layouts and host noise. Individual cases can be selected by appending their name to the benchmark command, for example `aligned_load_u64`.

As an additional diagnostic, run the patched binary in a separate process with `SBPF_INLINE_OFF=0`. Presence of that variable disables the fast mapping view; even the value `0` disables it. This retains generated guards and the changed runtime metadata, so it is not a substitute for an unmodified baseline.
