# fwdays corpus: this cluster's own service graph

`fwdays-service-graph.json` is a hand-authored array of `{name, description, calls,
called_by}` describing this exact cluster's own topology: every agent, MCP
server, embeddings backend, and vector/graph store from Tasks 1-4, and how
they call each other. It exists because the public `xray-memory` images ship
without CGO/Tree-sitter (`recall snapshot: this build has no Tree-sitter
parser`), and the source repo is private, so building a corpus by parsing real
source code is not available to this fork. `servicemap` -- which builds a
graph+vector snapshot from a hand-supplied call graph instead of parsing
anything -- is.

**The per-node field is `description`, not `summary`.** An earlier version of
this corpus (and of `fwdays-news-corpus.json`) used `summary`; Go's
`json.Unmarshal` silently drops unknown object keys, so that text was never
embedded and never stored -- every node fell back to an auto-synthesized
`"Service X. Calls: ... Called by: ..."` sentence, which is all
`get_graph_node` and `search_graph` had to work with. It looked like working
retrieval (ranking by name/calls text still returns plausible-looking hits)
right up until a query needed the actual descriptive text, at which point
`get_graph_node` came back essentially empty. Confirm the field name against
the tool itself before trusting a corpus built this way -- the CLI's own
`-h` output doesn't mention per-node fields beyond `{name, calls, called_by}`
at all; `description` only turned up by grepping the binary's JSON struct
tags (`strings <the xray-memory binary> | grep 'json:"description"'`).

## Rebuilding the snapshot

Run `xray-memory servicemap` in-cluster, pointed at an embeddings endpoint
that's already running (`llama-cpp-embeddings.llama-cpp:8090`, the same
nomic-embed-text-v1.5 backend `qdrant-mcp` uses):

```bash
kubectl create configmap xray-servicemap-input \
  --from-file=fwdays-service-graph.json=corpus/fwdays-service-graph.json

kubectl run xray-servicemap-build --restart=Never \
  --image=ghcr.io/den-vasyliev/abox/xray-memory:v1.23.66-61e4eaa \
  --overrides='{"spec":{"volumes":[{"name":"input","configMap":{"name":"xray-servicemap-input"}},{"name":"out","emptyDir":{}}],"containers":[{"name":"build","image":"ghcr.io/den-vasyliev/abox/xray-memory:v1.23.66-61e4eaa","command":["/ko-app/xray-memory","servicemap","-in","/input/fwdays-service-graph.json","-label","fwdays","-snapshot-dir","/out","-embedding-endpoint","http://llama-cpp-embeddings.llama-cpp:8090","-embedding-model","nomic-embed-text","-embedding-dims","256","-summary","abox+fwdays cluster topology: agents, MCP servers, embeddings backends, vector and graph stores, and how they call each other"],"volumeMounts":[{"name":"input","mountPath":"/input"},{"name":"out","mountPath":"/out"}]}]}}'

kubectl wait --for=condition=Ready pod/xray-servicemap-build --timeout=60s
kubectl cp xray-servicemap-build:/out/fwdays.graph.gob.gz \
  images/xray-memory-maps-fwdays/maps/fwdays.graph.gob.gz

kubectl delete pod xray-servicemap-build
kubectl delete configmap xray-servicemap-input
```

`-embedding-dims 256` and `-embedding-model nomic-embed-text` must match
`releases/fwdays-xray-memory.yaml`'s top-level `embeddings.model`/`embeddings.dims`
(both left at chart defaults, which are exactly these values) -- dims is a
hard fingerprint check at load (a mismatch skips the map), a model name
mismatch only warns.

Then rebuild and push the image: `gh workflow run "Build fwdays xray-memory
maps image"`, bump the tag in both the workflow and the HelmRelease, and cut a
release (`make push`) so Flux picks it up.

## Known limitation worth carrying into the ADR

The query embedder is the same `nomic-embed-text-v1.5` that ADR 0001/0003
measured tokenising Ukrainian near character-level. A smoke test before this
was wired into the chart confirmed it: an English query about which service
serves bge-m3 embeddings scored the right node at 0.91; the Ukrainian
equivalent scored its best (and not fully correct) hit at 0.43. This mirrors
Task 3/4's finding exactly, just for a different corpus -- xray-memory's own
embedding model choice is a separate axis from the one ADR 0001 already
decided for qdrant-mcp, and inherits the same weakness.

## The second map: ukrnews (Task 5 follow-up)

`fwdays-news-corpus.json` is the same `servicemap` input shape (`{name,
description, calls, called_by}`, edges left empty -- these are standalone
articles, not a call graph), built from real Ukrainian news prose rather than
a hand-authored description. The source data and the fetch script live in the
fwdays-harness-engineering repo, not here: `scripts/news-corpus/fetch_rss.py`
pulls RSS description text from a curated set of nv.ua feeds across many
topics, dedupes by guid, and writes `corpus.json`; this file is that same data
reshaped for `servicemap`, copied in so the map can be rebuilt from this repo
alone.

Rebuild the same way as `fwdays`, just pointed at this file and a different
label:

```bash
kubectl create configmap xray-news-servicemap-input \
  --from-file=servicemap-input.json=corpus/fwdays-news-corpus.json
# ...same servicemap invocation as fwdays, with -label ukrnews -in that file...
kubectl cp xray-servicemap-build:/out/ukrnews.graph.gob.gz \
  images/xray-memory-maps-fwdays/maps/ukrnews.graph.gob.gz
```

**73 real-text nodes in one `servicemap` call can exceed the tool's embedding
request timeout** (a hardcoded client-side deadline, not a flag -- `context
deadline exceeded (Client.Timeout exceeded while awaiting headers)`), because
this corpus's nodes carry actual article text rather than the couple of words
`fwdays`'s auto-synthesized `"Service X. Calls: ..."` sentences would have
needed. Splitting the input into smaller batches does NOT help: each
`servicemap` run OVERWRITES `<label>.graph.gob.gz` with only that run's nodes,
it does not merge across separate invocations (`cache_hits` only dedupes
identical text within a run's own embedder cache, not across runs) -- running
5 batches of ~15 nodes each left only the last batch's 13 nodes in the file.
What worked: temporarily raising `llama-cpp-embeddings`' CPU (`kubectl patch
deployment llama-cpp-embeddings -n llama-cpp ...` to 6 CPU / 4 CPU request),
which brought all 73 nodes in as one call under 15 seconds, then reverting it
back to the shipped 500m/2 afterward -- this is a one-off build-time need, not
a standing resource change.

Both `fwdays.graph.gob.gz` and `ukrnews.graph.gob.gz` ship in the same maps
image and are served by the same `xray-memory-fwdays` pod as two separate
projects -- that's what xray-memory's multi-map design is for, so this needed
no second Deployment, just a bigger image and `memory-agent-fwdays`'s prompt
updated to say which map answers which kind of question.

## Why also index into qdrant (qdrant-mcp-news)

The same 73 articles are also indexed into qdrant under `abox-news-bge-m3`
(`releases/fwdays-news-mcp-server.yaml`, `bge-m3` via the existing
`bge-m3-embeddings` backend) so xray-memory's own engine and qdrant+bge-m3 can
be Recall@k-compared on the identical corpus -- a different axis from Task
3/4's "which embedding model" comparison: this one holds the corpus and the
embedding model choice, and varies the retrieval engine itself.
