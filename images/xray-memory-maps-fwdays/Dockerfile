# A pre-built xray-memory snapshot, packaged as an OCI image volume so
# releases/fwdays-xray-memory.yaml can mount it read-only with
# snapshots.source=image -- no PVC, no seed initContainer, no CGO build.
#
# maps/fwdays.graph.gob.gz is built with `xray-memory servicemap` from
# corpus/fwdays-service-graph.json (a hand-authored description of this
# cluster's own agents/MCP servers/backends), embedded through
# llama-cpp-embeddings. See corpus/README.md for how to rebuild it.
FROM scratch
COPY maps/*.graph.gob.gz /maps/
