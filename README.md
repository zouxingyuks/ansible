# Ansible Operations Playbooks

这个仓库保存一组面向 Ansible Semaphore 的运维 playbook，以及用于本地启动 Semaphore UI 的 compose 配置。当前 playbook 主要用于 Linux/Kubernetes 节点维护场景，默认采用 preview-first 的安全模式：先检查和展示计划，只有显式传入 `*_execute=true` 才执行会修改系统状态的步骤。

## 目录结构

根 README 只维护当前路径下的稳定入口；目录内具体 playbook 以对应目录的 README 为准。

```text
.
├── .env.example
├── docker-compose.yml
├── docker-compose-mysql.yml
├── docker-compose-postgres.yml
├── memory/
├── network/
├── software/
└── system/
```

## 本地启动 Semaphore

复制示例环境变量后按需要调整端口、镜像版本、管理员账号和数据库连接信息：

```bash
cp .env.example .env
```

使用默认 SQLite 配置：

```bash
docker compose up -d
```

使用外部 PostgreSQL 或 MySQL 时，先确保数据库地址、库名和用户在 `.env` 中正确，再选择对应 compose 文件：

```bash
docker compose -f docker-compose-postgres.yml up -d
docker compose -f docker-compose-mysql.yml up -d
```

容器内 Semaphore 监听 `3000`，宿主机端口由 `PORT` 控制；未设置时 compose 文件默认映射到 `8843`，`.env.example` 中示例为 `3000`。

## Playbook 目录

- [memory/](memory/README.md)：内存与缓存相关维护任务。
- [network/](network/README.md)：系统网络配置相关维护任务。
- `software/`：软件安装或用户环境相关维护任务。
- `system/`：系统级环境或基础配置维护任务。

## Playbook 约定

仓库内 playbook 遵循这些共同约定：

- `hosts: all`：让 Semaphore 模板选择的 inventory 决定目标主机，避免依赖不存在的静态分组。
- `gather_facts: false`：减少对目标节点 Python/facts 的依赖；部分操作使用 `ansible.builtin.raw` 以兼容旧节点。
- `serial: 1`：一次只处理一个节点，适合节点维护和可控 canary。
- `ansible_port: 36633`：当前 playbook 默认使用非标准 SSH 端口；如果实际 inventory 已配置端口，可在 inventory 或 extra vars 中覆盖。
- `*_execute: false`：默认只做 preflight/preview；执行修改必须显式传入对应 execute 变量。

建议先做 syntax check，再限制到单台 canary 节点运行 preview，确认日志后再启用 execute：

```bash
ansible-playbook --syntax-check -i inventory.yml <playbook.yml>
ansible-playbook -i inventory.yml <playbook.yml> --limit 10.10.10.111
ansible-playbook -i inventory.yml <playbook.yml> --limit 10.10.10.111 -e <execute_var>=true
```

在 Semaphore 中运行时，将这些命令映射为模板里的 playbook 路径、inventory、credentials、limit 和 extra variables。遇到失败时，优先确认 Semaphore 实际执行的 playbook 路径和所选 inventory 是否与预期一致。

## 安全检查清单

运行会修改节点状态的任务前，至少确认：

1. Semaphore 任务日志中的 playbook 路径是本仓库中预期文件。
2. 所选 inventory 能匹配 playbook 的 `hosts: all`，并且 credentials 可以登录目标节点。
3. 非标准 SSH 端口 `36633` 与实际节点一致，或已经在 inventory/extra vars 中覆盖。
4. Preview 输出符合预期，并已通过 `--limit` 或 Semaphore limit 在单台 canary 节点验证。
5. 只有在确认要执行时才传入对应的 `*_execute=true`。

如果目标节点 Python 版本较旧，注意 `gather_facts: false` 只会跳过 facts；普通 Ansible module 仍可能要求目标节点有可用 Python。仓库中部分任务使用 `ansible.builtin.raw` 是为了降低这类依赖，但不要把 `shell` 当成绕过 Python 的方式。
