# RCompiler

## Overview
RCompiler is a compiler for a Rust-like teaching language. The current main
pipeline lowers Rx source code to a custom LLVM-like IR, runs a series of IR
optimizations, selects RV64 instructions, performs register allocation, and
prints RV64 assembly.

The compiler follows the online judge interface:

- `stdin`: Rx source code
- `stdout`: generated user assembly
- `stderr`: runtime support assembly from `builtin.s`

## Build

### Prerequisites
- CMake >= 3.10
- A C++17 compiler, such as GCC or Clang
- Optional for local RV64 validation: `riscv64-linux-gnu-gcc` and `qemu-riscv64`
- Optional for the existing semantic-2 benchmark script: REIMU

### Steps
```bash
cmake -S . -B build
cmake --build build
```

The Makefile wraps the same flow:

```bash
make build
```

## Usage

Compile one Rx program to assembly:

```bash
make run < path/to/input.rx > test.s 2> test_builtin.s
```

The generated program should be assembled together with the builtin runtime:

```bash
riscv64-linux-gnu-gcc -static test.s test_builtin.s -o test
qemu-riscv64 ./test
```

## Pipeline

The current compiler path is:

```text
source
  -> Simplifier
  -> Lexer
  -> Parser
  -> Semantic checks
  -> ConstEvaluator
  -> IRBuilder / CodeGenerator
  -> IR optimization pipeline
  -> Instruction selection
  -> Peephole optimization
  -> Assembly dead-code elimination
  -> Linear-scan register allocation
  -> Post-register-allocation peephole optimization
  -> RV64 assembly printer
```

## Source Layout

### Frontend
- `src/simplifier`: pre-processes the input stream before lexing.
- `src/lexer`: tokenizes the simplified source.
- `src/parser`: recursive descent parser with a Pratt parser for expressions.
- `src/ast`: AST node definitions and visitors.
- `src/semantic`: scope construction, name resolution, type checking, borrow
  checking, and constant evaluation.

### IR
- `src/ir`: custom LLVM-like IR node definitions and IR construction.
- The IR is organized around `IRRoot`, `IRFunction`, `IRBlock`, and instruction
  nodes such as `IRAlloca`, `IRLoad`, `IRStore`, `IRGetptr`, `IRBinaryop`,
  `IRBr`, `IRReturn`, `IRCall`, `IRPhi`, casts, and memory intrinsics.
- The IR keeps explicit type and width information, including 32-bit integer
  semantics, 64-bit `usize/isize` semantics, pointer storage, arrays, structs,
  and method-related ABI details.
- LLVM-style text printing exists for debugging, but the current production
  path lowers the in-memory IR directly to RV64 assembly.

### Optimization
- `src/optimize`: IR-level optimization passes.
- The optimization notes and next-step analysis are documented in
  [`doc/optimize.md`](doc/optimize.md).
- Recent long-parameter ABI debugging notes are in
  [`doc/long_long_param_debug_notes.md`](doc/long_long_param_debug_notes.md).

### Backend
- `src/linearScan`: active backend path, including instruction selection,
  peephole optimization, linear-scan register allocation, and assembly output.
- `src/codegen`: earlier code generation components.
- `builtin.s`: builtin runtime routines emitted on `stderr`.

## IR Optimization Pipeline

The current pass order in `main.cpp` is:

```text
SROA
Mem2Reg
DominantTree
SCCP
FunctionInline
SCCP
ConstantFold
CFGClean
ConstantFold
CFGClean
DCE
MemoryForward
ConstantFold
CFGClean
StrengthReduction
LocalCSE
MemoryForward
ConstantFold
CFGClean
LICM
DCE
```

Implemented optimization areas include:

- scalar replacement of small aggregate locals (`SROA`)
- stack-slot promotion and simple load/store propagation (`Mem2Reg`)
- dominator-based SSA/phi handling (`DominantTree`)
- sparse conditional constant propagation (`SCCP`)
- conservative function inlining with a cost model
- constant folding and algebraic simplification
- CFG cleanup and dead-code elimination
- alias-aware memory forwarding and dominated load reuse
- local and dominator-tree CSE/GVN for pure expressions, casts, `getptr`, and
  conservative loads
- loop-invariant code motion for safe scalar expressions and address chains
- strength reduction for clearly profitable power-of-two arithmetic

The optimizer is intentionally conservative around memory aliasing, calls,
`getInt`, `memcpy`, `memset`, loop-carried loads, and 64-bit `usize/isize`
semantics.

## Optimization Roadmap

The current optimization document treats `semantic-1`, `semantic-2`, and
`long-param` as correctness regression suites. Performance decisions are mainly
driven by hidden OJ optimization cases.

Short-term priorities from `doc/optimize.md`:

- record OJ deltas for every optimization commit
- avoid broad new passes that duplicate existing CSE, forwarding, or LICM work
- tune regressions by case, especially around SROA pointer fields, spill
  cleanup, inlining, LICM, and register pressure
- keep RV64 output within the currently accepted instruction set

Potential later work:

- more aggressive loop strength reduction
- multi-block function inlining
- `Global2Local` for safe scalar globals used by a single function
- further register-pressure-aware optimization heuristics

## Testing

The public testcase repository is
[RCompiler-Testcases](https://github.com/peterzheng98/RCompiler-Testcases).

Build first:

```bash
cmake --build build
```

Run the existing semantic-2 assembly benchmark script:

```bash
./asm_test.sh
```

The script supports ranges and verbose output:

```bash
./asm_test.sh 23
./asm_test.sh 10 20
./asm_test.sh -v 23
```

For RV64 correctness, prefer assembling with `riscv64-linux-gnu-gcc -static`
and running with `qemu-riscv64`, especially for long-parameter and hidden-OJ
style cases. The optimization notes currently use:

```bash
bash /tmp/qt2.sh RCompiler-Testcases/long-param/src
bash /tmp/qemu_test.sh
```

The historical IR test script is still present:

```bash
./ir_test.sh
```

It expects the LLVM-IR runtime file `builtin.ll`, while the current main
compiler path emits RV64 assembly plus `builtin.s`.

## RxLanguage Reference
The project is based on
[RLanguage Reference](https://github.com/peterzheng98/RCompiler-Spec/).

## Acknowledgements
- Thanks to TAs for the RLanguage Reference and their guidance.
- Thanks to my boyfriend for his support and encouragement throughout the project.
- Thanks to my roommate for her patience and understanding during late-night coding sessions.
