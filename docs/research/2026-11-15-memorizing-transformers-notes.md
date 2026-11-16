# Memorizing Transformers (lucidrains PyTorch): what is worth taking (2026-11-15)

Source: lucidrains/memorizing-transformers-pytorch (upstream, MIT), shallow-cloned and read, not run. Paper: Wu et al., "Memorizing Transformers" (ICLR 2022). Our GitHub fork was deleted in the cleanup; these notes are what we keep.

## What it is
A transformer whose attention in designated layers (default: the middle layer) also reads a kNN store of past key/value pairs. Each batch row has its own faiss HNSW index plus a memmap ring buffer holding the (k, v) payloads. The paper's sigmoid gate is replaced in code by one joint softmax over retrieved and local logits (README says so). Cosine-similarity attention with a learned per-head temperature is used in the memory layers. The code is about 850 lines across two files.

## 1. Memory written only on eviction from the recurrent window
`KNNAttention.forward` (`memorizing_transformers_pytorch/memorizing_transformers_pytorch.py` ~L255-267) splits the segment's (k, v): the last `xl_max_memories` stay as XL recurrence, the older part is added to kNN. Retrieval runs before the add, so a segment never retrieves itself.
**For us:** replay memory writes an entry when it falls out of the live context window, not on every turn. Retrieve-then-write in the same step is a cheap guard against self-matching.

## 2. Per-row memory namespaces, cleared on a boundary token
`KNNMemory` keeps one index and one db slice per batch row. `at_batch_indices` scopes which rows are read or written. `clear_memories_on_sos_token_id` resets a row's kNN store whenever that row contains the sos id (`MemorizingTransformer.clear_memory`, ~L429-440).
**For us:** one namespace per job or episode, with an explicit reset marker. Trailblazer and the replay store should key memories by job id, not share one global pool.
**Caveat (code gap):** the XL recurrence state is not cleared by that sos reset. `train.py` (L92-127) resets `xl_memories` by hand, once per context. Any document-boundary reset must clear both stores.

## 3. Joint softmax with a learned temperature, optional zero-init gate
Local and retrieved logits go into one softmax. The temperature is `exp(log(20))` per head, a learned parameter (`KNNAttention.scale`, L190). The Flamingo-style `output_gate` (`tanh`, zero-init, L289-290) exists but is off by default (`gate_output=False`).
**For us:** when grafting memory onto a pretrained local model, start with a zero-init `tanh` gate so the model is unchanged at step 0. Measure the joint form against the gate on the replay eval before choosing.

## 4. Cosine keys and fixed top-k with masking
Queries and keys are L2-normalized (`l2norm`, L34, L220) and the index uses inner product, so it is cosine retrieval. Top-k is fixed (default 32). Empty hits return `-1`, are masked out and their payloads zeroed (`KNNMemory.search`, L247-257).
**For us:** normalize once at write time and use IP in Trailblazer's vector index. Fixed k with a validity mask keeps shapes static for batched inference.

## 5. Eviction is a full reset, not a sliding window
`KNN.add` calls `reset()` once the entry count passes `max_num_entries`, because HNSW cannot delete entries (README says the same). The code comment "expiring old keys automagically" overstates this.
**For us:** a full wipe at cap is not acceptable for replay memory. If we need eviction, keep a ring of smaller indices or rebuild from a durable store.

## 6. Positional mapping (read as a likely bug, untested)
Faiss returns sequential internal ids, and `search` uses them directly as memmap slots (`db[batch, ids % max]`). `db_offsets` keeps counting after a cap reset, but faiss ids restart at 0. So after the first reset, a retrieved id may point at a different slot than the one just written. Only the stats path uses the `ids` array. I have not run this to confirm.
**For us:** store an explicit payload id per vector (faiss `IndexIDMap` or a Trailblazer row id), never a positional slot. Do not copy this indexing.

## Also noted
- Retrieved memories get no positional bias. The `knn_rel_pos_bias` is applied only to local logits. Retrieval is content-only.
- Stored keys and values are detached snapshots taken under the weights at write time. They go stale as weights change; the paper has the same property.
- XL recurrence can be shifted to higher layers (`shift_xl_memories_down`, ERNIE-Doc idea). Cheap to test if we add recurrence to a local model.

## Not worth taking
The enwik8 training script, the joblib threading, the memmap-on-disk default under `./.tmp`, and the HNSW-with-`nprobe` leftovers (`nprobe` is unused and the IVF comment is stale).
