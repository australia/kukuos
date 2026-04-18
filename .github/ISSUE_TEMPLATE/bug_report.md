---
name: Bug report
about: Something in the OS, compiler, or demo droplet is broken
title: "[bug] "
labels: bug
---

### What happened

<!-- One or two sentences. -->

### How to reproduce

<!-- Exact commands.  Include the kernel output you saw. -->

```
$ ./kuku-bama/ngunnga yalanji.elf kuku/*.kuku
$ qemu-system-i386 -kernel yalanji.elf -display none -serial mon:stdio -no-reboot
... (paste output) ...
```

### Expected behaviour

<!-- What should have happened instead. -->

### Environment

- Host OS:
- QEMU version (`qemu-system-i386 --version`):
- Commit SHA:
- Did the self-host check pass?  (`./kuku-bama/ngunnga --bama /tmp/new kuku-bama/ngunnga.kuku && cmp kuku-bama/ngunnga /tmp/new`)
