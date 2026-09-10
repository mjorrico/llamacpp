# models/

GGUF weights, bind-mounted into the containers at `/models`. The `.gguf` files
are gitignored, so download them here on a fresh machine.

## Install the hf CLI

```bash
curl -LsSf https://hf.co/cli/install.sh | bash
```

## Download a model

Run from this directory — `--local-dir .` writes the real file here instead of
symlinking into `~/.cache/huggingface`:

```bash
hf download hf://SuperPauly/harrier-oss-v1-0.6b-gguf/harrier-oss-v1-0.6B-BF16.gguf --local-dir .
```

Then point the `-m` flag in `../compose.yml` at the new filename and restart.
