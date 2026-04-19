# Project rule: no shortcuts

Everything in this project must be written from scratch in Kuku. Do not
reach for host-side tools to stand in for capabilities we don't have
yet. If something needs to exist and Kuku can't do it, the next step is
to teach Kuku to do it — by adding a primitive to
`kuku-bama/ngunnga.kuku`, self-hosting through it, and building the
feature from that primitive in a `.kuku` file.

Concretely, **do not**:

- use `tectonic` / `pdflatex` / `xelatex` / any TeX distribution to turn
  Kuku-emitted LaTeX into a PDF. The PDF generator has to be written in
  Kuku.
- wrap the Kuku HTTP server in a reverse proxy (nginx, caddy), TLS
  terminator, or even a systemd-level connection multiplexer. TLS, HTTP
  parsing, and request routing all have to be Kuku.
- use `openssh-server` to front the Yalanji shell for the public demo.
  The long-running roadmap item is SSH-in-Kuku; until it exists, the
  sshd dependency is a debt, not an endpoint.
- use `docker`, QEMU on the host for anything beyond kernel development,
  or `busybox` / `/bin/sh` / coreutils for demo utilities. Shell
  commands shown off in the demo must be Kuku ELFs.
- use `curl`, `wget`, `python3 -m http.server`, `nc`, or any other
  host-side tool to make a feature look like it works. Tests should
  exercise the Kuku binary directly.
- use `libm`, `libc`, the C toolchain, an external assembler, or a
  linker. The compiler already emits raw ELFs; every future feature
  extends the same path.
- use `gcc`, `clang`, or any non-Kuku compiler to produce an artefact
  that ends up in the repo's build graph or the demo.

The rule applies to auxiliary artefacts too — the LaTeX paper, the
website's HTML response, the web server's connection handler, the SSH
server, the TCP stack, the network card driver, a filesystem, a heap
allocator. All of them are written in Kuku.

Systemd-level wrapping on the droplet (running the Kuku ELF as a
service, giving it `CAP_NET_BIND_SERVICE`, restart-on-failure) is the
only external scaffolding currently allowed, and only because init is
the operating system's job and we haven't written our own yet. When
kuku-os can boot on the droplet directly, systemd goes too.

If a task in this repo appears to need a shortcut, the correct move is
to stop, push the task back onto the roadmap, and first add the
missing Kuku primitive or library. Do not ship the shortcut.
