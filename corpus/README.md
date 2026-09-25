# fwdays corpus: this cluster's own service graph

`fwdays-service-graph.json` is a hand-authored array of `{name, summary, calls,
called_by}` describing this exact cluster's own topology: every agent, MCP
server, embeddings backend, and vector/graph store from Tasks 1-4, and how
they call each other. It exists because the public `xray-memory` images ship
without CGO/Tree-sitter (`recall snapshot: this build has no Tree-sitter
parser`), and the source repo is private, so building a corpus by parsing real
source code is not available to this fork. `servicemap` -- which builds a
graph+vector snapshot from a hand-supplied call graph instead of parsing
anything -- is.

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
