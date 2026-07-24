# vLLM KV Events canonical form

KV Events are vLLM's mechanism for publishing real-time notifications about KV cache block lifecycle events over ZMQ pub/sub. External consumers (Dynamo router, llm-d-kv-cache-manager) use them to track which blocks are cached, evicted, or cleared.

## Event types

All inherit from `KVCacheEvent` in `vllm/distributed/kv_events.py`.

### BlockStored

Emitted when blocks are cached. The richest event type:

```python
class BlockStored(KVCacheEvent):
    block_hashes: list[ExternalBlockHash]        # One hash per block
    parent_block_hash: ExternalBlockHash | None   # Preceding block hash (prefix chain)
    token_ids: list[int]                          # Token IDs covered
    block_size: int                               # Tokens per block
    medium: str | None                            # "GPU", "CPU", "FS", "OBJ"
    group_idx: int | None                         # KV cache group index
    kv_cache_spec_kind: str | None                # Semantic cache type
    kv_cache_spec_sliding_window: int | None
    locality: str | None                          # "LOCAL" or "REMOTE"
    lora_id: int | None                           # Deprecated
    lora_name: str | None                         # LoRA adapter name
    extra_keys: list[tuple[Any, ...] | None] | None = None  # Per-block extra hash keys
```

### BlockRemoved

Emitted when blocks are evicted:

```python
class BlockRemoved(KVCacheEvent):
    block_hashes: list[ExternalBlockHash]    # Hashes of removed blocks
    medium: str | None                       # Storage medium
    group_idx: int | None                    # KV cache group index
    locality: str | None                     # "LOCAL" or "REMOTE"
```

### AllBlocksCleared

Terminal event emitted when all cache blocks are cleared (e.g., engine reset):

```python
class AllBlocksCleared(KVCacheEvent):
    pass  # No fields beyond the tag
```

## Wire format

Events are wrapped in an `EventBatch` envelope:

```json
{
  "ts": 1.0,
  "events": [
    {
      "type": "BlockStored",
      "block_hashes": [4291203, 1837291, 9104857],
      "parent_block_hash": null,
      "token_ids": [0, 1, 2, 3],
      "block_size": 64,
      "medium": "GPU",
      "group_idx": 0,
      "kv_cache_spec_kind": "full_attention",
      "locality": "LOCAL"
    }
  ]
}
```

## Key supporting types

| Type | Definition |
|------|-----------|
| `ExternalBlockHash` | `bytes \| int` union (default is bytes / SHA-256) |
| `KVCacheSpecKind` | Enum: `FULL_ATTENTION`, `MLA_ATTENTION`, `SLIDING_WINDOW`, etc. |
| `Locality` | `LOCAL` or `REMOTE` relative to the publishing instance |
| `EventBatch` | Top-level envelope: timestamp + list of events |

## Source files

| File | Purpose |
|------|---------|
| `vllm/distributed/kv_events.py` | Core event types, aggregator, publisher |
| `vllm/config/kv_events.py` | Configuration dataclass |
| `vllm/v1/kv_cache_interface.py` | `KVCacheSpecKind` enum |
| `vllm/v1/core/kv_cache_utils.py` | `ExternalBlockHash` type |

## Schema evolution notes

- `locality` field added in PR #42892 (array→map encoding for forward compatibility)
- Wire format switched from array to map encoding in PR #42892

## Source

- Session ses_06baf2392ffe7BXoPnFTG2MiZV (2026-07-24): Code inspection of vllm-project/vllm KV Events definitions.
