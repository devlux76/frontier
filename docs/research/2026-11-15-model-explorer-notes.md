# model-explorer: what is worth taking (2026-11-15)

Source: google-ai-edge/model-explorer (upstream, Apache-2.0, commit f912c54 of 2026-11-12), read from its source tree. We did not run or install it. Our GitHub fork will be deleted in the cleanup; these notes are what we keep.

## What it is
A hierarchical graph viewer for neural-network models (TFLite/LiteRT, TF, TFJS, MLIR, PyTorch ExportedProgram; ONNX via community adapters). The Python server converts a model file to a JSON graph; an Angular UI lays it out in a web worker and draws it with three.js. Nothing in the loop is exotic. Five mechanisms are worth keeping.

## 1. Layout only what is expanded
Groups are collapsed by default. Dagre (layered layout) runs only on expanded groups, from the deepest expanded group back to the root (`src/ui/src/components/visualizer/worker/graph_expander.ts`, `expandFromDeepestGroupNodes`; `worker/graph_layout.ts`). Layout runs in a worker (`worker/worker.ts`, `worker_service.ts`), so the UI thread stays free.
**For us:** Trailblazer's graph view should lay out one module or file on demand, with collapsed packages as group nodes. Never lay out the whole repo at once.

## 2. Namespace-derived hierarchy
Each op's namespace (`a/b/c`) yields ancestor group nodes `a/b`, `a`, root (`worker/graph_processor.ts`, ancestor loop around line 209). A `;`-separated namespace keeps its last component.
**For us:** Trailblazer declaration paths (`crate::mod::fn`) map directly onto this nsParent tree. The group hierarchy costs no extra modelling.

## 3. Splitting huge flat layers
If a group has more children than a threshold (`src/ui/src/components/visualizer/common/consts.ts` line 63: 400 local, 1000 external), the processor does a dry layout of the children with fake node sizes, DFS-walks from the layout roots and creates sub-groups (`worker/graph_processor.ts`, `splitLargeGroupNodes`, around line 523). This is the answer to "10k nodes in one layer".
**For us:** a file with thousands of declarations, or a llama.cpp op list with one flat scope, needs this. Trailblazer should apply the same bounded-fanout split before drawing.

## 4. Per-node data overlays (the JSON format)
A node-data provider returns `results` keyed by output tensor name or node id. Each result carries `thresholds` (value upper bound to bgColor/textColor) or `gradient` stops in [0,1] over the min/max of all values (`src/server/package/src/model_explorer/node_data_builder.py`, `ThresholdItem` and `GradientItem`, around lines 50-210). Adapters can also attach per-tensor `quantization` in `inputsMetadata` (`graph_builder.py`, around line 138).
**For us:** a per-op overlay for Moon and our llama.cpp builds. Key = op or tensor name; values = latency, bytes moved, or max abs error against the float reference. Gradient stops show quantization-error hotspots. The format is simple enough to copy as plain JSON.

## 5. Adapter API and quantization display
An adapter subclasses `Adapter` and implements `convert(model_path, settings)` returning `graphs` or `graphCollections` (`src/server/package/src/model_explorer/adapter.py`, lines 32-55). The LiteRT adapter formats each tensor's quantization as an affine formula such as `scale * (q - zp)` and handles blockwise and multi-axis schemes (`src/builtin-adapter/adapters/litert/quantization_formatter.cc` and `.h`).
**For us:** a GGUF adapter that emits the same graph JSON, with each tensor's Q4_0 or Q8 block scale shown as a formula in the info panel. This is the cheapest way to inspect what a quant type actually did to one tensor.

## Also noted
- Edge overlays (`src/ui/src/components/visualizer/common/edge_overlays.ts`): named sets of edges with a colour, shown when a related node is selected, optionally scoped to one graph. For us: Trailblazer "uses" edges as a switchable overlay on a call graph.
- Side-by-side comparison (`common/sync_navigation.ts`): modes `match_node_id`, `visualizer_config`, uploaded or loaded mapping. A mapping entry maps a left node id to a right node id. This is the matching model for comparing two graphs, e.g. f16 vs Q4_0 of the same model: match by tensor name first, and fall back to an explicit map file.
- Repeated-block detection (`worker/identical_groups_finder.ts`): a djb2-style hash over a group's op labels, neighbour labels and edge counts, mod 10000019, groups identical blocks. The hash is label-based and has no collision check, so verify any match structurally before trusting it. Useful for finding repeated transformer blocks.
- Rendering: three.js WebGLRenderer with an orthographic camera (`src/ui/src/components/visualizer/webgl_renderer_threejs_service.ts`), plus rounded-rectangle and edge meshes (`webgl_rounded_rectangles.ts`, `webgl_edges.ts`). Text goes through separate services (`webgl_texts.ts`, `svg_text_renderer_service.ts`).

## Not worth taking
- The Angular UI, the Python CLI and server, the Bazel/pigweed build, the Hugging Face demo.
- Dagre itself: a layered layout library we would pull in as an npm dependency. If we want it, reimplement only the part we use, per the dep-drop rule.
- Google-specific analytics (`ga_service.ts`) and the Colab notebooks.
- The approx. 7k lines of visualizer TypeScript. Take the mechanisms above, not the code.
