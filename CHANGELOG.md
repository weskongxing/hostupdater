# Changelog

本项目遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/) 格式。

---

## [Unreleased] - 修复版

基于上游 `GxxkX/hostupdater` 的 `files/usr/bin/hostupdater`（SHA `19560a2`）修改。
本次修改**只涉及 `/usr/bin/hostupdater` 与默认配置**，LuCI 界面、包结构与依赖均未改动，可原地替换。

### 修复

#### 1. 更新 hosts 后不重载 dnsmasq（严重）

**现象**：cron 每小时执行 `hostupdater update`，`/etc/hosts` 已更新，但设备解析结果仍是几小时前的旧值，必须重启路由器才生效。

**原因**：`update_hosts()` 只做 `printf > /etc/hosts`，脚本全文没有任何通知 dnsmasq 的逻辑。dnsmasq 在启动时把 hosts 读入内存，不主动重读。

实测证据（升级前）：
```
/etc/hosts 里  gist.github.com = 37.61.54.158   (09:15 写入)
dnsmasq 实际应答 gist.github.com = 59.24.3.173   (07:00 的旧值)
```

**修复**：新增 `reload_dns()`，在写入成功后调用：
- 优先 `/etc/init.d/dnsmasq reload`
- 兜底 `killall -q -HUP dnsmasq`
- 都失败时记录警告日志
- 可通过 `option reload_dns '0'` 关闭

同时 `do_restore()` 和 `update-source` 分支也会触发重载。

新增命令 `hostupdater reload-dns`，方便手动触发。

#### 2. 订阅源里标记为失效的条目被原样写入（严重）

**现象**：`/etc/hosts` 里出现明显不可用的 IP，导致对应域名反而打不开。

**原因**：`parse_hosts()` 只做 `IP + 主机名` 的正则校验，不检查行尾注释。而订阅源自己会用 `# Timeout` 标注测速失败的条目，本工具却照单全收。

实测证据（`raw.hellogithub.com/hosts` 源，两天内 `gist.github.com` 的取值）：
```
203.98.7.65       # 电信劫持页
243.185.187.39    # 240.0.0.0/4 保留段，不可路由
159.24.3.173      # 韩国 KT，DNS 污染值
159.106.121.75    # GFW 污染值
37.61.54.158      # 韩国 KT，源自己标了 # Timeout
```

**修复**：`parse_hosts()` 增加两道过滤：
- 跳过注释中含 `Timeout / Expired / Dead / Invalid / Failed / Unreachable / 超时 / 失效 / 不可用` 的行
- 跳过不可用的 IP：
  - IPv4：`0.0.0.0/8`、`127.0.0.0/8`、`224.0.0.0/4` 及以上（组播/保留）
  - IPv6：`::`、`::1`
- 日志会统计跳过条数：`解析完成: github (有效行 40, 跳过失效/非法 2)`

#### 3. 去重时破坏了原始 hosts（`::1 localhost` 丢失）

**现象**：OpenWrt 原始 `/etc/hosts` 是
```
127.0.0.1 localhost
::1     localhost ip6-localhost ip6-loopback
```
更新后变成
```
127.0.0.1 localhost
::1     ip6-localhost ip6-loopback      ← localhost 不见了
```

**原因**：`dedupe_merged()` 对**整个文件**做全局去重。`localhost` 在 `127.0.0.1` 行已登记，到 `::1` 行时被判定为重复而删除。IPv6 下 `localhost` 解析异常会连带影响部分依赖 `::1` 的服务。

另外同一个 awk 在切分主机名时没有在行内 `#` 注释处停止，会把注释词（如 `Timeout`）当成域名登记。

**修复**：`dedupe_merged()` 中
- `# BEGIN ORIGINAL HOSTS` / `# END ORIGINAL HOSTS` 段内的行**原样保留**，只把名字登记进 `seen` 供后续源去重
- 遇到 `#` 开头的字段立即 `break`

#### 4. 下载不校验 HTTP 状态码

**现象**：订阅源返回 404 / 502 或弹出认证页时，错误页正文被当作 hosts 内容保存，`download_success=true`，后续流程拿垃圾数据覆盖 `/etc/hosts`。

**原因**：`curl -s -L` 不把 HTTP 4xx/5xx 当作失败。

**修复**：改为 `curl -sfL --retry 3 --retry-delay 2`，非 2xx 直接判为下载失败。

#### 5. 直接覆盖 `/etc/hosts`，无内容校验

**现象**：写盘过程中断电/空间不足会留下截断的 hosts；所有源都解析不出有效条目时也会用"只有原始段"的内容覆盖。

**修复**：
- 先写 `${HOSTS_FILE}.new.$$`，`chmod 644` 后 `mv` 原子替换
- 校验：必须有**至少一个订阅源段落**包含数据行（只看全文会被 `ORIGINAL HOSTS` 段里的 `127.0.0.1 localhost` 蒙混过关）
- 校验不过则放弃更新并记录日志，原 hosts 保持不变

#### 6. 备份文件无限增长

**现象**：每次更新都 `cp hosts hosts.$(date +%Y%m%d_%H%M%S)`，一小时一份，一天 24 份且从不清理。路由器 flash 只有几十 MB。

实测：运行两天已积累 25 份备份。

**修复**：新增 `rotate_backups()`，按 `option backup_keep`（默认 24）只保留最近 N 份。

#### 7. 异常分支未清理临时文件

`do_update()` 中"没有找到启用的hosts源"、"合并结果为空"两条路径直接 `return`，跳过 `cleanup()`，`/tmp/hostupdater` 残留文件。已补齐。

---

### 新增

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `option reload_dns` | `1` | 更新后是否重载 dnsmasq |
| `option backup_keep` | `24` | 备份文件保留份数，`0` = 不轮转 |

新增子命令：

```sh
/usr/bin/hostupdater reload-dns    # 手动重载 dnsmasq
```

`hostupdater status` 现在会显示这两个配置项的当前值。

---

### 兼容性

- 纯 POSIX sh（ash）实现，无新增依赖
- 配置项均为**可选**，缺失时使用默认值，旧配置可直接沿用
- `/etc/hosts` 的注释结构与 `hosts.original` 备份机制完全不变
- 已在 ImmortalWrt 25.12.2 (ramips/mt7621, BusyBox 1.37.0, dnsmasq 2.93) 上实测

---

### 测试

在路由器上用真实 `ash` 跑过沙盒测试（不触碰真实 `/etc/hosts`）：

| 用例 | 结果 |
|---|---|
| `sh -n` 语法检查 | ✔ |
| `::1 localhost ...` 保留 | ✔ |
| `# Timeout` 条目被跳过 | ✔ |
| `0.0.0.0` / `127.x` / `243.x` / `::` 被过滤 | ✔ |
| 重复域名去重 | ✔ |
| 合法 IPv6（`2001:db8::1`）保留 | ✔ |
| 全部是不良条目时拒绝覆盖 hosts | ✔ |
| 备份轮转（6 份 → 保留 3 份） | ✔ |
| `update-source` 单源更新 | ✔ |
| `restore` 后触发重载 | ✔ |
| `reload_dns=1` 实际调用 `/etc/init.d/dnsmasq reload` | ✔ |
