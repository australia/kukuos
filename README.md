# kukuos

KukuOS is a vertically integrated computing stack in Kuku Yalanji:

- a concatenative language called Kuku
- a self-hosted compiler for it, written in Kuku
- a bare-metal i386 kernel and shell, written in Kuku
- a Linux-hosted mode via `--bama`, also compiled from Kuku
- crypto primitives (SHA-256/512, ChaCha20, Poly1305, Curve25519, Ed25519)
- a character-level Transformer with autograd and Adam in `examples/dambun.kuku`

A deeper technical overview is in
[`docs/research-brief.md`](docs/research-brief.md).

Every identifier is a Kuku Yalanji word; the glosses are meaningful
(`kujil` — hold — is `dup`; `bana` — water — is the serial stream; `milka` —
ear — is the keyboard).

No C. No libc. No assembler. No linker. The compiler emits x86 bytes
directly into an ELF, with a `--bama` flag to switch between a multiboot
kernel and a 32-bit Linux executable.

## Try it

```
ssh yalanji@24.199.97.226          # interactive shell, password: yalanji
ssh yalanji-box@24.199.97.226      # /bin/sh with four kuku CLIs on $PATH
```

Neither path goes through QEMU; both drop you inside a Kuku-compiled Linux
process. The interactive shell supports `kaday`, `binal`, `jalngka`,
`kunbayn`, `yirrkay TEXT`, `balkal NAME TEXT`, `yanyil NAME`, `dukul`.
`balkal` and `yanyil` reject `NAME` containing `/`, and both sandbox at
`/srv/box`.

## Build

```
./kuku-bama/ngunnga yalanji.elf kuku/*.kuku
qemu-system-i386 -kernel yalanji.elf -display none -serial mon:stdio -no-reboot
```

Rebuild the compiler from its own source:

```
./kuku-bama/ngunnga --bama /tmp/ngunnga-new kuku-bama/ngunnga.kuku
cmp kuku-bama/ngunnga /tmp/ngunnga-new   # fixed point
```

The first compiler was bootstrapped from a Python seed, which has since
been deleted. Everything in the repo now builds from the existing binary.

## The language

Full spec in [`docs/kuku.md`](docs/kuku.md). Summary:

| role         | words                                                                  |
|--------------|------------------------------------------------------------------------|
| stack        | `kujil` dup · `wuljil` drop · `wundil` swap                            |
| integer math | `muru` + · `dumbarril` − · `*` × · `&` `\|` `^` `<<` `>>` · `kulbal`/`kanbal` signed div/mod |
| float (x87)  | `bana-@`/`bana-!` load/store · `bana-muru`/`-dumbarril`/`-*`/`-/` · `bana-sqrt`/`-sin`/`-cos`/`-log`/`-exp` · `bana-junkay`/`-jirray`/`-buban` compare |
| compare      | `junkay` = · `jirray` > · `buban` < · `kari` not                       |
| memory       | `@` / `!` / `c@` / `c!`                                                |
| I/O          | `mana-baral` inb · `daya-baral` outb                                   |
| CPU          | `kiway` cli · `wumbul` sti · `warngku` hlt                             |
| Linux ABI    | `bama-kunbayn` exit · `bama-{balkal,babaji,ngunnga,nandal}` = write/read/open/close · `bama-wangkanil`/`-wararra` argc/argv |

Control flow:

```
balkalaway NAME                     # function
    yala ... yinya ... kunbayn      # if/else
    yabarrka ... janay ... kunbayn  # loop with break
kunbayn

wararra NAME 1 2 "kaday" 0 nandal   # initialised rodata
wararra-bss NAME 4096               # bss
```

The data stack lives at `[ebp]`, growing down by 4 bytes per push. `esp`
is reserved for the CPU call stack. Every primitive manipulates `[ebp]`
directly; there is no register-based temporary. This is why functions
sharing globals like `tmp-i` clobber each other across calls.

## How the compiler works

`kuku-bama/ngunnga.kuku` is a single ~1700-line source file that compiles
in two passes:

1. **Prescan.** Walk every input file once, recording every
   `wararra NAME … nandal`, `wararra-bss NAME SIZE`, and
   `balkalaway NAME … kunbayn` declaration. Each gets a symbol-table
   entry with a placeholder address; function bodies are not yet
   compiled. Sizes of initialised data blocks are summed at this stage
   so the final ELF layout is known before any code is emitted.

2. **Emit.** Walk the files again. Each `balkalaway` body is compiled
   linearly: for every token, try the primitive table (exact-match) and
   emit the fixed byte sequence if it's a primitive; otherwise look up
   the symbol table and emit a relative `call`, a `push imm32` for an
   integer literal, or an absolute pointer for a `wararra` / `wararra-bss`
   reference. Forward references are patched in a fixup pass.

There is no intermediate representation, no register allocator, no dead-code
elimination. The compiler walks tokens left-to-right and writes bytes.
A word like `muru` is always three bytes: `add eax, [ebp+4]; add ebp, 4;
mov [ebp], eax`. A call to a user function is always `e8 xx xx xx xx`.

ELF emission has two modes selected by the `--bama` flag:

- **Kernel (no flag).** Multiboot 0.6.96 header, entry at 0x10000C,
  single `PT_LOAD` covering text + rodata + bss. This is what
  `qemu-system-i386 -kernel` loads.
- **Linux user-space (`--bama`).** 32-bit static ELF, entry at
  0x8048054, single `PT_LOAD` at 0x08048000 with `PF_R|PF_W|PF_X`, no
  dynamic linker, no sections. Runnable directly on any Linux host with
  `CONFIG_IA32_EMULATION=y` (the default).

Both modes share exactly the same compiled `.text`; only the ELF wrapper
and the entry prologue differ.

## The kernel

`kuku/*.kuku` is a serial-console REPL plus drivers. No heap, no
scheduler, no interrupts; the CPU sits in `hlt` between commands.

Entry is `jakalbaku-warri` (Kuku Yalanji "first thing"). After multiboot
hands off control, the prologue sets `ebp` to the top of the data-stack
buffer and falls through to the main read-eval-print loop: read one line
from the 16550 UART (`bana`), tokenise it, dispatch the first word
against a small command table, and print the result back out.

Current drivers:

| driver | source word | what it does |
|--------|-------------|--------------|
| serial | `bana` | 16550 UART at COM1 (0x3F8), polled read/write |
| keyboard | `milka` | PS/2 scan-code reader, translated to ASCII |
| VGA text | `jalngka` | 80×25 text mode at 0xB8000 |
| PCI | `wari` | config-space probe via 0xCF8/0xCFC |
| virtio-net | `wari` (continued) | legacy init through `DRIVER_OK`, MAC read |

The virtio-net driver stops after status handshake; no queues, no
packets yet. A full network stack is the long-running roadmap item.

## `--bama` mode (Linux user-space)

Compiling with `--bama` emits a 32-bit static Linux ELF. The compiler
inserts an entry prologue that:

1. Sets `ebp` to the top of a BSS-allocated data-stack buffer.
2. Copies the Linux-provided `argc` (at `[esp]`) and `argv` pointer (at
   `esp+4`) into BSS slots that `bama-wangkanil` / `bama-wararra` read.
3. Jumps to `jakalbaku-warri`.

All I/O goes through `int 0x80`. Seven syscalls are wrapped:

```
bama-ngunnga   ( path flags -- fd )     open(2)
bama-balkal    ( fd buf n -- written )  write(2)
bama-babaji    ( fd buf n -- read )     read(2)
bama-nandal    ( fd -- ret )            close(2)
bama-kunbayn   ( code -- )              exit(2), does not return
bama-wangkanil ( -- argc )              cached from entry
bama-wararra   ( -- argv )              cached from entry
```

This is enough to write shell utilities, crypto implementations that
read stdin and write stdout, and the Transformer training/sampling
path in `examples/dambun.kuku`.

## Autograd in Kuku

`examples/dambun.kuku` implements reverse-mode autograd from scratch.
The unit is a 48-byte `Value` cell:

```
 +0   data          f64   forward value
 +8   grad          f64   accumulated gradient (∂L/∂self)
+16   op_type       u32   tag (unused by backward, kept for debugging)
+20   child1        ptr   first operand, or 0 for a leaf
+24   child2        ptr   second operand, or 0
+28   local_grad1   f64   ∂self/∂child1 at the forward values
+36   local_grad2   f64   ∂self/∂child2
+44   visited       u32   topological-sort flag
```

Every differentiable primitive (`kuku-muru`, `kuku-jirray`, `kuku-mulkurr`
for log, `kuku-baya` for exp, `kuku-wandil` for ReLU, `kuku-kulbal`
for divide, `kuku-jalngka-wundil` for reciprocal-sqrt, …) bump-allocates
a cell from the per-step arena, stores the operand pointers in
`child1`/`child2`, evaluates `.data` by loading the operands' `.data`
fields onto the x87 stack and applying the matching FPU instruction,
and precomputes the local partials from the forward values (for `a*b`:
`∂/∂a = b.data`, `∂/∂b = a.data`).

Backward is a two-step reverse-mode pass:

1. **Topological sort.** Starting from the loss, recursively mark
   `visited` on the cell and its children, then append the cell itself
   to a flat array. Children are read into the data stack before the
   recursive call (not into globals — an earlier version kept them in
   globals and the child-2 slot was overwritten by the recursion,
   silently pruning most of the graph).
2. **Accumulate.** Set `loss.grad = 1.0`. Walk the sorted array in
   reverse: for each cell, `child1.grad += local_grad1 * self.grad`
   and likewise for `child2`.

Between training steps the per-step arena pointer is reset to its base,
which reclaims all intermediate cells. The parameter arena is persistent;
its `visited` flags are cleared at the start of every backward pass.

## The Transformer

`dambun.kuku` is a 1-layer character-level Transformer with a 16-dim
embedding, 4 attention heads, 64-hidden MLP, and a 16-token block size,
on a 27-symbol vocabulary (26 lowercase + BOS). 4,192 trainable
parameters total, all FP64.

One training step:

1. Read the next line from `names.txt`, tokenise to
   `[BOS, c_1, …, c_n, BOS]`.
2. For each position, run the forward pass: embed → pre-RMSnorm →
   multi-head self-attention (with a growing KV cache over positions
   seen so far) → residual → pre-RMSnorm → 64-hidden ReLU MLP → residual
   → `lm_head`. Take cross-entropy against the next token.
3. Sum the per-position losses, divide by `n`, backpropagate.
4. Adam update: per-parameter `m ← β₁·m + (1-β₁)·g`,
   `v ← β₂·v + (1-β₂)·g²`, bias-correct, and
   `θ ← θ - η·m̂/(√v̂ + ε)`.

Inference (`./dambun balkalaway`) feeds BOS through the same forward
pass, divides logits by a temperature, softmaxes, draws from the
multinomial via an inverse-CDF walk, and repeats until BOS is produced
again or the block fills.

## Examples

Each file in [`examples/`](examples/) is a self-contained Kuku program
that compiles with `./kuku-bama/ngunnga --bama out src.kuku` and asserts
against published test vectors (non-zero exit on mismatch).

| file | what it is |
|------|------------|
| `nganjal.kuku` | file I/O round-trip via `bama-*` |
| `sha256.kuku`, `sha512.kuku` | NIST vectors |
| `chacha20.kuku` | RFC 7539 block |
| `curve25519-x25519.kuku` | RFC 7748 |
| `ed25519-sign.kuku` | RFC 8032 vector 1 |
| `dambun.kuku` | character-level Transformer: Value-based autograd, multi-head attention, Adam, temperature sampling |
| `paper.kuku` | writes a LaTeX paper about `dambun.kuku` to `/tmp/dambun-paper.tex` |

`dambun.kuku` trains from cross-entropy `log(27)` to about `1.01` in 100
single-example Adam steps on the 32 k US first-names corpus, and samples
20 names to stdout. Full write-up on the
[HuggingFace repo](https://huggingface.co/ajaxdavis/kuku-dambun) including
the PDF.

## Floating point

Strictly x87, because the compiler predates an SSE code path. Each
arithmetic primitive maps 1:1 to an FPU opcode (`FADDP`, `FDIVP`,
`FSQRT`, …). `bana-exp` is a 32-byte inline `FLDL2E`/`FMULP`/`FPREM`/
`F2XM1`/`FSCALE` sequence that computes `e^x = 2^(x · log₂ e)`.
Gradients and model parameters are FP64 throughout.

## Conventions

- Every identifier appears in `docs/dictionary.yaml`. Not enforced by
  the compiler.
- Keep the data stack balanced across every branch of `yala`/`yinya`.
- `tmp-a`/`tmp-i`/`tmp-j` are shared globals and get clobbered by
  sub-calls — give helpers with non-trivial state their own slots.
- Big zero buffers go in `wararra-bss`, not `wararra 0 0 … 0 nandal`
  (the latter emits real zeros into the ELF).
- After changing `kuku-bama/ngunnga.kuku`, confirm the self-host fixed
  point before committing.

## Pre-built image

```
docker run --rm -it thomasdavis/yalanji
```

Starts QEMU with the bundled kernel on the serial console.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). One hard rule: every
identifier must resolve to an entry in `docs/dictionary.yaml`.

## License

[MIT](LICENSE).
