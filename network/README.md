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

## `setup-system-dns.yml`

写入系统 `/etc/resolv.conf` DNS resolver 配置。默认只展示当前 resolver 文件和计划，不写文件。

这个 playbook 只管理普通文件形式的 `/etc/resolv.conf`。如果目标节点的 `/etc/resolv.conf` 是指向 `systemd-resolved` 或 NetworkManager 管理路径的 symlink，preview 会打印警告，execute 会中止并提示使用对应管理器的专用方案，避免覆盖系统 DNS 管理链路。

主要变量：

- `dns_nameservers`：必填，可传逗号分隔字符串或 YAML list，例如 `10.0.0.2,10.0.0.3`；每一项必须是 IPv4 或 IPv6 字面量。
- `dns_search_domains`：可选，可传逗号分隔字符串或 YAML list，例如 `example.internal,svc.cluster.local`；每一项只能包含 resolver search domain 安全字符。
- `dns_options`：可选，可传逗号分隔字符串或 YAML list，例如 `timeout:2,attempts:2`；每一项只能包含 resolver option 安全字符。
- `dns_probe_domain`：执行后用于 `getent hosts` 验证解析的域名，默认 `example.com`。
- `dns_resolv_conf_path`：固定为 `/etc/resolv.conf`，为避免误写其他 root 文件，不支持通过 Extra Variables 改成其他路径。
- `dns_execute`：默认 `false`。设为 `true` 后才备份并写入 resolver 文件。
- `ansible_port`：默认 `36633`。

建议先在单台 canary 节点运行 preview，确认目标节点不是 symlink 管理的 resolver 文件，且计划内容符合预期后再执行：

```bash
ansible-playbook --syntax-check -i inventory.yml network/setup-system-dns.yml
ansible-playbook -i inventory.yml network/setup-system-dns.yml --limit 10.10.10.111 -e dns_nameservers=10.0.0.2,10.0.0.3 -e dns_search_domains=example.internal
ansible-playbook -i inventory.yml network/setup-system-dns.yml --limit 10.10.10.111 -e dns_nameservers=10.0.0.2,10.0.0.3 -e dns_search_domains=example.internal -e dns_execute=true
```

在 Semaphore 中运行时，将 `dns_nameservers`、`dns_search_domains`、`dns_options`、`dns_probe_domain` 和 `dns_execute` 放到 Extra Variables；第一次模板运行建议设置 `limit` 为单台 canary 节点。

## `setup-system-hosts.yml`

管理系统 `/etc/hosts` 中一段固定 Ansible marker block，用于少数机器的快速 hosts 记录修改。默认只展示当前状态和计划，不写文件；长期、大批量域名管理仍应使用 DNS。

这个 playbook 不接管整个 `/etc/hosts`，也不会自动修改 marker block 外的既有记录。如果计划写入的 IP 或 hostname 已经出现在非托管区域，preview/verify 会输出 `WARN`，但不会删除或改写那些行。

主要变量：

- `hosts_entries`：`hosts_state=present` 时必填，可传逗号分隔字符串或 YAML list；每个元素是一整行 `/etc/hosts` 记录，例如 `10.0.0.10 api.internal api,10.0.0.11 db.internal db`。
- `hosts_state`：默认 `present`；设为 `absent` 时删除固定 marker block，且不要求传 `hosts_entries`。
- `hosts_file_path`：默认 `/etc/hosts`。
- `hosts_execute`：默认 `false`。设为 `true` 后才写入或删除 managed block。
- `ansible_port`：默认 `36633`。

建议先在单台 canary 节点运行 preview，确认 managed block 和重复记录警告符合预期后再执行：

```bash
ansible-playbook --syntax-check -i inventory.yml network/setup-system-hosts.yml
ansible-playbook -i inventory.yml network/setup-system-hosts.yml --limit 10.10.10.111 -e '{"hosts_entries":"10.0.0.10 api.internal api,10.0.0.11 db.internal db"}'
ansible-playbook -i inventory.yml network/setup-system-hosts.yml --limit 10.10.10.111 -e '{"hosts_entries":"10.0.0.10 api.internal api,10.0.0.11 db.internal db","hosts_execute":true}'
ansible-playbook -i inventory.yml network/setup-system-hosts.yml --limit 10.10.10.111 -e '{"hosts_state":"absent","hosts_execute":true}'
```

在 Semaphore 中运行时，将 `hosts_entries`、`hosts_state`、`hosts_file_path` 和 `hosts_execute` 放到 Extra Variables；第一次模板运行建议设置 `limit` 为单台 canary 节点。
