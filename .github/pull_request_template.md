### Summary

<!-- One or two sentences. -->

### Kuku Yalanji words introduced or used

<!-- List any new identifiers, confirm they're in docs/dictionary.yaml. -->

### Checklist

- [ ] Every new identifier is a real word in `docs/dictionary.yaml`.
- [ ] No C / assembly / English keywords in `.kuku` source.
- [ ] `./kuku-bama/ngunnga --linux /tmp/new kuku-bama/ngunnga.kuku && cmp kuku-bama/ngunnga /tmp/new` passes locally.
- [ ] `./kuku-bama/ngunnga yalanji.elf kuku/*.kuku` produces a kernel that boots in QEMU and greets with `kaday, bama`.
- [ ] If this is a compiler change, the `kuku-bama/ngunnga` binary is updated in the same commit.

### Related issue / roadmap milestone

<!-- Fixes #NN.  Or: advances milestone 2b (virtqueue rings). -->
