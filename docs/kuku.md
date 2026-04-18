# Kuku — the language

`kuku` (noun, Kuku Yalanji) = "news, language, word".

Kuku is a tiny stack-based language. Its built-in words are Kuku Yalanji words
from `docs/dictionary.yaml`. Source lives in `.kuku` files and is turned into an
i386 multiboot1 ELF kernel — no C, no libc, no gcc, no binutils, no nasm.

## The bootstrap problem

No computer speaks Kuku Yalanji natively. The very first compiler of any
language is always written in something that already runs — C's first compiler
was B, Rust's was OCaml, and so on. Kuku is no different: we need a *bootstrap*.

This repo's bootstrap is `ngunnga.py` ("open place"), a minimal Python script
whose only job is to read `.kuku` source, apply the rules in this document, and
emit an i386 ELF. Think of it as a stencil: it imposes no opinions of its own,
it just carries Kuku through to bytes.

When the OS is self-hosted enough, `ngunnga.py` will be replaced by a Kuku
program that emits its own bytes — at which point the bootstrap disappears
and every line in the repo (outside `docs/`) is Kuku.

Every user-defined name in the OS source must be a word that appears as a
`word:` entry in `docs/dictionary.yaml`, optionally joined with `-` (e.g.
`yanyil-balkal-jina`).

## Execution model

- Two stacks.
  - **Data stack**: 32-bit values; top pointed to by `ebp`. Grows downward.
  - **Return stack**: the hardware stack, `esp`. Used only for `call`/`ret`.
- All Kuku words are functions. Built-ins expand inline; user-defined words
  become `call`/`ret` functions.
- Signed 32-bit integers. Booleans: 0 is false; anything else is true.
  Comparison words push 1 or 0.

## Literals

| Form              | Effect                                                            |
|-------------------|-------------------------------------------------------------------|
| `123`, `-7`       | Push decimal integer.                                             |
| `0xB8000`         | Push hex integer.                                                 |
| `'A'`             | Push ASCII code of the character.                                 |
| `"kaday"`         | Push the address of a null-terminated byte sequence in rodata.    |

## Built-in words

### Stack
| Kuku       | Meaning                         | Behaviour              |
|------------|---------------------------------|------------------------|
| `kujil`    | hold / keep                     | `a — a a`  (dup)       |
| `wuljil`   | empty out                       | `a —`       (drop)     |
| `wundil`   | bring / take                    | `a b — b a` (swap)     |

### Arithmetic / bitwise
| Kuku       | Meaning              | Behaviour              |
|------------|----------------------|------------------------|
| `muru`     | together             | `a b — (a+b)`          |
| `dumbarril`| break / tear         | `a b — (a-b)`          |
| `&`        | (symbol)             | `a b — (a & b)`        |
| `|`        | (symbol)             | `a b — (a | b)`        |
| `^`        | (symbol)             | `a b — (a ^ b)`        |
| `<<`       | (symbol)             | `a b — (a << b)`       |
| `>>`       | (symbol)             | `a b — (a >> b)` logical|

### Comparison (push 1 / 0)
| Kuku       | Meaning              | Behaviour              |
|------------|----------------------|------------------------|
| `junkay`   | correct / straight   | `a b — (a == b)`       |
| `jirray`   | big / much           | `a b — (a  >  b)`      |
| `buban`    | small                | `a b — (a  <  b)`      |

### Memory
| Kuku       | Behaviour                                            |
|------------|------------------------------------------------------|
| `@`        | `addr — value32` — fetch 32-bit word.                |
| `!`        | `value32 addr —` — store 32-bit word.                |
| `c@`       | `addr — byte` — fetch unsigned byte.                 |
| `c!`       | `byte addr —` — store low byte.                      |

### I/O ports
| Kuku          | Behaviour                                            |
|---------------|------------------------------------------------------|
| `mana-baral`  | `port — byte` — inb.                                 |
| `daya-baral`  | `byte port —` — outb.                                |

### Interrupt / CPU
| Kuku       | Meaning          | Instruction |
|------------|------------------|-------------|
| `kiway`    | cold             | `cli`       |
| `wumbul`   | hot              | `sti`       |
| `warngku`  | sleep            | `hlt`       |
| `jalngka`  | smooth           | `pause`     |

## Control flow

- **Define a word**
  ```
  balkalaway foo
      …body…
  kunbayn
  ```
  `balkalaway` ≈ "discuss / talk together" — a new shared word.
  `kunbayn` = "finish" — closes any open block.

- **If / else** — pops top of stack; 0 → else, non-zero → then.
  ```
  yala            # then-branch if top != 0
      …
  yinya           # optional else-branch
      …
  kunbayn
  ```
  `yala` = "here / this"; `yinya` = "that / there".

- **Infinite loop**
  ```
  yabarrka
      …
  kunbayn
  ```
  `yabarrka` = "all the time, always".

- **Break out of innermost loop**
  ```
  janay
  ```
  `janay` = "stand / come to a stop".

- **Call a user word**: just write the word name.
  ```
  foo bar baz
  ```

## Data blocks

```
wararra wanju         # declare a byte array
    0 27 '1' '2' '3' "quick brown"  0
nandal
```
`wararra` = "empty box". `nandal` = "close, shut".
At runtime, a reference to `wanju` pushes its starting address.

Integer tokens inside a `wararra` block emit **one byte each** (mod 256).
String literals expand to their bytes (no trailing NUL unless you write `0`).
`maru-maru N` inside a data block emits `N` zero bytes — used for reserving
buffer space.

## Linux-ABI primitives (for host tools)

When compiled with `--bama`, Kuku emits a Linux x86 ELF that can run as a
normal process.  The following primitives become available:

| Kuku              | Meaning / syscall (eax)                                  |
|-------------------|----------------------------------------------------------|
| `bama-kunbayn`    | `status —`       exit(status).                           |
| `bama-balkal`     | `fd buf n — nw`  write(fd, buf, n).                      |
| `bama-babaji`     | `fd buf n — nr`  read(fd, buf, n).                       |
| `bama-ngunnga`    | `path flags — fd`  open(path, flags, 0644).              |
| `bama-nandal`     | `fd — ret`       close(fd).                              |
| `bama-jirray`     | `addr — new_brk` brk(addr).                              |
| `bama-wangkanil`  | `— argc`         the process's argument count.           |
| `bama-wararra`    | `— argv`         base address of the argv[] array.       |

`bama-` (from Kuku Yalanji *bama*, "people / the outer world") marks syscalls
into the host kernel — the boundary between our Kuku world and the host.

## Comments

`#` starts a line comment (terminated by end-of-line).  The `#` byte is
a symbol, not an English word.

## Source layout

One kernel ELF is produced from the concatenation of all `.kuku` files in the
repo, processed in alphabetical order.  The first word defined named
`jakalbaku-warri` becomes the multiboot entry point.
