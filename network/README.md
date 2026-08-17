# Network Playbooks

这个目录保存系统网络配置相关的节点维护 playbook。默认采用 preview-first 模式，只有显式传入 execute 变量才会修改目标节点状态。

## `setup-system-proxy.yml`

写入系统级 HTTP/HTTPS proxy profile，并确保系统 bash/zsh 启动文件 source 该 profile。默认只展示计划，不写文件。

主要变量：

- `proxy_url`：必填，例如 `http://proxy.example:7890`。
- `proxy_execute`：默认 `false`。设为 `true` 后才写入 profile 和 shell startup block。
- `proxy_profile_path`：默认 `/etc/profile.d/proxy.sh`。
- `proxy_no_proxy`：可传逗号分隔字符串或 YAML list，默认包含 localhost、RFC1918 网段和 Kubernetes service 域。
- `ansible_port`：默认 `36633`。

建议先在单台 canary 节点运行 preview，确认输出符合预期后再执行：

```bash
ansible-playbook --syntax-check -i inventory.yml network/setup-system-proxy.yml
ansible-playbook -i inventory.yml network/setup-system-proxy.yml --limit 10.10.10.111 -e proxy_url=http://proxy.example:7890
ansible-playbook -i inventory.yml network/setup-system-proxy.yml --limit 10.10.10.111 -e proxy_url=http://proxy.example:7890 -e proxy_execute=true
```

执行后，新开的 bash/zsh shell 会通过系统启动文件加载 proxy profile；当前 shell 需要手动 source 对应 profile 才会立即生效。
