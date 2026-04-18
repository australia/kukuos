# Yalanji — an operating system written in Kuku Yalanji

[![CI](https://github.com/australia/kukuos/actions/workflows/ci.yml/badge.svg)](https://github.com/australia/kukuos/actions/workflows/ci.yml)
[![license: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Docker pulls](https://img.shields.io/docker/pulls/thomasdavis/yalanji.svg)](https://hub.docker.com/r/thomasdavis/yalanji)

> **Acknowledgement of Country.**  This project uses the **Kuku Yalanji**
> language of the Kuku Yalanji people, Traditional Custodians of the land
> spanning the wet tropics between the Annan and Daintree rivers in Far
> North Queensland, Australia.  We pay respect to Elders past and present,
> and acknowledge that Kuku Yalanji — the language and its cultural
> stewardship — belongs to the Kuku Yalanji people, not to this code.

Yalanji is a from-scratch i386 operating system whose source is written
exclusively in **Kuku Yalanji**.  Every function, variable, directory, file
name and user-facing string is a real Kuku Yalanji word taken from the
community dictionary
([mobtranslate.com](https://github.com/australia/mobtranslate.com)).

The repo contains:

1. The OS source (`kuku/*.kuku`) — ~500 lines of a stack language called *Kuku*.
2. The compiler source (`kuku-bama/ngunnga.kuku`) — also Kuku, ~1700 lines.
3. The compiler binary (`kuku-bama/ngunnga`) — the Kuku compiler, compiled from
   its own source.  It emits raw i386 machine code.
4. Documentation (`docs/`) — the dictionary, the language spec, and this README.

That's the whole project.  There is no C, no libc, no assembler, no linker, no
shell glue, and no Dockerfile/Makefile in the repo.

---

## What's novel about this

- **An OS in an Aboriginal Australian language.**  The identifiers in the
  source are not transliterations or tokens — they are Kuku Yalanji words
  taken directly from the community dictionary, each with a meaning that
  informs what the code does.  `kujil` (hold) duplicates the stack top.
  `wararra` (empty box) is a buffer.  `bana` (water) is a stream — our
  serial driver.  `milka` (ear) is the keyboard.
- **A programming language built for this project, in itself.**  The Kuku
  compiler is written in Kuku.  It reads `.kuku` source, runs in two passes
  (prescan + compile), and writes out an i386 ELF with multiboot or Linux
  entry — all by emitting bytes directly.  No C toolchain is involved at any
  point.
- **Self-hosted with a retired seed.**  The very first compiler binary was
  produced by a ~500-line Python seed (unavoidable — the first compiler of
  any language is written in something else).  That seed has been deleted.
  The compiler now rebuilds itself byte-for-byte in a single pass.
- **BSS-aware output.**  `wararra-bss NAME SIZE` declares a symbol that
  occupies space in `p_memsz` but not in `p_filesz` — the same mechanism a
  C linker uses for `.bss`, implemented from first principles in the
  compiler's own Kuku source.
- **~23 KB compiler, ~5.6 KB kernel.**  Both boot-tested under QEMU.

---

## Quick start

### Option 0 — SSH straight into the public demo

```
ssh yalanji@24.199.97.226
# password: yalanji
```

Connects you to a cheap Digital Ocean droplet whose login shell IS a QEMU
instance booting the Yalanji kernel.  Sessions cap at 10 minutes.

### Option 1 — pull the pre-built image

```
docker run --rm -it thomasdavis/yalanji
```

The container starts QEMU with the bundled kernel and drops you into the
serial shell.  Exit with `kunbayn` (halt) or `Ctrl-A X`.

### Option 2 — build from source

Requirements: Linux, `qemu-system-i386` for running.

```
# compile the OS from Kuku source (no other toolchain needed)
./kuku-bama/ngunnga yalanji.elf kuku/*.kuku

# boot it
qemu-system-i386 -kernel yalanji.elf -display none \
                 -serial mon:stdio -no-reboot
```

### Option 3 — rebuild the compiler itself

Prove self-hosting in one command:

```
./kuku-bama/ngunnga --linux /tmp/ngunnga-new kuku-bama/ngunnga.kuku
cmp kuku-bama/ngunnga /tmp/ngunnga-new && echo "fixed point reached"
```

---

## Shell commands inside the OS

| Typed word  | Kuku Yalanji meaning  | Effect                                                |
|-------------|-----------------------|-------------------------------------------------------|
| `kaday`     | come                  | Greets you: `kaday, bama!`                            |
| `binal`     | know                  | Prints kernel identity and status                     |
| `wari`      | sign / call           | Scans the PCI bus and reports any device found        |
| `jalngka`   | smooth                | Clears the screen                                     |
| `kunbayn`   | finish                | Halts the CPU                                         |

Any unrecognised input is answered with `binal kari: <word>` — "I don't know
that word."

---

## Writing Kuku

Full spec: [`docs/kuku.md`](docs/kuku.md).

| Role       | Kuku                                                        |
|------------|-------------------------------------------------------------|
| Stack      | `kujil` dup · `wuljil` drop · `wundil` swap                 |
| Arithmetic | `muru` + · `dumbarril` − · `*` × · `&` `\|` `^` `<<` `>>`   |
| Compare    | `junkay` = · `jirray` > · `buban` < · `kari` not            |
| Memory     | `@` fetch32 · `!` store32 · `c@` · `c!`                     |
| I/O ports  | `mana-baral` inb · `daya-baral` outb                        |
| CPU        | `kiway` cli · `wumbul` sti · `warngku` hlt · `jalngka` pause |
| Linux ABI  | `bama-kunbayn` exit · `bama-balkal` write · `bama-babaji` read · `bama-ngunnga` open · `bama-nandal` close · `bama-wangkanil` argc · `bama-wararra` argv |

Control flow:

```
balkalaway NAME          # define a word
    yala                 # if top of stack is non-zero
        ...
    yinya                # else
        ...
    kunbayn              # endif
    yabarrka             # loop forever
        ...
        janay            # break
    kunbayn
kunbayn                  # end word (emits ret)

wararra NAME 1 2 3 "kaday" 0 nandal   # byte-literal data block
wararra-bss NAME 262144               # zero-filled BSS block
```

---

## Reading and writing files

Kuku programs can be compiled for Linux (flag `--linux`) instead of as a
bare-metal kernel.  In that mode you get the Linux x86 syscall ABI, which
gives you file I/O via four words:

| Word              | Signature                   | Syscall     |
|-------------------|-----------------------------|-------------|
| `bama-ngunnga`    | `( path flags -- fd )`      | `open(2)`   |
| `bama-balkal`     | `( fd buf n -- written )`   | `write(2)`  |
| `bama-babaji`     | `( fd buf n -- read )`      | `read(2)`   |
| `bama-nandal`     | `( fd -- ret )`             | `close(2)`  |

The entry-point word of a `--linux` program is `jakalbaku-warri`.

A minimal round-trip — write bytes to a file, read them back, print them —
lives at [`examples/nganjal.kuku`](examples/nganjal.kuku):

```kuku
wararra wararra-path      "/tmp/kaday.txt" 0  nandal
wararra wararra-message   "kaday, bama! from kuku.\n" 0  nandal
wararra wararra-message-n   24  nandal
wararra wararra-n  0 0 0 0  nandal
wararra-bss wararra-buf  256

balkalaway jakalbaku-warri
    # WRITE: open(path, O_WRONLY|O_CREAT|O_TRUNC), write, close
    wararra-path 0x241 bama-ngunnga
    kujil wararra-message wararra-message-n @ bama-balkal wuljil
    bama-nandal wuljil

    # READ: open(path, O_RDONLY), read, close, echo to stdout
    wararra-path 0 bama-ngunnga
    kujil wararra-buf 256 bama-babaji
    wararra-n !
    bama-nandal wuljil
    1 wararra-buf wararra-n @ bama-balkal wuljil
kunbayn
```

Build and run it:

```
$ ./kuku-bama/ngunnga --linux nganjal examples/nganjal.kuku
$ ./nganjal
kaday, bama! from kuku.

$ xxd /tmp/kaday.txt
00000000: 6b61 6461 792c 2062 616d 6121 2066 726f  kaday, bama! fro
00000010: 6d20 6b75 6b75 2e0a                      m kuku..
```

The same program is what the Kuku compiler itself uses internally to read
`.kuku` source files and write out ELF kernels — just on a much larger
scale.  Note that the bare-metal kernel (the one you boot over SSH) does
**not** yet have a filesystem; that's a separate project after SSH-in-Kuku.

## Best practices

- **Every identifier you introduce must appear in `docs/dictionary.yaml`.**
  The compiler does not enforce this (it cannot read YAML yet), but the
  project convention is strict.  Multi-word names are hyphenated:
  `yanyil-balkal-jina` (see-tell-byte) is a legitimate name.
- **Keep the data stack balanced across every branch of a `yala`/`yinya`
  pair.**  Both the then- and else-arms must finish at the same stack
  depth, or subsequent code will reference garbage.  This is by far the
  most common source of bugs in stack code.
- **When a helper needs scratch storage, give it its OWN static variable.**
  `tmp-a` / `tmp-b` / `tmp-i` / `tmp-j` / `tmp-k` are shared across the
  compiler and get clobbered by sub-calls.  The compiler learned this
  painfully; the `streq-p` / `streq-q` / `ch-size` / `shift` / `file-size`
  variables exist to keep specific pieces of state safe.
- **Big zero buffers go in `wararra-bss`, not `wararra ... maru-maru N
  nandal`.**  The former costs nothing in file size; the latter emits real
  zero bytes into the binary.
- **After changing the compiler** (`kuku-bama/ngunnga.kuku`), rebuild with
  the current binary and confirm the fixed point before committing:

  ```
  ./kuku-bama/ngunnga --linux /tmp/new kuku-bama/ngunnga.kuku
  cmp kuku-bama/ngunnga /tmp/new && mv /tmp/new kuku-bama/ngunnga
  ```

  If `cmp` disagrees on the first pass, compile twice — the second pass
  should converge.

---

## Publishing a Docker image

There is no `Dockerfile` in the repo — that's by design.  Build and push from
a one-shot recipe piped into `docker build -f -` so nothing lands in the
working tree.

### 1. One-time setup

```bash
# create a Docker Hub account at https://hub.docker.com/signup
docker login                           # username + PAT recommended over password
export DH=thomasdavis      # your namespace, e.g. "thomasbeamible"
```

(For `docker login`, generate a Personal Access Token at
https://hub.docker.com/settings/security rather than using your password.)

### 2. Build the image (nothing gets written to the repo)

```bash
cd /path/to/abo
# rebuild the kernel first so the image has the latest bytes
./kuku-bama/ngunnga yalanji.elf kuku/*.kuku

printf 'FROM debian:stable-slim
RUN apt-get update && apt-get install -y --no-install-recommends qemu-system-x86 \\
    && rm -rf /var/lib/apt/lists/*
COPY yalanji.elf /yalanji.elf
CMD ["qemu-system-i386","-kernel","/yalanji.elf","-display","none","-serial","mon:stdio","-no-reboot"]\n' \
    | docker build -f - -t "$DH/yalanji:latest" .
```

### 3. Smoke-test the image locally

```bash
docker run --rm -it "$DH/yalanji:latest"
# type commands: kaday, binal, jalngka, kunbayn
# exit with Ctrl-A X
```

### 4. Push

```bash
docker push "$DH/yalanji:latest"
docker tag  "$DH/yalanji:latest" "$DH/yalanji:$(date +%Y%m%d)"
docker push "$DH/yalanji:$(date +%Y%m%d)"     # keep a dated snapshot too
```

Anyone can now run the OS with:

```bash
docker run --rm -it $DH/yalanji
```

---

## Hosting a public SSH playground on a Digital Ocean droplet

The plan: put a tiny Linux droplet on the public internet, create a demo
user whose login shell IS the Kuku OS running under QEMU.  When someone
SSHes in, they drop straight into the Yalanji shell; when they run
`kunbayn` (or Ctrl-A X), the SSH session ends.

Nothing in Kuku runs a real SSH server — OpenSSH on the droplet handles the
transport and hands the serial stream to our kernel.  Implementing SSH
inside Kuku itself is a separate, much larger project (see *Roadmap*).

### 1. Create the droplet

Requires the `doctl` CLI and an API token (`doctl auth init`).  A cheap
`s-1vcpu-512mb-10gb` droplet ($4/month at time of writing) is plenty —
we only run QEMU with a 5 KB kernel.

```bash
# list regions and pick one near you
doctl compute region list

# upload your SSH public key (skip if you already have it up there)
doctl compute ssh-key import admin-key --public-key-file ~/.ssh/id_ed25519.pub

# grab the fingerprint
FP=$(doctl compute ssh-key list --format FingerPrint --no-header | head -1)

# create the droplet
doctl compute droplet create yalanji-demo \
    --region nyc1 \
    --size s-1vcpu-512mb-10gb \
    --image debian-12-x64 \
    --ssh-keys "$FP" \
    --wait

IP=$(doctl compute droplet get yalanji-demo --format PublicIPv4 --no-header)
echo "droplet at $IP"
```

### 2. Provision it

```bash
ssh root@$IP <<'REMOTE'
set -eux
apt-get update
apt-get install -y --no-install-recommends qemu-system-x86 docker.io

# pull the image (replace thomasdavis)
docker pull thomasdavis/yalanji:latest

# create a demo user whose login shell = QEMU
useradd -m -s /usr/local/bin/yalanji-shell yalanji
echo 'yalanji:yalanji' | chpasswd                 # password: yalanji

cat >/usr/local/bin/yalanji-shell <<'SHELL'
#!/bin/sh
exec docker run --rm -i thomasdavis/yalanji
SHELL
chmod +x /usr/local/bin/yalanji-shell

# allow password SSH for this account only
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
cat >>/etc/ssh/sshd_config <<'SSHCFG'

Match User yalanji
    PasswordAuthentication yes
    PermitTTY yes
    X11Forwarding no
    AllowTcpForwarding no
    MaxAuthTries 3
SSHCFG
systemctl reload ssh
REMOTE
```

### 3. Publish the connect string

Anyone with an SSH client can now play with Yalanji:

```
ssh yalanji@$IP
# password: yalanji
```

Include that line, with the real IP, in your README so people can try it.

### 4. Protect against abuse

- Install `fail2ban` to rate-limit brute-force attempts:
  `apt-get install -y fail2ban` (it auto-enables for sshd).
- Keep the droplet's root account key-only (`PasswordAuthentication no`
  globally + `Match User yalanji` override as above).
- Consider putting the whole `yalanji-shell` command in a systemd unit with
  `MemoryMax=64M` and `CPUQuota=25%` so a misbehaving QEMU cannot starve
  the host.

---

## Roadmap: SSH *inside* Kuku

Serving the OS over OpenSSH (as above) is a one-afternoon job.  Putting an
SSH server *inside* the Kuku OS itself is genuinely months of work.  Rough
shape of the road:

| # | Milestone                                                | Approx. Kuku lines | Status |
|---|----------------------------------------------------------|--------------------|--------|
| 0 | 32-bit port I/O primitives (`mana-baral-jirray`, `daya-baral-jirray`) | +2 | ✅ done |
| 1 | PCI config-space probe (`wari` shell command)            | ~80                | ✅ done — virtio-net detected at bus 0 slot 3 |
| 2a| Virtio-net legacy device init (status handshake + MAC)   | ~130               | ✅ done |
| 2b| Virtio-net virtqueue rings + one TX / one RX frame       | ~500               | ☐ next |
| 3 | Ethernet framing + ARP reply                             | ~400               | ☐      |
| 4 | IPv4 + ICMP echo ("Kuku replies to ping")                | ~800               | ☐      |
| 5 | TCP (state machine, retransmit, window)                  | ~3000              | ☐      |
| 6 | Cryptography: SHA-256, AES-GCM, Ed25519, Curve25519      | ~4000              | ☐      |
| 7 | SSH protocol (RFC 4250-4254)                             | ~3000              | ☐      |

All told, something like 12-15 thousand lines of pure Kuku on top of what we
have now.  The OS will also need a heap allocator, a basic scheduler, a
timer, and an interrupt system — none of which currently exist.

### Where we are (milestones 1 and 2a)

The `wari` shell command now:

1. Walks every slot on PCI bus 0 (reads config dwords from ports
   `0xCF8`/`0xCFC`) and prints what it finds.
2. Locates the virtio-net device (vendor `0x1AF4`) and reads its BAR0
   (I/O-space base).
3. Drives the device through the full legacy init sequence —
   `0 → ACK → ACK|DRIVER → features → ACK|DRIVER|DRIVER_OK` — by writing
   to the status register at `BAR0 + 0x12`.
4. Reads the 6-byte MAC from device-specific config at `BAR0 + 0x14`.

Sample run on the public demo:

```
yalanji> wari
wari: baral binal.
    jina=00000000  kuku=12378086
    jina=00000001  kuku=70008086
    jina=00000002  kuku=11111234
    jina=00000003  kuku=10001AF4  <-- virtio
    bar=0000C000   status=07
    mac =52:54:00:12:34:56
wari: kunbayn.
```

`status=07` = `ACK | DRIVER | DRIVER_OK` — the NIC is now owned by our
(Kuku) driver.  It hasn't sent a single packet yet, but the device is
ready to have queues attached.

### Where we're going (milestone 2b)

The next chunk is **virtqueue rings**: allocate page-aligned memory for
descriptors + avail + used rings, write the queue PFN to the device, pack
a descriptor pointing at a 14-byte Ethernet frame, bump the avail index,
ring the doorbell, poll the used ring.  Unlocks "Kuku sends one ARP
request" — a concrete observable result on `tcpdump -i <tap>`.

**The honest answer is:** use OpenSSH on the droplet for years to come and
treat SSH-in-Kuku as a long multi-year research goal.  Today's milestone is
"Kuku OS knows a NIC exists."  Tomorrow's is "it can send one packet."

---

## On bootstrapping, English, and honesty

Every programming language ever created had its first compiler written in
*something else*.  C's was written in B.  Rust's in OCaml.  Go's in C.  Ours
in Python.  That seed is gone now, but its shadow lives in the bytes of
`kuku-bama/ngunnga` — this is true of every self-hosted language.

English lives only in `docs/`.  C keywords like `int` and `void` do not
appear anywhere because there is no C.  Assembly mnemonics like `mov` and
`push` do not appear anywhere because there is no assembler — the compiler
emits raw x86 bytes straight from Kuku.  Only two classes of non-Kuku tokens
exist in the binaries: **0s and 1s** (the electrical patterns the CPU
understands) and **ASCII in user-facing strings** (so you can type on your
keyboard).

---

## Credits and acknowledgements

Kuku Yalanji belongs to the Kuku Yalanji people of Far North Queensland,
Australia.  The dictionary bundled at `docs/dictionary.yaml` is sourced
verbatim from
[australia/mobtranslate.com](https://github.com/australia/mobtranslate.com).
This project is a research artifact and is not an authoritative reference
for the language — for learning or speaking Kuku Yalanji, go to the people
and their teachers.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the one hard rule (every
identifier must be a real Kuku Yalanji word from
`docs/dictionary.yaml`), the self-host verification step, and what
is and isn't welcome.

## License

Source code is [MIT-licensed](LICENSE).  The bundled dictionary at
`docs/dictionary.yaml` is sourced from
[australia/mobtranslate.com](https://github.com/australia/mobtranslate.com)
and retains its upstream terms; cultural stewardship of Kuku Yalanji
belongs to the Kuku Yalanji people.
