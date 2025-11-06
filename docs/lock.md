我先快速查看并提取 `lightrag/kg/shared_storage.py` 中的函数/方法列表，然后并行搜索整个仓库里它们的调用位置。

[1 tool called]

我现在并行搜索这些函数/方法在仓库中的使用位置，并提取调用点路径。

[16 tools called]

我已读取文件并定位了仓库内的调用点。下面按“函数/方法 → 作用 → 被哪些模块调用”梳理，尽量精炼但覆盖全面。

### 概览
这个文件提供了跨单/多进程统一的异步/同步锁封装、按键分片的多把锁管理（Keyed Lock）、共享命名空间数据、初始化/清理生命周期、以及流水线状态与“更新标志”的分发机制，是全局并发控制与跨进程共享状态的基础设施。

### 日志与调试计数
- direct_log(message, enable_output=True, level="DEBUG")
  - 作用: 将消息直接输出到 stderr，遵循 `lightrag.utils.logger` 的当前日志级别。
  - 调用: 本文件内部大量使用（错误/清理/锁获取日志）。

- inc_debug_n_locks_acquired() / dec_debug_n_locks_acquired() / get_debug_n_locks_acquired()
  - 作用: 调试用计数器，记录已获取的 Keyed Lock 数量（在 DEBUG_LOCKS=True 时生效）。
  - 调用: 本文件 `_KeyedLockContext.__aenter__/__aexit__` 流程。

### 基础工具函数
- _get_combined_key(factory_name, key)
  - 作用: 生成命名空间+键的复合 key。
  - 调用: Keyed 锁内部。

- _perform_lock_cleanup(lock_type, cleanup_data, lock_registry, lock_count, earliest_cleanup_time, last_cleanup_time, current_time, threshold_check=True)
  - 作用: 通用清理逻辑（超时的异步/多进程 keyed 锁），返回清理数量、更新的时间戳。
  - 调用: `_release_shared_raw_mp_lock`、`KeyedUnifiedLock._release_async_lock`、`KeyedUnifiedLock.cleanup_expired_locks`。

- _get_or_create_shared_raw_mp_lock(factory_name, key)
  - 作用: 在多进程模式下，获取/创建由 `multiprocessing.Manager()` 管理的进程间锁（按组合键）。
  - 调用: `KeyedUnifiedLock._get_lock_for_key`。

- _release_shared_raw_mp_lock(factory_name, key)
  - 作用: 释放多进程模式下的 manager.Lock 引用计数，并触发过期清理。
  - 调用: `KeyedUnifiedLock._release_lock_for_key`。

### 锁封装类
- class UnifiedLock
  - 作用: 统一封装 asyncio.Lock 与 multiprocessing.Lock，支持 `async with`/同步上下文，且在多进程模式结合一个辅助 async_lock 防止事件循环阻塞。
  - 主要方法:
    - __aenter__/__aexit__/__enter__/__exit__/locked: 统一的获取/释放行为与日志。
  - 调用: 由 `get_*_lock` 与 `KeyedUnifiedLock._get_lock_for_key` 返回给上层使用。

- class KeyedUnifiedLock
  - 作用: 管理“按键分片”的多把锁（同一命名空间内多 key），支持单/多进程；本地仅保存异步锁表，多进程锁每次获取时取 manager.Lock；构造时可设置默认日志开关。
  - 主要方法:
    - __call__(namespace, keys, enable_logging=None): 语法糖，返回 `_KeyedLockContext`，允许 `async with storage_keyed_lock("ns", ["k1","k2"]): ...`
    - _get_or_create_async_lock / _release_async_lock: 管理本地 async keyed 锁及其引用计数与过期清理。
    - _get_lock_for_key / _release_lock_for_key: 组合 async 锁与多进程锁，构建 `UnifiedLock`，并处理引用计数。
    - cleanup_expired_locks(): 主动清理过期的 async+mp keyed 锁，并返回统计。
    - get_lock_status(): 返回当前 async/mp keyed 锁的总数与待清理数量。
  - 典型调用:
    - 通过全局方法 `get_storage_keyed_lock()` 对外暴露。

- class _KeyedLockContext
  - 作用: “按键分片锁”的异步上下文管理器，确保对 keys 按排序顺序获取锁、出现异常时严格回滚（包含释放主锁/引用计数/调试计数），退出时逆序释放。
  - 主要方法:
    - __aenter__: 逐个 key 获取 `UnifiedLock` 并进入。
    - __aexit__: 逆序退出并释放引用计数，使用 asyncio.shield 防止释放过程被取消中断。
    - _rollback_acquired_locks: 进入流程中途异常的回滚保障。
  - 调用: 由 `KeyedUnifiedLock.__call__` 创建；上层以 `async with get_storage_keyed_lock([...])` 使用。

### 锁获取入口（面向全局使用）
- get_internal_lock(enable_logging=False)
  - 作用: 返回内部管理用的全局锁（保护本模块内部共享结构）。
  - 被使用: 本文件内部（如初始化/flags 修改等）。

- get_storage_lock(enable_logging=False)
  - 作用: 返回存储一致性的全局锁，多用于存取 vector/kv/doc 状态时的互斥。
  - 被使用:
    - `lightrag/kg/milvus_impl.py` 多处
    - `lightrag/kg/qdrant_impl.py`
    - `lightrag/kg/postgres_impl.py` 多处
    - `lightrag/kg/redis_impl.py`
    - `lightrag/kg/mongo_impl.py` 多处
    - `lightrag/kg/networkx_impl.py`
    - `lightrag/kg/json_kv_impl.py`
    - `lightrag/kg/json_doc_status_impl.py`
    - `lightrag/kg/faiss_impl.py`
    - `lightrag/kg/nano_vector_db_impl.py`（构造时持有）

- get_pipeline_status_lock(enable_logging=False)
  - 作用: 返回流水线状态的全局锁，保护 `pipeline_status` 命名空间的并发更新。
  - 被使用:
    - `lightrag/lightrag.py`
    - `lightrag/api/routers/document_routes.py`

- get_graph_db_lock(enable_logging=False)
  - 作用: 返回图数据库操作的全局锁，确保对图数据库的原子性操作。
  - 被使用:
    - `lightrag/kg/neo4j_impl.py`
    - `lightrag/kg/memgraph_impl.py`
    - `lightrag/kg/postgres_impl.py`（图相关部分）
    - `lightrag/kg/mongo_impl.py`（图相关部分）
    - `lightrag/lightrag.py`（传递给 graph 层）

- get_storage_keyed_lock(keys, namespace="default", enable_logging=False)
  - 作用: 返回“按命名空间+键”的多把锁上下文，适合细粒度并发控制（比如针对某些 documentId/实体Id/边）。
  - 被使用:
    - `lightrag/utils_graph.py` 多处
    - `lightrag/operate.py` 多处

- cleanup_keyed_lock()
  - 作用: 强制执行一次 keyed 锁的过期清理，返回清理统计与当前状态。
  - 被使用:
    - `lightrag/api/lightrag_server.py`（运维端点）

- get_keyed_lock_status()
  - 作用: 不触发清理，仅返回 keyed 锁的当前状态。
  - 被使用: 仅本文件导出，服务层可能用于观测（grep 显示直接调用在服务中用的是 cleanup）。

- get_data_init_lock(enable_logging=False)
  - 作用: 返回“数据初始化”全局锁，确保初始化阶段的互斥，避免重复加载或并发竞态。
  - 被使用:
    - 各存储实现首次加载/重载时：`neo4j_impl.py`、`memgraph_impl.py`、`mongo_impl.py`、`postgres_impl.py`、`redis_impl.py`、`qdrant_impl.py`、`milvus_impl.py`、`faiss_impl.py`、`json_kv_impl.py`、`json_doc_status_impl.py`、`lightrag/lightrag.py`。

### 共享数据与流水线初始化/状态
- initialize_share_data(workers: int = 1)
  - 作用: 初始化共享数据与锁，区分单进程（asyncio.Lock + 本地 dict）与多进程（Manager + 进程锁 + 共享 dict）模式；并创建 `KeyedUnifiedLock` 实例与异步辅助锁。
  - 被使用:
    - `lightrag/lightrag.py`
    - `lightrag/api/run_with_gunicorn.py`（根据 Gunicorn worker 数）
- initialize_pipeline_status(namespace="pipeline_status")
  - 作用: 初始化流水线状态命名空间，填充 `busy/docs/batchs/history_messages` 等字段。仅在未初始化时执行。
  - 被使用:
    - API 启动/示例/自检中大量调用（文档、examples、server、tools）。
- get_namespace_data(namespace, first_init=False)
  - 作用: 获取（必要时创建）某命名空间对应的共享字典；对 `pipeline_status` 做特殊保护（未先调用 `initialize_pipeline_status` 会抛 `PipelineNotInitializedError`，除非 first_init=True）。
  - 被使用:
    - `lightrag/lightrag.py`、`lightrag/api/lightrag_server.py`、`lightrag/api/routers/document_routes.py`、`kg/json_*_impl.py` 等。

### 更新标志（跨 worker 的数据刷新通知）
- get_update_flag(namespace)
  - 作用: 为某 worker 在某命名空间注册一个可变布尔“更新标志”，用于标记“需要从文件/存储重载数据”。
  - 被使用:
    - `kg/json_kv_impl.py`、`kg/json_doc_status_impl.py`、`kg/faiss_impl.py`、`kg/networkx_impl.py`、`kg/nano_vector_db_impl.py`（构造时保存引用）。

- set_all_update_flags(namespace)
  - 作用: 将某命名空间下所有 worker 的“更新标志”设置为 True，通知它们需要重载。
  - 被使用:
    - 多个存储实现中，写操作或批量变更后触发（见各 impl 的多处）。

- clear_all_update_flags(namespace)
  - 作用: 将标志清零（False），通常在完成一次 reload 后调用。
  - 被使用:
    - `kg/json_kv_impl.py`、`kg/json_doc_status_impl.py`（初始化载入完成后清零）。

- get_all_update_flags_status()
  - 作用: 汇总所有命名空间的各 worker 更新标志布尔状态，供 API 观测。
  - 被使用:
    - `lightrag/api/routers/document_routes.py`（对外暴露状态）。

- try_initialize_namespace(namespace) -> bool
  - 作用: 判断当前 worker 是否获得“首个初始化权限”（只允许一个 worker 加载初始数据）；已有人初始化则返回 False。
  - 被使用:
    - `kg/json_kv_impl.py`、`kg/json_doc_status_impl.py`。

### 资源清理
- finalize_share_data()
  - 作用: 关闭/清理 Manager 与共享资源，重置全局变量；单进程则做内存清理。
  - 被使用:
    - `lightrag/api/gunicorn_config.py`、`lightrag/api/lightrag_server.py`（进程退出/重载时）。

### 关键代码参照

```1044:1053:/Users/hongdongjian/Documents/workspace/python/LightRAG/lightrag/kg/shared_storage.py
def get_internal_lock(enable_logging: bool = False) -> UnifiedLock:
    """return unified storage lock for data consistency"""
    async_lock = _async_locks.get("internal_lock") if _is_multiprocess else None
    return UnifiedLock(
        lock=_internal_lock,
        is_async=not _is_multiprocess,
        name="internal_lock",
        enable_logging=enable_logging,
        async_lock=async_lock,
    )
```

```1092:1102:/Users/hongdongjian/Documents/workspace/python/LightRAG/lightrag/kg/shared_storage.py
def get_storage_keyed_lock(
    keys: str | list[str], namespace: str = "default", enable_logging: bool = False
) -> _KeyedLockContext:
    """Return unified storage keyed lock for ensuring atomic operations across different namespaces"""
    global _storage_keyed_lock
    if _storage_keyed_lock is None:
        raise RuntimeError("Shared-Data is not initialized")
    if isinstance(keys, str):
        keys = [keys]
    return _storage_keyed_lock(namespace, keys, enable_logging=enable_logging)
```

```1171:1210:/Users/hongdongjian/Documents/workspace/python/LightRAG/lightrag/kg/shared_storage.py
def initialize_share_data(workers: int = 1):
    """
    Initialize shared storage data for single or multi-process mode.
    ...
    """
    global         _manager,         _workers,         _is_multiprocess,         _storage_lock,         _lock_registry,         _lock_registry_count,         _lock_cleanup_data,         _registry_guard,         _internal_lock,         _pipeline_status_lock,         _graph_db_lock,         _data_init_lock,         _shared_dicts,         _init_flags,         _initialized,         _update_flags,         _async_locks,         _storage_keyed_lock,         _earliest_mp_cleanup_time,         _last_mp_cleanup_time
    # Check if already initialized
    if _initialized:
        direct_log(
            f"Process {os.getpid()} Shared-Data already initialized (multiprocess={_is_multiprocess})"
        )
        return
    _workers = workers
    if workers > 1:
        _is_multiprocess = True
        _manager = Manager()
        ...
```

如果你希望，我可以把以上“被使用的具体行号”也逐一列出，或针对你当前正在看的位置（你 IDE 光标在第 1092 行附近）详细解释 `get_storage_keyed_lock` 的调用流程与死锁规避细节。

- 我接下来可以为特定方法画出调用关系图（谁调用它/它调用谁），或聚焦你关心的实现分支（单进程 vs 多进程）。需要我继续吗？

- 也可以进一步汇总“哪些命名空间/哪些 key”在实际业务中被锁定（例如 `utils_graph.py`、`operate.py` 中针对哪些实体/边使用 keyed lock）。