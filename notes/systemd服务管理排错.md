# 第 1 周 · 周二：systemd 服务管理

> 一句话目标：把一个程序交给系统托管 —— 能统一管理、能开机自启、崩了能自动拉起来。

---

## 一、systemctl 常用命令

| 命令 | 作用 | 大白话 |
|------|------|--------|
| `systemctl start 服务名` | 立即启动 | 现在就跑起来 |
| `systemctl stop 服务名` | 停止 | 现在就停 |
| `systemctl restart 服务名` | 重启 | 停一下再跑 |
| `systemctl status 服务名` | 看状态 | 活着没 / 为什么挂了 |
| `systemctl enable 服务名` | 设开机自启 | 以后开机自己起来 |
| `systemctl disable 服务名` | 取消开机自启 | 开机别自己起来 |
| `systemctl is-enabled 服务名` | 查是否开机自启 | 回显 `enabled` / `disabled` |
| `systemctl is-active 服务名` | 查是否正在运行 | 回显 `active` / `inactive` |
| `systemctl daemon-reload` | 重新加载所有服务文件 | 改完配置必做 |
| `systemctl list-unit-files --type=service` | 列出所有服务 | 看看都有啥 |

**start 和 enable 完全不是一回事**（最容易混的一对）：

- `start` = 现在启动，**重启电脑后就没了**
- `enable` = 登记开机自启，**当下不会启动**
- 两个都要，就两条都敲：`systemctl enable --now 服务名`（新版支持一步到位）

`enable` 干了什么？就在 `/etc/systemd/system/multi-user.target.wants/` 目录下建了一个指向服务文件的**软链接**。开机时 systemd 去这个目录里挨个把服务拉起来，仅此而已。

---

## 二、头号大坑：改了服务文件必须 daemon-reload

**现象**：改完 `/etc/systemd/system/xxx.service`，敲 `systemctl restart xxx`，发现改动根本没生效。

**原因**：systemd 开机时已经把所有服务文件**读进内存**了，你改的是硬盘上的文件，内存里还是旧的。`restart` 只是重新跑一遍程序，不会重读配置。

**正确顺序**（三选一记住就行）：

```bash
# 新建或修改服务文件后
systemctl daemon-reload        # 1. 让 systemd 重新读文件
systemctl restart xxx          # 2. 再重启服务

# 验证
systemctl status xxx
```

> 类比：改了 Word 文档要点保存，改了服务文件要 `daemon-reload`。没保存的改动，重开多少次都没用。

---

## 三、systemctl status 输出怎么读

```bash
systemctl status nginx
```

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since 四 2026-09-17 10:00:00 CST; 2h 13min ago
 Main PID: 1234 (nginx)
    Tasks: 2
   Memory: 3.2M
   CGroup: /system.slice/nginx.service
```

| 行 | 告诉我什么 |
|----|-----------|
| **Loaded 行** | ① 服务文件**在哪**（`/usr/lib/...` 或 `/etc/systemd/system/...`）<br>② **是否开机自启**（`enabled` / `disabled`） |
| **Active 行** | ① 当前**运行状态**（`active (running)` / `inactive (dead)` / `failed`）<br>② 什么时候起的、跑了多久 |
| **Main PID** | 主进程号，排查时直接拿它去 `ps`、`top` 里查 |

> 记法：**Loaded 看"登记信息"，Active 看"活着没"**。想知道开机自启状态，其实一眼扫 Loaded 行就够了，不用专门敲 `is-enabled`。

---

## 四、.service 文件的三个核心段

服务文件放 `/etc/systemd/system/xxx.service`，三个段缺一不可：

| 段 | 回答的问题 | 常用字段 |
|----|-----------|---------|
| `[Unit]` | 这是什么、什么时候启动 | `Description`、`After`、`Requires` |
| `[Service]` | 怎么启动、崩了怎么办 | `Type`、`ExecStart`、`Restart`、`User` |
| `[Install]` | 装到哪个启动级别 | `WantedBy` |

> 注意大小写：`[Unit]` `[Service]` `[Install]`，首字母必须大写，Linux 严格区分大小写。

### 完整示例

```ini
[Unit]
Description=我的巡检脚本
After=network.target

[Service]
Type=simple
ExecStart=/opt/scripts/check.sh
Restart=on-failure
User=root

[Install]
WantedBy=multi-user.target
```

### 三个关键字段

**`ExecStart`** —— 服务启动时执行什么命令。
- 必须写**绝对路径**（`/opt/scripts/check.sh` 而不是 `check.sh`）
- 脚本要有执行权限 `chmod +x`，且第一行有 `#!/bin/bash`
- 想跑完就退出还保持 active，加 `RemainAfterExit=yes`

**`Restart`** —— 程序挂了要不要自动拉起来。
- `on-failure`：非正常退出（退出码非 0）才重启 ← **最常用**
- `always`：不管怎么退出都重启
- `no`：永不重启（默认值）

**`WantedBy`** —— 这个服务挂在哪块"插线板"上，决定**什么时候跟着起来**。
- `multi-user.target` = 多用户命令行模式（对应老的运行级别 3，服务器默认）
- `graphical.target` = 图形界面模式（级别 5）
- 它不是"要不要启动"，而是"**系统进入到这个级别时，顺带把我也拉起来**"

---

## 五、服务起不来的排查顺序（通用套路）

```bash
# 1. 先看状态，确认到底是不是 failed
systemctl status check

# 2. 看这个服务自己的日志（-u 指定服务，-e 跳到末尾）
journalctl -u check -e

# 3. 把 ExecStart 那条命令单独复制出来，手动跑一遍
/opt/scripts/check.sh

# 4. 还是没报错就查这三项
ls -l /opt/scripts/check.sh    # 有没有 x 执行权限
head -1 /opt/scripts/check.sh  # 有没有 #!/bin/bash
echo $PATH                     # 脚本里用的命令在不在 PATH 里
```

**第三步是老运维的招**：systemd 不报错不代表命令没毛病，把命令单独拎出来跑一遍，问题立刻现形。常见的三类原因：

1. **路径问题** —— 写了相对路径，systemd 找不到
2. **权限问题** —— 脚本没 `chmod +x`
3. **环境问题** —— 手动跑有 `PATH`、有环境变量，交给 systemd 跑就都没有了（服务里要用绝对路径）

---

## 六、易错点清单（我自己答错的，重点记）

| # | 易错点 | 正确认知 |
|---|--------|---------|
| 1 | 以为 `restart` 就能让新配置生效 | 必须先 `systemctl daemon-reload`，这一条我错了两次 |
| 2 | 不知道 `Loaded:` 行有啥用 | 服务文件位置 + 开机自启状态，一眼能看全 |
| 3 | 以为 `WantedBy` 是"登录规格" | 是"插在哪个 target 上"，决定跟谁一起启动 |
| 4 | `Restart=` 不知道填啥 | 填 `on-failure`：程序崩了自动重启 |
| 5 | 拼写：`ExecStart`、`WantedBy`、`enable` | Linux 里拼错直接失效，首字母大写、注意 `Start` 的 S |

---

## 七、动手验收

- [ ] 写一个 `.service` 文件，把一个 `date >> /tmp/test.log` 的脚本变成服务
- [ ] `daemon-reload` → `enable` → `start` 三步走通
- [ ] `systemctl status` 看到 `Active: active`，且 `Loaded` 行显示 `enabled`
- [ ] 重启虚拟机，服务自己起来了，日志有新内容

---

**素材归档**：`systemctl status` 截图、`journalctl` 报错截图 → 放进 `assets/`
