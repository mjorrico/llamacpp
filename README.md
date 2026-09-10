# llamacpp

A llama.cpp server on the local GPU, exposed on port **8199** and on the
`llama_network` docker network as the host `llama`.

## Start it

Two profiles share the GPU and the port, so exactly one runs at a time.
Neither is a default profile — a bare `docker compose up` starts nothing.

```bash
docker compose --profile llm   up -d --remove-orphans   # chat / completions
docker compose --profile embed up -d --remove-orphans   # embeddings
```

`--remove-orphans` is what stops the other profile: a service whose profile
isn't active counts as an orphan, so swapping is a single command.

`.env` must define `LLAMA_API_KEY`. Compose refuses to start without it, on
purpose — llama.cpp reads an empty key list as "no auth" and would serve
every endpoint unauthenticated.

```bash
curl -s localhost:8199/health
curl -s localhost:8199/v1/models -H "Authorization: Bearer $LLAMA_API_KEY"
```

`/health` and the web UI are open; everything else needs the bearer token.

## Change the model

1. Put the `.gguf` in `models/` — see [models/README.md](models/README.md).
2. Edit the `-m /models/<file>` flag of the service you're changing:
   `llama-server` (profile `llm`) or `llama-embed` (profile `embed`).
3. Restart that profile with the command above.

## Common mistakes

**`-c` is the total context, split across `-np` slots.** Each request gets
`-c / -np` tokens, not `-c`. The `llm` profile's `-c 65536 -np 8` is 8K per
slot — raising `-np` for more concurrency silently shrinks every slot's
window unless you raise `-c` to match.

**`-ub` must be at least your longest embedding input.** A pooled sequence
has to fit in one ubatch, so the `embed` profile pairs `-ub 2048` with its
2048-token slots. Longer inputs fail rather than truncate.

**Swapping profiles without `--remove-orphans`** leaves the old container
holding the GPU and port 8199; the new one won't come up.

**Changing `container_name` doesn't change the address.** Consumers reach
this stack at `http://llama:8080` via the network alias, which both profiles
share so an llm/embed swap needs no client change. The `llama_network` name
is pinned in compose for the same reason — without it compose would create
`llamacpp_llama_network` and external `external: true` consumers would fail
to find it.

**`-ngl 99` assumes the model fits in VRAM.** A bigger quant that doesn't
will fail to load or crawl on CPU. Budget for the KV cache too — it scales
with `-c`, not with the file size.
