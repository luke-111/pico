# Sandbox

pico 的 `run_shell` 可以在 [bubblewrap](https://github.com/containers/bubblewrap) 沙盒里执行命令，把模型能影响的范围收在仓库目录内。

## 三种模式

| `--sandbox` | 含义 |
|-------------|------|
| `off`（默认） | 直接在主 shell 跑命令 |
| `best_effort` | 有 bubblewrap 就用，没有就降级为直接跑，并发出告警 |
| `required` | 必须有 bubblewrap，否则 `run_shell` 直接报错拒绝 |

```bash
pico --sandbox best_effort
pico --sandbox required
```

## 安装 bubblewrap

bubblewrap 模式只支持 Linux。macOS 和 Windows 上 sandbox 只能保持 `off`。

```bash
sudo apt install bubblewrap        # Debian/Ubuntu
sudo dnf install bubblewrap        # Fedora
sudo pacman -S bubblewrap          # Arch
```

## 沙盒边界

启用沙盒后，`run_shell` 跑的命令默认：

- **读** — 仓库目录、`/usr` `/bin` `/lib` 等系统只读路径
- **写** — 仅限仓库目录和系统 tmp
- **网络** — 默认允许，需要关闭时看下面的配置项

## 在 .pico.toml 里配置

```toml
[sandbox]
mode = "best_effort"       # off | best_effort | required
backend = "auto"           # auto | bubblewrap | none
workspace_write = true     # 是否允许写仓库目录
excluded_commands = []     # 哪些命令豁免沙盒（如 docker exec ...）

[sandbox.filesystem]
extra_readonly_paths = []  # 额外只读挂载点
deny_read = []             # 显式拒绝读取的路径（覆盖默认 ALLOW）
deny_write = []            # 显式拒绝写入的路径
```

## 环境变量

沙盒里继承的环境变量经过 allowlist 过滤，避免 API key 这类敏感变量泄漏给子进程。默认放行 `HOME`、`LANG`、`PATH`、`PWD`、`SHELL`、`TERM` 等。`--secret-env-name` 可以追加需要 redact 的变量名。

## 故障排查

| 现象 | 处理 |
|------|------|
| `bubblewrap is required but not found` | 安装 bubblewrap，或换成 `--sandbox best_effort` |
| `run_shell` 报 `permission denied` 写文件 | 写路径在沙盒外，检查 `workspace_write` 和 `sandbox.filesystem.deny_write` |
| sandbox 卡住不返回 | 命令可能在等输入。run_shell 默认 timeout 20s，可以调 |
| `run_shell` 直接被拒绝 | 工具策略判定这条命令在做 `cat`、`grep`、`find` 这类应该用 search 或 read_file 完成的事 |

## 推荐配置

- 在不熟悉的项目里跑 pico：`--sandbox best_effort --approval ask`
- 在自己长期维护的项目：`--sandbox off --approval auto`
- 跑 CI 或自动化场景：`--sandbox required --approval never`
