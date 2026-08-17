# Memory Playbooks

这个目录保存内存与缓存相关的节点维护 playbook。默认采用 preview-first 模式，只有显式传入 execute 变量才会修改目标节点状态。

## `cleanup-linux-memory-cache.yml`

释放 Linux 可回收内存缓存。它只写入 `/proc/sys/vm/drop_caches`，不会杀进程、不会清 swap、不会重启服务，也不会持久化 sysctl 配置。

主要变量：

- `memory_cleanup_execute`：默认 `false`。设为 `true` 后才执行 `sync` 并写入 `drop_caches`。
- `drop_caches_value`：默认 `3`，只允许 `1`、`2` 或 `3`。
- `ansible_port`：默认 `36633`。

建议先在单台 canary 节点运行 preview，确认输出符合预期后再执行：

```bash
ansible-playbook --syntax-check -i inventory.yml memory/cleanup-linux-memory-cache.yml
ansible-playbook -i inventory.yml memory/cleanup-linux-memory-cache.yml --limit 10.10.10.111
ansible-playbook -i inventory.yml memory/cleanup-linux-memory-cache.yml --limit 10.10.10.111 -e memory_cleanup_execute=true
```

注意：drop caches 是缓存回收操作，不是进程内存清理。执行后 clean page cache、dentries 和 inodes 可能被丢弃，后续缓存重建期间可能短暂增加 I/O 或 CPU 压力。
