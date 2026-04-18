# Contributing to kukuos / Yalanji

Thanks for wanting to help.  This project is unusual — read this whole file
before you send a PR.

## What this project is, in one sentence

An i386 operating system, and the stack language (Kuku) that compiles it,
both written entirely in Kuku Yalanji — an Aboriginal Australian language —
with no C, no libc, no assembler, no linker, and no external build tool.

## The one hard rule: dictionary discipline

**Every identifier you introduce MUST be a real word from Kuku Yalanji.**
The dictionary at `docs/dictionary.yaml` is authoritative.  If the word
isn't there, you don't get to use it.

- Multi-word names are hyphenated: `yanyil-balkal-jina` (see-tell-byte) is
  fine.  Inventing Kuku-sounding tokens is not.
- Do not Anglicise.  Don't write `print-hex`; use `yanyil-balkal-hex`
  (yanyil = see, balkal = speak, hex = from the dictionary).
- If the concept you need really has no Kuku Yalanji word, raise an issue
  to discuss before guessing.  Dictionary drift is the one kind of technical
  debt this project cannot absorb.

## What goes in the binaries

We ship two binaries and they are bit-for-bit reproducible from source:

| File                   | What it is                                      | Committed? |
|------------------------|-------------------------------------------------|------------|
| `kuku-bama/ngunnga`    | The Kuku compiler.  Compiles itself from        | **yes**    |
|                        | `kuku-bama/ngunnga.kuku`.  Required seed.       |            |
| `yalanji.elf`          | The OS kernel, built from `kuku/*.kuku`.        | no         |

The kernel is a derived artefact; regenerate it locally with:

```
./kuku-bama/ngunnga yalanji.elf kuku/*.kuku
```

The compiler **is** committed because there is no Python/C fallback — the
self-hosted seed is the only way into the project.  If you change
`kuku-bama/ngunnga.kuku`, you must rebuild the seed:

```
./kuku-bama/ngunnga --linux /tmp/new kuku-bama/ngunnga.kuku
cmp kuku-bama/ngunnga /tmp/new && mv /tmp/new kuku-bama/ngunnga
```

CI will refuse any PR whose committed `kuku-bama/ngunnga` binary does not
compile itself to a byte-identical copy.

## What's NOT allowed in the source

- No C, C++, Rust, Go, or any other language.  Only `.kuku` files.
- No assembly mnemonics (`mov`, `push`, `call`).  We emit raw x86 bytes
  from Kuku; assembler-speak has no home here.
- No English keywords (`int`, `void`, `if`, `while`).  They give the game
  away.
- No Makefile, Dockerfile, or shell-script build glue in the repo.  The
  single command `./kuku-bama/ngunnga yalanji.elf kuku/*.kuku` is the build
  system.

Exceptions: the `docs/` folder (prose, dictionary, spec) is English-in-
Markdown on purpose, and `.github/workflows/*.yml` is the only YAML
allowed.

## Scope of contributions

Welcome:
- Bug fixes to the compiler or kernel.
- New shell commands that use real Kuku Yalanji words.
- Driver work (virtio-net, timer, PIC, ATA) that moves the SSH-in-Kuku
  roadmap forward.
- Documentation improvements, especially examples for newcomers.
- Dictionary expansion in coordination with
  https://github.com/australia/mobtranslate.com.

Please don't:
- Rewrite the compiler in another language.
- Mechanically rename Kuku Yalanji words into English.
- Add runtime dependencies (libc, libssl, etc.).  The whole point is that
  there aren't any.

## Submitting a change

1. Fork + branch.  Keep PRs focused on one thing.
2. Make sure `./kuku-bama/ngunnga --linux /tmp/new kuku-bama/ngunnga.kuku &&
   cmp kuku-bama/ngunnga /tmp/new` still passes.
3. Rebuild the kernel and boot-test locally:
   ```
   ./kuku-bama/ngunnga yalanji.elf kuku/*.kuku
   qemu-system-i386 -kernel yalanji.elf -display none \
                    -serial mon:stdio -no-reboot
   ```
4. Open a PR.  Describe what Kuku Yalanji word you chose and why.

CI will run the self-host check and a boot smoke test automatically.

## Cultural respect

Kuku Yalanji belongs to the Kuku Yalanji people.  If your PR touches the
dictionary or makes claims about the language, coordinate with the
mobtranslate.com maintainers rather than acting unilaterally.  When in
doubt, ask.
