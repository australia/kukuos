# Yalanji OS — Glossary

This file is the **only** place in the repo where English is allowed to describe the
codebase. It maps the Kuku Yalanji identifiers used in our source to their meanings,
so an English-speaking reader can still orient themselves. All words below are drawn
from `docs/dictionary.yaml` (Kuku Yalanji dictionary, github.com/australia/mobtranslate.com).

## Repository layout (after retiring all non-Kuku tooling)

```
abo/
├── kuku/                # the OS, pure Kuku Yalanji source
│   ├── jakalbaku.kuku        (boot entry)
│   ├── bubu.kuku             (kernel main + banner)
│   ├── yanyil.kuku           (VGA 0xB8000 + serial mirror)
│   ├── bana.kuku             (COM1 serial)
│   ├── milka.kuku            (keyboard / serial input)
│   ├── wararra.kuku          (string compare)
│   └── balkalaway.kuku       (shell loop)
├── kuku-bama/           # host-mode Kuku (the compiler)
│   ├── ngunnga.kuku          (the compiler, in Kuku)
│   └── ngunnga              (the compiled Linux ELF; self-hosted)
├── docs/                # English-allowed zone
│   ├── dictionary.yaml       (the Kuku Yalanji dictionary)
│   ├── kuku.md               (language spec)
│   └── glossary.md           (this file)
└── yalanji.elf          # the compiled kernel ELF (build artifact)
```

No Python. No bash. No Dockerfile. No C. No Makefile. Just Kuku source and its
compiled binaries.

## Build (rebuild the OS)

```
./kuku-bama/ngunnga yalanji.elf kuku/*.kuku
```

## Rebuild the compiler itself from source

```
./kuku-bama/ngunnga --linux kuku-bama/ngunnga kuku-bama/ngunnga.kuku
```

The fixed point is reached on the first pass: the binary produced by this command
is byte-identical to the binary that ran it.

## Run the OS

No tooling is bundled; pick any i386 multiboot-capable runtime you like. For
example:

```
qemu-system-i386 -kernel yalanji.elf -display none -serial mon:stdio -no-reboot
```

To drive the shell over a TCP socket (e.g. for testing from the host):

```
qemu-system-i386 -kernel yalanji.elf -display none \
        -serial tcp:0.0.0.0:4444,server=on,wait=off -no-reboot &
nc -N 127.0.0.1 4444
```

Shell commands inside the OS: `kaday` (hello), `binal` (system info),
`jalngka` (clear screen), `kunbayn` (halt).

## OS name

- **Yalanji** — "this place / this language" (demonstrative). Used as the project name.

## Directories

| Directory    | Kuku Yalanji meaning        | Role                                    |
|--------------|-----------------------------|-----------------------------------------|
| `jakalbaku/` | beginning, first (time)     | Boot entry / multiboot header + `_start` |
| `dukurr/`    | inside, mind (noun)         | Kernel source                           |
| `ngunnga/`   | an open place (noun)        | Build staging / runtime assets          |
| `docs/`      | (English; exempted)         | Human documentation and the dictionary   |

## Kernel modules (files under `dukurr/`)

| File           | Kuku Yalanji meaning                      | Role                                  |
|----------------|-------------------------------------------|---------------------------------------|
| `bubu.c/.h`    | ground, earth (noun)                      | Kernel base / entry (`bubu_warri`)    |
| `yanyil.c/.h`  | to see, to examine (transitive-verb)      | VGA text output driver                |
| `milka.c/.h`   | ear (noun)                                | PS/2 keyboard input                   |
| `wararra.c/.h` | empty box (noun)                          | Buffer and string helpers             |
| `balkalaway.c` | to discuss, to talk together (i-verb)     | Shell loop                            |

## Function / variable name roots

| Identifier root | Kuku Yalanji meaning         | Used for                                  |
|-----------------|------------------------------|-------------------------------------------|
| `warri`         | to run, to fly (i-verb)      | "run" / entry point for a module          |
| `ngunnga`       | open place (noun)            | "open / initialise" functions             |
| `nandal`        | to close, shut (t-verb)      | "close / shutdown" functions              |
| `balkal`        | to tell, make (t-verb)       | "print / emit" functions                  |
| `babaji`        | to ask (t-verb)              | "read / prompt" functions                 |
| `mana`          | to get (t-verb, command)     | "get" accessors                           |
| `daya`          | to give (t-verb, command)    | "put / give" setters                      |
| `kujil`         | to keep, hold (t-verb)       | "hold / wait" functions                   |
| `janay`         | to stand, come to a stop (i) | "stop / halt" functions                   |
| `kunbayn`       | to finish (i-verb)           | Shutdown / end-of-loop                    |
| `binal`         | to know (associative)        | Info / status readers                     |
| `binalku`       | to remember (associative)    | Memory-related                            |
| `dukul`         | head, top                    | Head of a buffer, cursor row              |
| `jina`          | foot                         | Tail / bottom index                       |
| `mara`          | hand                         | Handle / pointer                          |
| `bangkarr`      | body                         | Struct / body of a thing                  |
| `wararra`       | empty box                    | Buffer                                    |
| `baral`         | road, path                   | Address / path                            |
| `bayan`         | house, shelter               | Container / struct instance               |
| `miyil`         | eye                          | Screen / viewport                         |
| `baya`          | fire, flame                  | Active / running state                    |
| `wungar`        | sun                          | Clock / time                              |
| `kuku`          | news, language (noun)        | A word / string / token                   |
| `bama`          | people, person               | The user                                  |
| `jirakal`       | new (adjective)              | "new" constructors                        |
| `jirray`        | big, many (adjective)        | Size = large                              |
| `buban`         | small (noun)                 | Size = small                              |
| `yanji`         | replete, full                | "full" state                              |
| `maru-maru`     | with nothing, empty (manner) | "empty" state                             |
| `nyubun`        | one                          | Constant 1                                |
| `jambul`        | two                          | Constant 2                                |
| `kulur`         | three                        | Constant 3                                |
| `yala`          | here, this (demonstrative)   | "this"                                    |
| `yinya`         | that, there (demonstrative)  | "that / next"                             |

## Shell commands

| Typed word | Meaning                  | Effect                                           |
|------------|--------------------------|--------------------------------------------------|
| `kaday`    | come                     | Prints a greeting to the screen                   |
| `binal`    | know                     | Prints kernel identity / status                   |
| `jalngka`  | smooth (adjective)       | Clears (smooths) the screen                       |
| `kunbayn`  | finish                   | Halts the CPU                                     |

## Tool-imposed English names (unavoidable)

The following names come from the toolchain and cannot be changed without forking
the tools. They are accepted as technical constants:

- `Dockerfile` — file name required by `docker build`.
- `.c`, `.h`, `.s`, `.ld` — file extensions required by gcc / gas / ld.
- C keywords (`int`, `void`, `return`, `if`, `while`, ...) — part of ISO C.
- Assembly directives (`.section`, `.global`, `.code32`) and mnemonics (`mov`, `jmp`, ...).
- Header names in `<stdint.h>` etc. — we avoid the C standard library entirely; we
  only use the freestanding subset and define our own fixed-width types.
- Port numbers (`0x60`, `0x64`, `0xB8000`) — hardware MMIO addresses.

Everything we **define** — directories, files, functions, types, variables,
user-facing strings — is drawn from the Kuku Yalanji dictionary.
