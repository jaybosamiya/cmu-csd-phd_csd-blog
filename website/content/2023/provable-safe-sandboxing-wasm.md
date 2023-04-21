+++
title = "Provably-Safe Sandboxing with WebAssembly"
# TODO: Update before final PR is made
date = 2023-04-19

[taxonomies]
areas = ["Security"]
tags = ["webassembly", "sandbox", "formal-methods", "distinguished-paper-award", "internet-defense-prize"]

[extra]
author = {name = "Jay Bosamiya", url = "https://www.jaybosamiya.com/" }
# TODO: Update committee list
committee = [
    {name = "The Man, The Myth, The Legend, Harry Q. Bovik", url = "http://www.cs.cmu.edu/~bovik/"},
    "TODO: Update this with actual writing skills committee members",
]
+++

> What if you could run untrusted code and still be able to sleep at night, safe and sound?
<p></p>

Disclaimer: our award-winning work [[1]](#references) can only calm your unsafe-software related fears; we recommend complementing this by additionally checking for monsters under your bed, and leaving a night light on, for any fears of things that go bump in the night.

<figure><a name="fig1"></a><br>

![A block diagram, representing intra-process sandboxing. Multiple sandboxes are shown inside a single host process, each of which interact via an API with the runtime. The runtime itself interacts with the kernel via syscalls. Multiple sandboxes can run within a single process, and multiple processes can run on the same OS kernel.](./intra-process-sandboxing.svg)

<figcaption>Figure 1: Intra-process sandboxing</figcaption>

<br></figure>

Whether you want to include third party libraries in your code, support software plugins, use a smart content delivery network, or just browse the Web, you might need to execute untrusted code, which creates a risk that it will compromise its environment. Intra-process software sandboxing ([Figure 1](#fig1)), such as with Software Fault Isolation (SFI), is a useful primitive that allows for safe and lightweight execution of such untrusted code in the same process as its environment. Unfortunately, despite being a well-studied technique with a rich and long history, previous efforts to deploy it in production have failed, due to technical and marketplace hurdles.

We observe that [WebAssembly](https://webassembly.org/) (Wasm) is ideally positioned to provide this crucial primitive and support such applications, since Wasm promises both safety _and_ performance, while serving as a compiler target for many high-level languages. As a virtual architecture designed with sandboxing in mind, it has clean, succinct, and well-defined semantics, allowing for safe execution of high-performance code on the Web. However, this same design can also benefit non-Web applications, since the Wasm standard explicitly separates the core Wasm language from the specific API provided to each Wasm module by the runtime or other modules. For example, instead of offering a Web-oriented API, (say) for manipulating the DOM, many runtimes offer the WebAssembly System Interface (WASI) API to run Wasm beyond the Web. All of this has made Wasm a popular target for compilers, from languages like C, C++, Rust, Java, Go, C#, PHP, Python, TypeScript, Zig, Kotlin, and more. Thus, a single compiler _from_ Wasm to executable code is sufficient to immediately support sandboxed code execution for all such languages. This makes Wasm an attractive narrow waist to provide high-performance lightweight sandboxing.

However, Wasm's safety guarantees are only as strong as the implementation that enforces them. While Wasm might seem to immediately provide sandboxing, note that the actual implementation of the compiler from Wasm is a critical part of the trusted computing base (TCB) for that guarantee. In particular, any bug in the compiler could threaten the sandboxing protections, and indeed such bugs have been found in existing runtimes, and would lead to arbitrary code execution by an adversary. For example, using carefully crafted Wasm modules, an attacker could achieve a memory-out-of-bounds read in Safari/WebKit using a logic bug (CVE-2018-4222), memory corruption in Chrome/V8 using an integer overflow bug (CVE-2018-6092), an arbitrary memory read in Chrome/V8 using a parsing bug (CVE-2017-5088), arbitrary code execution in Safari/WebKit using an integer overflow bug (CVE-2021-30734); a sandbox escape in both Lucet [[6]](#references) and Wasmtime [[7]](#references) using an optimization bug (CVE-2021-32629); and many others. A plausible explanation for such disastrous sandbox-compromising bugs, even in code designed with sandboxing as an explicit focus, is that the correct, let alone secure, implementation of high-performance compilers is difficult and remains an active area of research, despite decades of work.

Upon reviewing the design space for executing Wasm code, we identified a crucial gap: Wasm implementations that provide _both_ strong security and high performance. In our work, we thus propose, explore, and implement two distinct techniques, with varying performance and development complexity, which guarantee safe sandboxing using provably-safe compilers. The first draws on traditional formal methods to produce mathematical, machine-checked proofs of safety. The second carefully embeds Wasm semantics in safe Rust code such that the Rust compiler can emit safe executable code with good performance. We describe each of these techniques in the upcoming sections, but additionally refer the interested reader to our paper [[1]](#references) for further details.

## vWasm: A Formally Verified Sandboxing Compiler

The first of our techniques, implemented as an open-source compiler, vWasm [[2]](#references), achieves provably-safe sandboxing via formal verification. Formal verification of software consists of writing a formal (mathematical) statement of the property we wish to prove about the software, and then writing a formal proof that shows that this statement is true, for our software. The proof is machine-checked and thus provides the highest degree of assurance in its correctness. In contrast to techniques such as software testing, fuzzing, and manual reviews, formal verification is able to reason about all execution paths, thereby accounting for any possible input. This means that behaviors like buffer overflows, use-after-frees, etc. are completely ruled out. We describe vWasm's top-level property, as well as our proof strategy, shortly.

Our choice of verification tool, F\* [[4]](#references), is a general-purpose functional programming language with effects, built for formal verification. Syntactically, it is closest to languages from the ML family (such as OCaml, F#, or SML). It has the full expressive power of dependent types, and has proof automation backed by Z3, an SMT solver. Code written in F\* can be extracted to multiple languages, and for vWasm, we use F\*'s OCaml extraction. Proofs are written within vWasm as a combination of pre-/post-conditions, extrinsic lemmas, and intrinsic dependently-typed values. Also, to aid in proof management, we regularly use F*'s layered effects.

vWasm is implemented as a compiler from Wasm to x86-64 (abbreviated as x64 henceforth), but it is designed to keep most of its code and proofs generic with respect to the target architecture. Here, we describe the process of compiling to x64, but the techniques generalize in a straightforward way to other architectures such as ARM. In compiling from Wasm to x64, there are three important conceptual stages: (i) a frontend which compiles Wasm to an architecture-parametric intermediate representation (IR), (ii) a sandboxing pass which acts upon the architecture-parametric IR, and (iii) a printer which outputs the x64 assembly code.

The frontend for the compiler is both untrusted and unverified. This means that one neither needs to trust its correctness for the overall theorem statement to be true, nor does one need to write proofs about it. Note that this is in stark contrast with traditional compiler verification, such as with CompCert [[5]](#references), where any stage of the compilation must either be trusted or verified. This means that we are free to use any compiler technology for the compiler's frontend, including arbitrarily complicated optimizations, as long as it outputs code within our architecture-parametric IR. Since compiler optimization is orthogonal to our primary goal, for vWasm's frontend, we implemented only a simple register allocator and a basic peep-hole optimizer. We leave other optimizations for future work.

On the other end of the compilation pipeline is the x64 assembly printer, which is trusted to be correct. This means it is included in vWasm's overall TCB, but we note that the printer is largely a straightforward one-to-one translation of our IR to strings, making it fairly simple to audit.

Finally, the sandboxing pass, which lies between the above two, is untrusted but verified to be correct. We define this formally below, but informally, this means that the sandboxing code has been proven (and the proof mechanically checked) to produce safely sandboxed code, given any input. Within the sandboxing pass, all accesses (reads or writes) into the Wasm module's linear memory, indirect function call table, imports, globals, etc. are proven (sometimes after suitable transformations) to be safe. To prove sandbox safety, we additionally prove that the sandboxing pass also guarantees (a restricted form of) Control-Flow Integrity (CFI) that ensures that any checks performed for sandboxing cannot be bypassed, and thus must be obeyed.

Formally reasoning about the safety of sandboxing requires first defining a machine model, and then defining what sandbox safety is in that model. Our machine model covers the subset of x64 targeted by the compiler. A simplified version of this model can be found in our paper, while the complete model can be found in our open-sourced code. We define the semantics for x64 as small-step semantics, allowing for reasoning about even potentially infinitely running code. Within this machine model, the program state contains an `ok` field, which is set to the value `AllOk` if and only if, until that point in execution, nothing invalid has occurred. Crucially, this also means that no accesses outside the memory allocated to the module have occurred. Sandboxing is safe if and only if, informally, starting from any initial `AllOk` state, executing the sandboxed code for any number of steps leads to an `AllOk` state.

Written more formally in F\*, but still slightly simplified for easier reading:

```
val sandbox_compile
  (a:aux) (c:code) (s:erased state): Err code
    (requires (
        (s.ok = AllOk) /\
        (reasonable_size a.sandbox_size s.mem) /\
        (s.ip `in_code` c) /\ ...))
    (ensures (fun c' ->
        forall n. (eval_steps n c' s).ok = AllOk))
```

<br>

This statement, written as pre- and post-conditions for the sandboxing pass `sandbox_compile`, shows that any code (`c'`) output by the sandboxer is formally guaranteed via the machine-checked proof to be safe. The pass takes two arguments `a` (auxiliary data) and `c` (the input program), and a computationally-irrelevant argument `s` (the initial state of the program, which is used for reasoning in our proofs, but that is erased when running the compiler), and returns output code `c'` under the custom effect `Err` (which allows the compiler to quit early upon error, for example if it finds a call to a non-existent function).  The statement guarantees that as long as the pre-conditions in the requires clause are satisfied, the post-condition in the ensures clause provably holds on the produced output code. The pre-conditions say that the initial state must be safe, have a reasonable sandbox size, and start from a valid location in the code; if these conditions are met, the output code `c'` will be safe when executed for any number of steps `n`.

The proofs for this theorem span approximately 3,500 lines of F\* code, not including the machine model or any of the supporting framework we built to write this proof. In total, vWasm consists of approximately 15,000 lines of F\* code and proofs, and required approximately two person-years of development effort.

## rWasm: High-Performance Informally-Proven-Safe Sandboxing

Our second technique, implemented as an open-source compiler, rWasm [[3]](#references), achieves provably-safe sandboxing via a careful embedding of Wasm semantics into safe Rust, such that the Rust compiler can then emit high-performance, safe machine code. This approach provides multiple benefits, such as portability across architectures, performance that is competitive with other unsafe compilers, and the ability to introduce runtime extensions (such as inline reference monitors---IRMs) that can be optimized in-tandem with the executed code.

Our insight for this approach is that the specific property of safe sandboxing is heavily intertwined with memory safety. In particular, code written in a memory-safe language cannot escape the confines of the memory provided to it. Informally, this means that by lifting (potentially unsafe) code to a memory-safe language, and then compiling that lifted code to machine code, the generated machine code must be safely sandboxed, due to the memory safety of the intermediate memory-safe language.

While other memory-safe languages would also suffice to obtain safe sandboxing, we pick Rust as our memory-safe language of choice for rWasm, since it is a non-garbage-collected systems-oriented language, which allows us to obtain predictable performance. While Rust _does_ have a non-memory-safe escape hatch via the `unsafe` keyword (since certain scenarios, such as writing an operating system, might need more control than directly allowed by the language), as long as this keyword is not used (ensured by the declaration `#![forbid(unsafe)]`), Rust guarantees memory safety. Given the prevalence of Rust in industry, and how seriously the Rust team takes unsoundness bugs, safe Rust is thus battle-tested to be memory safe, even if not (yet) proven to be so. Early efforts towards formalization of Rust and its security guarantees have already begun, such as with the RustBelt and Oxide projects.

We implement all stages of rWasm in safe Rust, but note that none of it needs to be trusted or verified. This means we do not need to depend upon the safety or correctness of any part of rWasm for the safety of the produced executable machine code. Instead, the safety of the produced code simply comes from the lack of any unsafe in the generated Rust code. Contrast this with say, wasm2c, which requires either trusting the compiler, or its generated C code, since C does not guarantee memory safety.

Astute readers will note that sandbox safety in any type-safe language also depends on the language's runtime libraries. Fortunately, rWasm imports nothing, uses only allocation-related features (for `Vec`), and even eliminates dependency on the Rust standard library via the `#![no_std]` directive. As with any sandbox, care is required when exposing an API to sandboxed code (e.g., to avoid APIs enabling sandbox bypasses directly or via confused deputies), but such concerns are orthogonal to sandbox construction.

## Evaluation

How do vWasm and rWasm perform in practice? We measure both techniques on a collection of quantitative and qualitative metrics, and while more details can be found in our full paper, we show some selected results here.

<figure><a name="fig2"></a><br>

![A graph, plotting normalized slowdown (on a log scale) on the y-axis against the Wasm runtimes on the x-axis. A summary of the graph is in the upcoming text.](./execution-time.svg)

<figcaption>Figure 2: Mean execution time of PolyBench-C benchmarks across the Wasm runtimes, normalized to pure native execution. Interpreters have square brackets; just-in-time (JIT) compilers have braces; the rest are ahead-of-time (AOT) compilers. vWasm* disables sandboxing.</figcaption>

<br></figure>

Run-time performance is critical for practical adoption in most applications. Hence, benchmark our compilers and various baselines using the PolyBench-C benchmark suite, which consists of thirty programs and has been a standard benchmark suite for Wasm since its inception. [Figure 2](#fig2) summarizes our results, showing the normalized execution time of the benchmarks on the Wasm runtimes.  Each point in the chart is the ratio of the mean time taken to execute the benchmark with the particular runtime vs. the mean time taken to execute by compiling the C code directly to non-sandboxed x64, skipping Wasm entirely.

The results indicate that, unsurprisingly, compilation strictly dominates interpretation for run-time performance. With respect to our compilers, we see that vWasm consistently outperforms the interpreters on all benchmarks, and that rWasm is competitive even with the compilers which are optimized for speed, and not necessarily safety. We note that the relative performance amongst the compilers can vary drastically based upon the workload (for example, on some of the longer-running programs in the benchmark suite, rWasm is more than twice as fast as WAVM [[8]](#references), which itself is twice as fast as rWasm on other benchmarks). Looking at vWasm and vWasm* (which is vWasm but with the sandboxing pass disabled), we find that the run time is marginally affected (by only 0.2%), indicating that almost all of the slowdown for vWasm, compared to other compilers, is due to the unverified portion of the compiler, which can be improved without needing to write any new proofs or even impacting existing proofs.

Next, we quantify the development effort needed to implement both vWasm and rWasm. The former took approximately two person-years to develop, including both code and proofs, while the latter took one person-month. This stark contrast is a testament to the daunting amount of work formal verification requires, even with modern, automated tools like F\*. It also illustrates the significant benefit of rWasm's carefully leveraging Rust's investment in safety.

Finally, provable safety is an important property of a verified sandboxing compiler, but one might wish to prove other properties, such as traditional compiler correctness. Here, vWasm has the upper hand, as this is feasible to do in F\*, and we have even structured the compiler to make such proofs possible. In contrast, proving correctness for rWasm would be a challenging task, since one would need to formally model the Rust language, show that rWasm preserves Wasm semantics in compiling to Rust, and then implement a semantics-preserving Rust compiler (or prove `rustc` as semantics-preserving). The nature of the provable sandboxing property is what puts it into the sweet spot where we obtain it "for free" when compiling to Rust, and we believe there may be other such properties where one can obtain provable guarantees in a similar fashion. However, all these properties are a strict subset of what might be proven for an implementation like vWasm, which is built in a full-blown verification-oriented language.

## Conclusion

In this work, we have explored two concrete points in the design space for implementing a sandboxing execution environment, with a focus on WebAssembly. We proposed designs for these two points, implemented them as open-source tools, vWasm and rWasm, and evaluated them on a collection of both quantitative and qualitative metrics. We show that run-time performance and provable safety are not in conflict, and indeed rWasm is the first Wasm runtime that is both provably-sandboxed and fast.

We refer the interested reader to our paper [[1]](#references) and to our open-source tools vWasm [[2]](#references) and rWasm [[3]](#references).

---

A version of this blogpost was previously posted as an [article in USENIX ;login:](https://www.usenix.org/publications/loginonline/provably-safe-multilingual-software-sandboxing-using-webassembly).

---

<a name="references"></a>
<small>
[1] Provably-Safe Multilingual Software Sandboxing using WebAssembly. Jay Bosamiya, Wen Shih Lim, and Bryan Parno. In Proceedings of the USENIX Security Symposium, August, 2022. Distinguished Paper Award _and_ Internet Defense Prize. [https://www.usenix.org/conference/usenixsecurity22/presentation/bosamiya](https://www.usenix.org/conference/usenixsecurity22/presentation/bosamiya)<br>
[2] vWasm: A formally-verified provably-safe sandboxing Wasm-to-native compiler. [https://github.com/secure-foundations/vWasm/](https://github.com/secure-foundations/vWasm/)<br>
[3] rWasm: A cross-platform high-performance provably-safe sandboxing Wasm-to-native compiler. [https://github.com/secure-foundations/rWasm/](https://github.com/secure-foundations/rWasm/)<br>
[4] F*: A Proof-Oriented Programming Language. [https://fstar-lang.org/](https://fstar-lang.org/)<br>
[5] Xavier Leroy, Sandrine Blazy, Daniel Kästner, Bernhard Schommer, Markus Pister, and Christian Ferdinand. CompCert - a formally verified optimizing compiler. In Embedded Real Time Software and Systems (ERTS). SEE, 2016.<br>
[6] Announcing Lucet: Fastly's native WebAssembly compiler and runtime. [https://www.fastly.com/blog/announcing-lucet-fastly-native-webassembly-compiler-runtime](https://www.fastly.com/blog/announcing-lucet-fastly-native-webassembly-compiler-runtime), March 2019.<br>
[7] Wasmtime: A small and efficient runtime for WebAssembly & WASI. [https://wasmtime.dev/](https://wasmtime.dev/)<br>
[8] WAVM: WebAssembly virtual machine. [https://wavm.github.io/](https://wavm.github.io/)<br>
</small>
