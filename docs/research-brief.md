# KukuOS Research Brief

This repository is more interesting than its README currently makes clear.
It is not just "an OS in a niche language". It is a vertically integrated
computing stack built around a language whose identifiers are drawn from Kuku
Yalanji:

- a concatenative language (`docs/kuku.md`)
- a self-hosting compiler in that language (`kuku-bama/ngunnga.kuku`)
- a bare-metal i386 kernel and shell (`kuku/*.kuku`)
- a Linux-hosted userland mode via `--bama`
- cryptographic implementations with standard-vector examples
- a character-level Transformer with reverse-mode autograd (`examples/dambun.kuku`)
- a Kuku program that emits a LaTeX paper about that Transformer (`examples/paper.kuku`)

## Thesis

KukuOS is best understood as a language-and-systems research artifact with an
unusually tight vertical slice: naming, syntax, compiler, runtime, operating
system, cryptography, and ML are all expressed inside the same small language.
The interesting claim is not just "this works", but "how much of modern
computing can be reconstructed inside a semantically coherent, self-hosted,
non-English concatenative environment?"

## Why It Is Technically Interesting

### 1. The language is a real design constraint, not decoration

Kuku is not a Forth clone with renamed words. The repo treats Kuku Yalanji as
part of the programming model:

- built-ins are Kuku Yalanji words with semantically chosen glosses
- user-defined identifiers are expected to come from `docs/dictionary.yaml`
- the glossary is part of the code-reading workflow, not an afterthought

That makes the project interesting to PL people for at least two reasons:

- naming is treated as semantics, not just syntax sugar
- the language boundary forces a different style of abstraction and code reading

### 2. The compiler is small, self-hosted, and emits machine code directly

`kuku-bama/ngunnga.kuku` is a 1,375-line compiler written in Kuku itself. It
lexes, prescans, compiles, resolves fixups, and writes a 32-bit ELF directly.
There is no assembler, linker, libc, or C in the active codebase.

The compiler contains raw primitive records that map Kuku words directly to x86
bytes, including an x87 floating-point subset. This is closer to "a handwritten
ELF and opcode generator" than to a conventional hosted compiler pipeline.

### 3. The same language straddles bare metal and host mode

The `--bama` switch changes the target from a multiboot kernel ELF to a Linux
executable. That means the same language serves both as:

- kernel implementation language
- compiler implementation language
- host-side application language

That duality is one of the strongest parts of the project. It makes Kuku useful
before full self-sufficiency on bare metal, while preserving a single-language
story.

### 4. The examples are not toy arithmetic

The examples directory is where the repo really opens up:

- `sha256.kuku`, `sha512.kuku`: NIST-vector hash implementations
- `chacha20.kuku`: RFC 7539 block function
- `curve25519-x25519.kuku`: RFC 7748
- `ed25519-*.kuku`: point ops, scalars, signing path
- `yalanji-shell.kuku`: Linux-hosted shell with file-pad commands
- `dambun.kuku`: reverse-mode autograd, attention, MLP, Adam, sampling
- `paper.kuku`: emits a LaTeX paper about `dambun`

This is the real argument for the repo: it demonstrates computational breadth.

### 5. The ML example changes the scale of the project

`examples/dambun.kuku` is not "AI-themed". It is an actual end-to-end training
and sampling program:

- IEEE-754 double constants and x87 arithmetic
- `Value` cells for autograd
- topological sort + backward pass
- xorshift32 + Box-Muller RNG
- tokenization and corpus loading
- embeddings, multi-head attention, RMSNorm, MLP
- Adam optimization
- parameter save/load
- autoregressive sampling

There is also an unusual meta-level move here: `paper.kuku` generates a paper
about `dambun`, so the language is being used both to build the artifact and to
describe the artifact.

## Verified Claims

These were checked directly against the repo on 2026-04-19 in this workspace.

- Self-host fixed point: `./kuku-bama/ngunnga --bama /tmp/ngunnga-selfhost kuku-bama/ngunnga.kuku` produced a binary identical to `kuku-bama/ngunnga`.
- Compiler artifact: `kuku-bama/ngunnga` is a statically linked 32-bit ELF and is about 24 KB.
- SHA-256 example: compiling and running `examples/sha256.kuku` produced `BA7816BF...15AD`, the standard digest for `abc`.
- ChaCha20 example: compiling and running `examples/chacha20.kuku` produced the RFC 7539 test-vector block.
- Ed25519 example: compiling and running the signing example emitted the RFC 8032 vector-1 signature.
- Linux shell: `examples/yalanji-shell.kuku` compiles to a 6.8 KB static ELF and runs as a normal Linux process, no QEMU required.
- Transformer artifact: `examples/dambun.kuku` compiles to a roughly 32 KB static ELF.
- Training path: running `/tmp/dambun binalku /tmp/kuku-names.txt` on a tiny temporary corpus completed, saved a 262 KB model file, and showed loss decreasing from about `3.342` to `0.995` over 100 steps.
- Sampling path: running `/tmp/dambun balkalaway` after that produced plausible name-like strings.
- Paper generation: `examples/paper.kuku` emitted a valid LaTeX file to `/tmp/dambun-paper.tex`.

## Scale At A Glance

Observed in this repo:

- 32 Kuku source files
- 7,553 lines across `kuku-bama/`, `kuku/`, and `examples/`
- 326 `balkalaway` word definitions

Those numbers matter because the project is still small enough to comprehend as
a whole, while being broad enough to demonstrate serious capability.

## Best Framing For Academic Programmers

If your audience is into PL, compilers, systems, formalism, or odd-but-serious
experiments, the strongest framing is:

> KukuOS is a self-hosting concatenative language and operating-system stack
> whose naming and user-defined vocabulary are drawn from Kuku Yalanji. The
> compiler emits x86 ELF binaries directly, targets both multiboot kernels and
> Linux executables, and the examples go all the way up to standards-tested
> cryptography and a trained character-level Transformer with reverse-mode
> autograd.

What tends to land well:

- It collapses language design, compilation, systems, crypto, and ML into one substrate.
- It is small enough to inspect and weird enough to be memorable.
- It uses natural-language semantics as a programming constraint rather than a branding layer.
- It demonstrates self-hosting without leaning on a conventional toolchain once bootstrapped.

## Short Version You Can Send

KukuOS is a vertically integrated computing stack in Kuku Yalanji: a tiny
concatenative language, a self-hosting compiler that emits 32-bit ELF binaries
directly, a bare-metal OS, Linux-hosted userland tools, standards-tested crypto
implementations, and a character-level Transformer with autograd and Adam - all
in the same language, with no C, libc, assembler, or linker in the active
codebase.

## Fast Demo Path

```bash
./kuku-bama/ngunnga --bama /tmp/ngunnga-selfhost kuku-bama/ngunnga.kuku
cmp kuku-bama/ngunnga /tmp/ngunnga-selfhost

./kuku-bama/ngunnga --bama /tmp/sha256 examples/sha256.kuku && /tmp/sha256
./kuku-bama/ngunnga --bama /tmp/yalanji-shell examples/yalanji-shell.kuku
./kuku-bama/ngunnga --bama /tmp/dambun examples/dambun.kuku
```

## One Caveat

Some English-language docs still describe earlier bootstrap history, including a
Python seed compiler that is no longer present in the active repo. The source
tree today is stronger than that older framing suggests: the current repository
is already centered on the self-hosted Kuku compiler and Kuku source.
