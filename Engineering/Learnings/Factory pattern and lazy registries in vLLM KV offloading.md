---
title: "Factory pattern and lazy registries, worked through vLLM KV offloading"
date: "2026-09-23"
type: "learning"
topic: "design patterns"
source_repo: "vllm-project/vllm"
source_commit: "1ea7c63f4a"
---

# Factory pattern and lazy registries, worked through vLLM KV offloading

A guide to the Factory family of design patterns, then the same ideas traced
through real vLLM code. All code below is copied verbatim from
`vllm-project/vllm` at commit `1ea7c63f4a` (post-v0.29.0 `main`).

---

## Part 1 — The theory

### The problem it solves

Object creation couples the caller to a concrete class. The moment you write:

```python
tier = FileSystemTierManager(root_dir="/mnt/kv_cache", n_read_threads=64)
```

that call site now knows about `FileSystemTierManager` specifically. If you
later support an object store or a peer-to-peer tier, every such call site needs
a branch:

```python
if tier_type == "fs":
    tier = FileSystemTierManager(...)
elif tier_type == "obj":
    tier = ObjectStoreSecondaryTierManager(...)
elif tier_type == "p2p":
    tier = P2PSecondaryTierManager(...)
```

That branch tends to get duplicated, and adding a type means finding every copy.
A factory moves the decision into exactly one place, so the caller asks for "a
secondary tier" and receives whichever concrete class the configuration names.

The value is not "avoiding `new`". It is **who owns the knowledge of which
concrete class to build**. A factory makes that one module's job.

### When it earns its keep

- The concrete type is chosen at runtime, typically from configuration.
- New implementations are expected, and ideally from outside your codebase.
- Construction has real logic: validation, defaults, dependent sub-objects.
- Several objects must be built as a consistent set.

### When it does not

- There is one implementation and no prospect of a second. A factory here is
  indirection with no payoff.
- The caller genuinely needs the concrete type's API. Wrapping construction does
  not help if you immediately downcast.
- Construction is a plain constructor call with no decision to make.

### The variants

**Simple factory** (not in the GoF book, but the most common thing people mean):
one function or classmethod that switches on a value and returns a concrete
instance.

**Factory method** (GoF): a method that subclasses override to decide what to
create. The base class calls it without knowing the answer. Adding a product
means adding a subclass, not editing a switch.

**Abstract factory** (GoF): one object that creates a *family* of related
products guaranteed to work together. The point is the guarantee — you cannot
mix a product from one family with a product from another.

**Registry** (a common refinement of simple factory): the switch is replaced by
a `dict` from name to class, populated at import time. Adding a product becomes
one registration line and zero edits to the factory itself. This is what
"pluggable" usually means in practice.

### The refinement that matters most: lazy registration

A naive registry stores classes:

```python
_registry["fs"] = FileSystemTierManager      # requires importing it first
```

To fill that dict you must import every implementation, which drags in every
implementation's dependencies at startup — S3 clients, RDMA libraries, CUDA
extensions — whether or not anyone selected them.

Storing a **loader function** instead defers the import until lookup:

```python
def loader() -> type[SecondaryTierManager]:
    module = importlib.import_module(module_path)
    return getattr(module, class_name)

_registry[tier_type] = loader
```

Registration then needs only two strings, so the registry module imports
nothing. This is the difference between a registry that is merely tidy and one
that is usable in a project with heavy optional dependencies.

### The second refinement: the unregistered escape hatch

A registry is closed: you can only ask for names someone registered. Adding a
fallback that imports from a caller-supplied module path makes it open, so a
third party can plug in an implementation **without modifying or forking your
code**. The cost is that a typo in a name produces an import error rather than
"unknown type", so the error message has to work harder.

### Trade-offs to state honestly

- **Indirection.** "Where does this object actually come from?" becomes a
  multi-hop question. Jump-to-definition stops working at the factory.
- **Errors move to runtime.** A misspelled name is a startup failure, not a type
  error. Message quality is therefore part of the design, not a nicety.
- **Registration order matters.** Registries populate at import time, so
  whether a name exists depends on what has been imported.
- **Discoverability drops.** The set of valid values lives in a dict, not in a
  type. Listing registered names in the error is close to mandatory.

---

## Part 2 — The pattern in vLLM KV offloading

vLLM's KV-offload stack uses three registries stacked inside each other, plus an
abstract factory. It is a good worked example because the layers are small and
each one answers a different question.

```
KVConnectorFactory        -> which connector?          (kv_connector)
  OffloadingSpecFactory   -> which offloading spec?    (spec_name)
    OffloadingSpec (ABC)  -> manager + worker pair     (abstract factory)
      SecondaryTierFactory-> which storage tier?       (type)
      CachePolicyFactory  -> which eviction policy?    (eviction_policy)
```

A user's YAML drives all of it:

```json
{
  "kv_connector": "OffloadingConnector",
  "kv_connector_extra_config": {
    "spec_name": "TieringOffloadingSpec",
    "eviction_policy": "lru",
    "secondary_tiers": [{"type": "fs", "root_dir": "/mnt/kv_cache"}]
  }
}
```

Every quoted string is a registry key.

### 2.1 The registry, verbatim

`vllm/v1/kv_offload/factory.py`:

```python
class OffloadingSpecFactory:
    _registry: dict[str, Callable[[], type[OffloadingSpec]]] = {}

    @classmethod
    def register_spec(cls, name: str, module_path: str, class_name: str) -> None:
        """Register a spec with a lazy-loading module and class name."""
        if name in cls._registry:
            raise ValueError(f"Connector '{name}' is already registered.")

        def loader() -> type[OffloadingSpec]:
            module = importlib.import_module(module_path)
            return getattr(module, class_name)

        cls._registry[name] = loader

    @classmethod
    def get_spec_cls(cls, extra_config: Mapping[str, Any]) -> type[OffloadingSpec]:
        spec_name = extra_config.get("spec_name", "CPUOffloadingSpec")
        if spec_name in cls._registry:
            spec_cls = cls._registry[spec_name]()
        else:
            spec_module_path = extra_config.get("spec_module_path")
            if spec_module_path is None:
                raise ValueError(f"Unsupported spec type: {spec_name}")
            logger.warning_once(
                "Loading out-of-tree offloading spec. This API is "
                "experimental and subject to change in the future "
                "as we iterate the design."
            )
            logger.info(
                "Loading out-of-tree offloading spec '%s' from '%s'.",
                spec_name,
                spec_module_path,
            )
            spec_module = importlib.import_module(spec_module_path)
            spec_cls = getattr(spec_module, spec_name)
        assert issubclass(spec_cls, OffloadingSpec)
        return spec_cls

    @classmethod
    def create_spec(cls, config: OffloadingConfig) -> OffloadingSpec:
        spec_name = config.extra_config.get("spec_name", "CPUOffloadingSpec")
        spec_cls = cls.get_spec_cls(config.extra_config)
        logger.info("Creating offloading spec with name: %s", spec_name)
        return spec_cls(config)


# Register various specs here.
OffloadingSpecFactory.register_spec(
    "CPUOffloadingSpec", "vllm.v1.kv_offload.cpu.spec", "CPUOffloadingSpec"
)
OffloadingSpecFactory.register_spec(
    "TieringOffloadingSpec",
    "vllm.v1.kv_offload.tiering.spec",
    "TieringOffloadingSpec",
)
```

Every element of the theory is visible here:

- `_registry` maps a **name** to a **loader**, never to a class.
- `register_spec` takes two strings, so this module imports neither spec.
  Selecting `CPUOffloadingSpec` never imports the tiering stack, which pulls in
  object-store and P2P backends.
- Duplicate registration raises immediately rather than silently overwriting.
- `get_spec_cls` falls back to `spec_module_path` for out-of-tree specs, with
  `warning_once` marking the API as experimental.
- `assert issubclass(spec_cls, OffloadingSpec)` is the type check the registry
  gave up by keying on strings. Without it an out-of-tree name could return
  anything.
- The default (`"CPUOffloadingSpec"`) is duplicated in `get_spec_cls` and
  `create_spec` — a small wart worth noticing, since the two can drift.

### 2.2 The abstract factory

`vllm/v1/kv_offload/base.py`:

```python
class OffloadingSpec(ABC):
    """Spec for an offloading connector"""

    @classmethod
    def build_metric_definitions(
        cls, extra_config: dict[str, Any]
    ) -> dict[str, "OffloadingMetricMetadata"]:
        """Return Prometheus metric definitions emitted by this spec."""
        return {}

    def __init__(self, config: OffloadingConfig):
        self.config = config
        self.extra_config = config.extra_config
        ...

    @abstractmethod
    def get_manager(self) -> OffloadingManager:
        """Get an OffloadingManager that will be used
        by the scheduler-side offloading connector to track
        offloaded blocks and manage evictions.
        """
        pass

    @abstractmethod
    def get_worker(self, kv_caches: CanonicalKVCaches) -> OffloadingWorker:
        """Get an OffloadingWorker that handles async KV transfers for this spec.
        """
        pass
```

This is an **abstract factory**, and the reason is architectural rather than
aesthetic. vLLM v1 splits across processes:

- the **scheduler** process needs an `OffloadingManager` — tracks which blocks
  are offloaded, performs lookups, decides evictions, touches no tensors;
- the **worker** processes need an `OffloadingWorker` — moves bytes, holds the
  KV tensors, knows no policy.

Those two must be a matched pair. A CPU-only manager paired with a tiering
worker would be incoherent. The spec is the single object that produces both, so
the pairing cannot be got wrong.

The consuming side, `offloading_connector.py`:

```python
        offloading_config = build_offloading_config(vllm_config, kv_cache_config)
        self._canonical_layout = offloading_config.canonical_layout
        spec = OffloadingSpecFactory.create_spec(offloading_config)

        self.connector_scheduler: OffloadingConnectorScheduler | None = None
        self.connector_worker: OffloadingConnectorWorker | None = None
        if role == KVConnectorRole.SCHEDULER:
            self.connector_scheduler = OffloadingConnectorScheduler(
                spec, vllm_config, kv_cache_config
            )
        elif role == KVConnectorRole.WORKER:
            self.connector_worker = OffloadingConnectorWorker(
                spec, vllm_config, kv_cache_config
            )
```

`OffloadingConnector` mentions no concrete spec, manager, or tier. The same
class is constructed in both processes with a different `role`, and each half
asks the same spec class for its own product. Adding a new offload medium
requires no change to this file.

Note also that `build_metric_definitions` is a **classmethod**. The Prometheus
metrics a deployment exposes are therefore determined by *which spec was
selected*, before any instance exists — a factory deciding a static property of
the system, not just an object.

### 2.3 Factory method: the same call, two answers

`get_manager` is a factory method — the subclasses decide.

`CPUOffloadingSpec.get_manager` (`vllm/v1/kv_offload/cpu/spec.py`):

```python
    def get_manager(self) -> OffloadingManager:
        if not self._manager:
            # store_threshold: how many times a chunk must be offered for
            # storage before it is eligible for CPU offloading.  Values < 2
            # disable filtering (a threshold of 1 equals no filter; 0 is the
            # default).
            store_threshold = int(self.extra_config.get("store_threshold", 0))

            # Maximum entries in the internal tracker's LRU table.
            max_tracker_size = int(self.extra_config.get("max_tracker_size", 64_000))

            self._manager = CPUOffloadingManager(
                num_chunks=self.num_chunks,
                cache_policy=self.eviction_policy,
                ...
```

`TieringOffloadingSpec.get_manager` (`vllm/v1/kv_offload/tiering/spec.py`)
rejects a setting the CPU spec accepts, then builds a whole tier hierarchy:

```python
    def get_manager(self) -> OffloadingManager:
        """Get the TieringOffloadingManager.
        ...
        """
        if not self._manager:
            if int(self.extra_config.get("store_threshold", 0)) >= 2:
                raise ValueError(
                    "store_threshold is not supported for TieringOffloadingSpec"
                )
```

Two observations worth carrying to your own code:

1. **Both memoise** (`if not self._manager`). A factory method called more than
   once must decide whether it is a *getter* or a *creator*. These are getters
   that create on first use; the names say `get_`, which matches.
2. **Validation lives in the product's own factory method.** `store_threshold`
   is meaningful for CPU and meaningless for tiering, so the rejection belongs
   in the tiering subclass, not in a shared config validator that would have to
   know about every spec.

### 2.4 The nested registry, and how config becomes kwargs

Inside `TieringOffloadingSpec.get_manager`, the same pattern recurses:

```python
                primary_kv_view = primary_tier.get_kv_memoryview()
                for i, tier_config in enumerate(self.secondary_tier_configs):
                    tier = SecondaryTierFactory.create_secondary_tier(
                        tier_config, primary_kv_view, self
                    )
                    secondary_tiers.append(tier)
```

`vllm/v1/kv_offload/tiering/factory.py` ends with the line that matters most for
anyone configuring this system:

```python
        return tier_cls(
            offloading_spec=offloading_spec,
            primary_kv_view=primary_kv_view,
            tier_type=tier_type,
            backpressure_detector=bp_detector,
            **config,
        )
```

`config` is the tier's YAML dict with the keys the factory itself consumed
(`type`, `module_path`, `backpressure`) popped off. **Everything remaining is
splatted as keyword arguments into the concrete tier's `__init__`.**

This has two consequences that are easy to miss and important in practice:

- **Adding a constructor parameter to a tier immediately makes it
  configurable**, with no plumbing anywhere. A new `min_blocks_per_load_task:
  int = 32` argument on `FileSystemTierManager.__init__` is settable from YAML
  the moment it exists.
- **An unknown key is a `TypeError` at startup, not a silently ignored
  setting.** `FileSystemTierManager.__init__() got an unexpected keyword
  argument 'min_blcoks_per_load_task'` kills the engine. That is fail-closed,
  and it is the right behaviour: a typo cannot quietly produce a run with
  default settings that you then interpret as a result.

  Contrast with keys read via `self.extra_config.get(...)` in a spec's
  `__init__`, such as `offload_prompt_only` — **those are silently ignored if
  misspelled**, because a dict `get` with a default cannot tell a typo from an
  absent key. The same config file therefore has two different failure modes
  depending on which layer consumes the key. When running A/B experiments,
  verify the rendered arguments rather than trusting the file.

The tier factory's error message also shows the discoverability fix:

```python
            raise ValueError(
                f"Unknown secondary tier type: {tier_type!r}. "
                f"Supported types: {list(cls._registry)}. "
                "For an out-of-tree tier, also set 'module_path'."
            )
```

It lists the registered names and points at the escape hatch. A registry whose
error is just `KeyError: 'fs2'` is hostile.

### 2.5 The same shape a third time

`vllm/v1/kv_offload/cpu/policies/factory.py` repeats it for eviction policies:

```python
    _registry: dict[str, Callable[[], type[CachePolicy]]] = {}

    @classmethod
    def register_cache_policy(
        cls, name: str, module_path: str, class_name: str
    ) -> None:
        """Register a cache policy with a lazy-loading module and class name."""
        if name in cls._registry:
            raise ValueError(f"Cache policy '{name}' is already registered.")

        def loader() -> type[CachePolicy]:
            module = importlib.import_module(module_path)
            return getattr(module, class_name)

        cls._registry[name] = loader
```

Identical structure, different product type. Its docstring even names the
precedent — `"mirrors OffloadingSpecFactory.get_spec_cls's spec_module_path
fallback"`. Consistency across registries is itself a design decision: once a
reader has understood one, they have understood all of them.

---

## Part 3 — How to apply this

A checklist distilled from the above.

1. **Confirm you need it.** Is the concrete class chosen at runtime? Will there
   be more implementations? If both answers are no, use a constructor.
2. **Key on a string from config**, and put the default in one place, not two.
3. **Register loaders, not classes**, if implementations have heavy or optional
   dependencies. Two strings per registration, zero imports.
4. **Reject duplicate registration** loudly rather than overwriting.
5. **Validate what the registry cannot.** With string keys you lose static
   typing, so `assert issubclass(...)` at the boundary.
6. **Provide an escape hatch** (`module_path`) if you want third-party
   implementations without forks.
7. **Invest in the error message.** List the registered names, mention the
   escape hatch. This is the pattern's main usability cost.
8. **Decide getter vs creator** for factory methods and name them accordingly;
   memoise if they are getters.
9. **Be deliberate about `**config` splatting.** It gives free configurability
   and fail-closed typo detection, at the cost of coupling YAML keys to
   constructor parameter names — renaming a parameter becomes a breaking config
   change.
10. **Use an abstract factory when products must be consistent**, such as a
    scheduler-side and worker-side pair. That guarantee is the reason to prefer
    it over two independent factories.

## Related

- `spec` in vLLM is overloaded. `*Spec` classes are specifications
  (`KVCacheSpec` is a frozen dataclass describing a layer's cache format and
  building nothing), whereas `OffloadingSpec` is an ABC that mostly builds.
  Separately, `vllm/v1/spec_decode/` and `SpeculativeConfig` use `spec` to mean
  **speculative**, which is unrelated to either.
- Registries elsewhere in vLLM follow the same shape, notably
  `KVConnectorFactory` (17 registered connectors, keyed on `kv_connector`).