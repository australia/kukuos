# kukuos

An i386 operating system, a stack language called Kuku, and a self-hosted
compiler for it. The compiler is written in Kuku. So is the kernel. So are
the crypto primitives (SHA-256/512, ChaCha20, Poly1305, Curve25519, Ed25519)
and the character-level Transformer in `examples/dambun.kuku`.

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
