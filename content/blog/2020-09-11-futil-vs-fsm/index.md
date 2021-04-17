+++
title = "miniHLS: Construct Statically-Timed FSMs"
extra.author = "Rachit Nigam"
extra.latex = true
extra.bio = """
  [Rachit Nigam](https://rachitnigam.com) is a second year PhD student interested in
  programming languages & computer architecture. In his free time, he
  [subtweets](https://twitter.com/notypes/status/1170037148290080771) his advisor and [annoys tenured professors](https://twitter.com/natefoster/status/1074401015565291520).
"""
+++

The blog post describes [miniHLS][mhls], a tool for constructing statically-timed
hardware designs and uses it to evaluate the FSMs generated from a state-of-the-art
hardware compiler infrastructure.

### Generating Hardware Accelerators

Hardware acceleration in the form of specialized processors has become
increasingly common---from the graphics processing units (GPUs) that
accelerate everything from shader programs to linear algebra, to the tens
of new specialized AI chips that promise to run your favorite machine learning
models an order of magnitude faster.

Field programmable gate arrays (FPGAs) are a kind of reconfigurable
architecture that can be used to simulate hardware designs.
More recently, they have been become widely available on public clouds such
as AWS and Azure, accelerating many kinds of computations.
For an in-depth look into reconfigurable fabrics for accelerator design,
see this [blog post][reconf-future].

While incredibly flexible, FPGAs are a pain to program. The tools and languages
repurpose preexisting hardware design languages such as Verilog and VHDL.
While perfectly usable for precise hardware design, Verilog-like languages
require specification of every gate, wire, and clock cycle.
Building and iterating accelerators with them is a chore; imagine trying to
implement a matrix-multiply kernel by specifying every single wire (then
try imagining changing one design parameter requiring a pervasive rewrite).

Because of this verbose and tedious programming model, researchers and
practitioners have tried to design higher level programming models, either
by re-purposing legacy languages like C/C++ or building [entirely][spatial]
[new][aetherling] [languages][dahlia].

However, building such languages requires a lot of infrastructure---designing
the frontend, picking an architectural template, optimizing the generated
hardware, and generating code that can be run on an FPGA.
Instead of redesigning everything from scratch, the [FuTIL][] infrastructure hopes
to create "LLVM for hardware design" by allowing frontends to target a small
but expressive language and by automatically handling the "boring" optimization
and generation problems.

### Fused Temporal Intermediate Language (FuTIL)

FuTIL is an intermediate language (IL) and a compiler infrastructure for building
accelerator generators.
This section provides a high-level overview for the IL.

Before understanding the role of FuTIL, we need to understand how modern
compilers are constructed.
Every major compiler uses an intermediate language to represent the input
programs, analyze them, and optimize them.
As compilers have grown in complexity, they have started using many different
ILs to represent different *abstraction levels*.
For example, the [Rust][] compiler uses three different IL to analyze and
optimize Rust programs: HIR for type checking the program, MIR to implement
Rust-specific optimizations, and LLVM IR to implement generic optimizations.
This IR-based organization structure allows Rust to reuse the optimizations in
the LLVM compiler toolchain while also implementing specific optimizations
using MIR.

FuTIL is a generic mid-level IR for accelerator generators. It sits in the
middle of the high-level input programs that imply an abstract architecture
and the low-level Verilog programs that instantiate a concrete architecture.
By concretizing some aspects of the final architecture, while keeping others
abstract, FuTIL is able to implement optimizations that can be shared across
many different toolchains.

**FuTIL programs**. FuTIL programs are constructed using *components* which
roughly correspond to functions in the software world, and modules in the
hardware world.
Each component has three sections:
(1) *cells* which describes all hardware units used by the component,
(2) *wires* which describes connections between the hardware units, and
(3) *control* which describes the execution schedule of the program.
We elide the specifics of the first two sections. For a detailed overview,
please take a look at the [FuTIL documentation][futil].

The *control* program is the novel aspect of FuTIL's design. A control program
looks like:
```C
seq {
  init;
  while lt.out with cmp {
    incr;
  }
}
```
Here, `seq` and `while` are two *control-flow operators*. They describe the
execution flow of the program. `init`, `incr`, and `cmp` are dataflow graphs
are defined in the *wires* section of the component. At a high-level, they
define an "action". For example, `init` might define the connections required
to initialize a register, `incr` for incrementing the value in the register,
and `cmp` defines the connections to compare the current value in the register.
Using these groups, the control program precisely describes the control flow.

The key insight in the design of FuTIL is that by explicitly representing
the control flow of a program, the mid-level IR can implement several
optimizations that would otherwise not be possible in a Verilog designs.
For example, if the control program says that two dataflow graphs execute
in sequence, we know that they can share the same hardware resource and that
they will never conflict.


### The Dahlia-to-FuTIL Compiler

[Dahlia][] is an imperative, array-based programming language that is used
to generate hardware accelerators.
Dahlia programs look very similar to simple C/C++ programs. For example,
the following implements a dot-product kernel:

```C
decl a: float[8];
decl b: float[8];
let sum = 0;
for (let i = 0 .. 8) {
  sum += a[i] * b[i];
}
```

The first two lines define two memories `a` and `b` and the `for` loop iterates
over their elements and accumulates the values into a register `sum`.

The Dahlia compiler compiles Dahlia programs into FuTIL in three steps:
(1) *type checking*, which establishes that the generated design will have
predictable area-latency trade-offs,
(2) *lowering*, which removes complex control structures like `for` loops, loop
unrolling, and memory banking, and
(3) *code generation*, which instantiates hardware components and generates
FuTIL programs.

The FuTIL backend for Dahlia is fairly mature and has been tested using the
linear algebra kernels from the [Polybench][] benchmark suite. The backend
currently does not support signed, fixed-point, or floating-point arithmetic. It
also does not support C-style `struct` definitions.

**Motivating the low-level comparison**. A FuTIL-based compiler has to represent
the semantics of Dahlia programs using FuTIL constructs and rely on the FuTIL
compiler to optimize programs. In this case, it is reasonable to ask how
much performance is lost during the translation and if there any optimizations
that can be implemented in the FuTIL compiler to recover the lost performance.
In order to support such a comparison, I implemented a simple compiler that
transforms FSM descriptions into hardware designs.

### miniHLS: Statically Timed FSM Generation

miniHLS is a very simple tool: It takes definitions of FSMs and transforms
them in hardware designs.

Following is an example FSM definition:
```
(start = 0, done = [6]) {
  0: { a <= 4; b <= 12; next(1) }
  1: { _cond <= b != 0; next(2) }
  2: { next(_cond, 3, 6) }
  3: { tmp <= b; next(4) }
  4: { b <= a mod b; next(5) }
  5: { a <= tmp; next(1) }
  6: { b <= a; done }
}
```

Each FSM consists of three things:
(1) the start state of the FSM,
(2) the done states of the FSM, and
(3) actions performed in each state.
The first two are straightforward: We simply specify the number of the corresponding
states.

Action definitions are the heart of the FSM language. Each action consists of
updates to *registers* using some constant followed by the name of the next
state. Each action takes exactly one cycle providing exact control over the
execution of the program.

This representation has several consequences. First, multi-cycle computations
need to be explicitly staged and spread across different actions.
Second, conditional state transitions are explicitly specified using the `next`
function which uses a 1-bit register to decide which transition to take.
Third, the designs can be analyzed to get the precise latency required to
perform a computation.
All of these design decisions make the FSM description language easy to compile
to Verilog and get the most performance from the design.
However, this also make FSM designs non-composable---any changes fundamentally
require changing the latency of the design and therefore is a pervasive change.

In the next section, we analyze designs generated from the FSM language and
the Dahlia-to-FuTIL compiler.

### FuTIL vs miniHLS

In order to understand the performance implications of using FuTIL as a compiler
backend, we'll compare the generated designs to functionally equivalent ones
from the miniHLS compiler. First, however, we acknowledge that this comparison
is not apples-to-apples: The FuTIL designs are easy to generate from Dahlia
compiler, whereas generating FSM designs from Dahlia would require careful
analysis of the timing behavior of programs.

Since are goal is to evaluate the efficacy of FuTIL's FSM generation, we chose
to implement the Euclidean algorithm, whose main computation is a loop body.

**GCD**. Implementation of the Euclidean algorithm in both Dahlia (compiled to
FuTIL) and as an FSM. The steps column shows the number of steps taken by the
algorithm. We compare the following designs:
1. **FuTIL-real-mod**: The default design generated by the Dahlia compiler that
implements the modulus operator in Verilog. Every other design uses the 1-cycle
behavioral primitive `%`.
2. **FuTIL**: FuTIL design generated with the `%` primitive.
3. **FSM**: Naive implementation that implements a swap of variables `a` and `b`
using a temporary.
4. **FSM-opt**: Optimized FSM that uses the register read-write semantics to
swap values of `a` and `b` in one cycle.


| Inputs      | Steps | FSM | FSM-opt | FuTIL | FuTIL-real-mod |
|-------------|-------|-----|---------|-------|----------------|
| (4, 12)     | 1     | 17  | 12      | 16    | 41             |
| (28, 1124)  | 2     | 21  | 15      | 20    | 93             |
| (28, 334)   | 3     | 26  | 18      | 24    | 121            |
| (240, 1070) | 4     | 31  | 21      | 28    | 149            |
| (25, 2261)  | 5     | 36  | 24      | 32    | 177            |
| (816, 2260) | 6     | 41  | 27      | 36    | 205            |

Surprisingly, the FuTIL generated design beats the design generated by the
naive FSM implementation. The next section contains a detailed analysis of the
timing differences in the two designs. Furthermore, as expected, the optimized
design beats the naive FSM and FuTIL by recognizing the swap pattern in the
kernel.

**Timing Analysis**.

The body of the GCD loop in the FSM language is implemented as:
```
  1: { _cond <= b != 0; next(2) }
  2: { next(_cond, 3, 6) }
  3: { tmp <= b; next(4) }
  4: { b <= a mod b; next(5) }
  5: { a <= tmp; next(1) }
```

State $1$ computes the loop conditional value and saves it into a register.
The value of this register is used to decide whether to execute the loop or
exit it. The FSM language current has a syntactic restriction that requires
the condition in `next` to be a register. If we remove this, the loops have
the exact same timing behavior.

The speedup of the designs in `FSM-opt` can be attributed directly to the
single cycle swap which reduces the number of cycles spent in the body from
$3$ to $1$:

```
  1: { _cond <= b != 0; next(2) }
  2: { next(_cond, 3, 6) }
  3: { b <= a mod b; a <= b; next(4) }
```

At the time of writing, it is unclear if such a swap optimization can be
directly implemented in FuTIL since it requires analyzing program paths and
preserving their functionality.

### Conclusion

We presented the design and implementation of `miniHLS`, a tool for constructing
precise statically timed FSMs and used it to evaluate the timing behavior of
a state of the art compiler infrastructure.
The short term utility of miniHLS is allowing compiler developers to quickly
prototype microbenchmark FSMs and use it to come up with design optimizations.
In the longer term, we hope that miniHLS can be used as a compiler target from
high-level languages and simplify the design iteration process even more.

[reconf-future]: https://rachitnigam.com/post/reconf-future/
[futil]: https://github.com/cucapra/futil/
[spatial]: https://spatial-lang.org/
[aetherling]: https://aetherling.org/
[dahlia]: https://capra.cs.cornell.edu/dahlia/
[rust]: https://www.rust-lang.org/
[polybench]: http://web.cse.ohio-state.edu/~pouchet.2/software/polybench/
[mhls]: https://github.com/rachitnigam/minihls