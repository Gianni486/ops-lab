## 案例二：Nginx 服务起不来（配置语法错误）

**现象**：一次配置变更后重启 Nginx，服务起不来，站点直接打不开。`systemctl status` 显示 `failed`，`systemctl reload` 也报 `Job for nginx.service invalid.`

**变更前服务是正常的**（1 个 master + 4 个 worker）：

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since 三 2026-10-21 17:04:43 CST; 1min 12s ago
  Process: 13140 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 13137 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 13134 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 13142 (nginx)
    Tasks: 5
   CGroup: /system.slice/nginx.service
           ├─13142 nginx: master process /usr/sbin/nginx
           ├─13143 nginx: worker process
           ├─13144 nginx: worker process
           ├─13145 nginx: worker process
           └─13146 nginx: worker process
```

**变更后启动失败**，`journalctl -u nginx -n 10` 的报错：

```
10月 21 17:06:32 myLinux486 systemd[1]: Starting The nginx HTTP and reverse proxy server...
10月 21 17:06:32 myLinux486 nginx[13315]: nginx: [emerg] unknown directive "mimi" in /etc/nginx/ngi
10月 21 17:06:32 myLinux486 nginx[13315]: nginx: configuration file /etc/nginx/nginx.conf test fail
10月 21 17:06:32 myLinux486 systemd[1]: nginx.service: control process exited, code=exited status=1
10月 21 17:06:32 myLinux486 systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
10月 21 17:06:32 myLinux486 systemd[1]: Unit nginx.service entered failed state.
10月 21 17:06:32 myLinux486 systemd[1]: nginx.service failed.
```

**排查命令**：

```bash
systemctl status nginx                 # ① 看状态：failed，且最后几行自带报错
journalctl -u nginx -n 10              # ② 只看 nginx 的日志，精准定位
journalctl -xe -l                      #    输出被截断时加 -l 看完整行
nginx -t                               # ③ 语法检查，直接告诉你是第几行错了
grep -n "mimi" /etc/nginx/nginx.conf   # ④ 确认出错行位置
vi /etc/nginx/nginx.conf               # ⑤ 修正
nginx -t && systemctl start nginx      # ⑥ 验证 + 启动
```

**根因**：`/etc/nginx/nginx.conf` **第 6 行**多了一个未知指令 `mimi`——变更时手滑写错（本应是别的指令），保存后直接重启，配置语法不过关，Nginx 拒绝启动。

**解决**：修正第 6 行后 `nginx -t` 通过，`systemctl start nginx` 服务恢复 `active (running)`，`curl -I localhost` 正常返回。

**学到什么**：

1. **先看 `systemctl status`，别一上来就翻日志**——status 的最后几行已经把 systemd 抓到的报错打出来了，本次 `unknown directive "mimi"` 一眼可见；
2. **`journalctl -u 服务名` 比 `journalctl -xe` 精准**——`-xe` 会把整台机器的日志都倒出来，`-u nginx` 只看这一个服务；输出出现 `...` 被省略时加 `-l` 显示完整；
3. **`nginx -t` 是最快的定位手段**——它会直接告诉你「第几行、什么错」。注意这是 Nginx 自带的，不是通用命令：httpd 用 `apachectl configtest`、systemd 用 `systemd-analyze verify`、HAProxy 用 `haproxy -c`，没有专用检查命令的服务就只能靠日志；
4. **CentOS 7 的 nginx.service 自带 `ExecStartPre=/usr/sbin/nginx -t`**——意思是「启动之前先跑一遍语法检查」，检查不过就直接掐掉，**根本进不了启动流程**。这也是为什么日志里 `ExecStartPre` 那行是 `status=1/FAILURE`；
5. **`reload` 只对正在运行的服务有效**——本次服务已经是 failed 状态，所以 `systemctl reload nginx` 报 `Job for nginx.service invalid.`（日志原文：`Unit nginx.service cannot be reloaded because it is not active`）。**没跑起来只能用 `start`，跑起来之后改配置才用 `reload`**；
6. **`reload` 优于 `restart`**——reload 是让 master 重读配置、换一批 worker，**已有连接不断**；restart 会把进程杀掉重起，连接全断。生产环境改配置一律 `nginx -t && systemctl reload nginx`；
7. **改配置前先备份**：`cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak`，改坏了能一秒回滚。

**后续预防**：

- 所有配置变更走「备份 → 改 → 语法检查 → reload」的固定动作，不裸改；
- 能进配置管理的进配置管理（第 6 周的 Ansible 正好干这个）；
- 关键服务加存活监控，起不来立刻告警（第 5 周 Prometheus 会覆盖）。
