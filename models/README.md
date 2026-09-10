# models/

GGUF weights served by llama.cpp. This directory is bind-mounted into both
containers at `/models` (see `../compose.yml`), so a file dropped here is
visible to the server as `/models/<filename>` without rebuilding anything.

The `.gguf` files themselves are not tracked — only this README is. Each one
has to be downloaded on a fresh machine.

## Install the Hugging Face CLI

```bash
curl -LsSf https://hf.co/cli/install.sh | bash
```

That puts `hf` in `~/.local/bin`. For gated or private repos, authenticate
first with `hf auth login` (or export `HF_TOKEN`); public GGUF repos need
nothing.

## Download a model

Run from this directory. `--local-dir .` is what makes `hf` write the real
file here instead of leaving it in `~/.cache/huggingface` behind a symlink:

```bash
cd models
hf download hf://SuperPauly/harrier-oss-v1-0.6b-gguf/harrier-oss-v1-0.6B-BF16.gguf --local-dir .
```

The `hf://<user>/<repo>/<path>` form pulls a single file. The equivalent
positional form is `hf download <user>/<repo> <filename> --local-dir .`, and
`--include '*Q4_K_M*'` works when you want to match a quant by pattern rather
than name it exactly.

Add `--dry-run` first to confirm you picked the file you meant — quant repos
often hold a dozen variants of the same weights and they are multi-GB each.

Multi-part models (`...-00001-of-00003.gguf`) must all be downloaded; point
llama.cpp at the first shard and it picks up the rest.

## Wire it into the server

The filename is hard-coded in `../compose.yml`, so edit the `-m` flag of the
profile you are changing:

- `llama-server` (profile `llm`) — chat/completion model
- `llama-embed` (profile `embed`) — embedding model, also needs `--embeddings`

Then restart that profile:

```bash
cd ..
docker compose --profile llm up -d --remove-orphans
```

Only one profile runs at a time: they share the GPU and port 8199.

### Things that usually need adjusting with a new model

- `-ngl 99` offloads every layer to the GPU. A model that no longer fits will
  fail to load or fall back to painful CPU speed — check `nvidia-smi` against
  the file size plus KV cache before assuming a bigger quant fits.
- `-c` is the total context across all slots, split `-np` ways. Raising `-np`
  without raising `-c` shrinks each slot's window.
- For embedding models, `-ub` must be at least the longest input in tokens,
  since a pooled sequence has to fit in one ubatch.
- `-a <alias>` sets the model name reported by `/v1/models` and expected in
  request bodies; the embed service pins it to `harrier-270m` so clients do
  not have to track filenames.

## Verify

```bash
curl -s localhost:8199/health
curl -s localhost:8199/v1/models -H "Authorization: Bearer $LLAMA_API_KEY"
```

`../tests/ask.py` and `../tests/embed_throughput.py` exercise the two profiles
end to end.

## Choosing a quant

`Q4_K_M` is the usual default — roughly half the size of `Q8_0` with little
quality cost. `Q8_0` is near-lossless and worth it for small models where the
VRAM is cheap (as with the 2.6B here). `BF16` is unquantized: use it for
converting or benchmarking, rarely for serving.

Currently present:

| File | Role |
| --- | --- |
| `Qwen3.5-4B-Q4_K_M.gguf` | chat model for the `llm` profile |
| `harrier-oss-v1-270M-Q4_K_M.gguf` | embedding model for the `embed` profile |
| `LFM2.5-2.6B-Q8_0.gguf` | alternate chat model, not currently referenced |
