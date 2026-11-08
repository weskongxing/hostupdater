# Host Updater - OpenWrt软件包

一个用于OpenWrt的hosts订阅源管理软件包，支持多个hosts源的定时抓取和手动更新。

## 图片预览

<img width="1912" height="954" alt="wechat_2025-08-28_113832_241" src="https://github.com/user-attachments/assets/a48455a0-960a-461d-b4ce-a61b3fe0772d" />

## 功能特性

- 🔄 **多源管理**: 支持配置多个hosts订阅源
- ⏰ **定时更新**: 支持设置定时自动更新（1-24小时间隔）
- 🖱️ **手动更新**: 支持通过Web界面或命令行手动更新
- 💾 **自动备份**: 自动备份原始hosts文件，支持一键还原
- 🌐 **Web界面**: 提供完整的Luci Web管理界面
- 📝 **日志记录**: 详细的操作日志记录
- 🧹 **干净卸载**: 卸载时自动还原系统到安装前的状态
- 🧠 **去重合并**: 合并最终hosts时会自动去除重复的域名映射，按"先出现先保留"的策略处理，原始系统hosts优先级最高，其次是订阅源按顺序依次合并，避免冲突与污染。

## 安装方法

### 方法1: 从源码编译

1. 将本软件包放入OpenWrt SDK的`package/utils/`目录
2. 在OpenWrt源码根目录执行：
   ```bash
   make menuconfig
   ```
3. 进入 `Utilities` → `hostupdater` 并选择为 `M` 或 `*`
4. 编译：
   ```bash
   make package/hostupdater/compile V=s
   ```

### 方法2: 直接安装

将编译好的`.ipk`文件上传到路由器，然后执行：
```bash
opkg install hostupdater_1.0.0-1_all.ipk
```

## 配置说明

### 基本配置

软件包安装后，配置文件位于：
- UCI配置: `/etc/config/hostupdater`
- 源配置: `/etc/hostupdater/sources.conf`

可选配置项（不写也能正常工作，使用默认值）：

```
config hostupdater 'main'
	option enabled '1'
	option interval '1'
	option auto_backup '1'
	option log_level 'info'

	# 更新 /etc/hosts 后自动重载 dnsmasq，让新条目立即生效（默认 1）
	option reload_dns '1'

	# 备份文件保留份数，超出的自动删除（默认 24，0 = 不轮转）
	option backup_keep '24'
```

> ⚠️ `reload_dns` 很重要：dnsmasq 在启动时把 hosts 读进内存，不主动重读。
> 关掉它的话，更新完必须手动执行 `/etc/init.d/dnsmasq reload` 或重启，否则设备拿到的仍是旧值。

详细修复记录见 [CHANGELOG.md](CHANGELOG.md)。

### 添加hosts源

编辑 `/etc/hostupdater/sources.conf` 文件，格式如下：
```
# 格式: 名称|URL|启用状态(1=启用,0=禁用)
gitcdn|https://hosts.gitcdn.top/hosts.txt|0
tmdb_ipv4|https://raw.githubusercontent.com/cnwikee/CheckTMDB/refs/heads/main/Tmdb_host_ipv4|0
tmdb_ipv6|https://raw.githubusercontent.com/cnwikee/CheckTMDB/refs/heads/main/Tmdb_host_ipv6|0
```

### 去重与优先级策略
- 最终 `/etc/hosts` 会包含：
  - 原始系统hosts（安装时备份的内容）
  - 每个订阅源的BEGIN/END注释块
- 去重策略：按行解析得到"IP + 主机名列表"，对主机名进行去重，保留首次出现的映射
- **原始系统hosts段（`BEGIN/END ORIGINAL HOSTS`）内的内容原样保留，不参与去重**
  （否则 `127.0.0.1 localhost` 会把 `::1` 行的 `localhost` 挤掉）
- 行内 `#` 之后的内容不会被当作主机名
- 优先级：
  1. 原始系统hosts（最高）
  2. 第1个订阅源
  3. 第2个订阅源 … 依次类推
- 这样可以避免同一域名被后续源覆盖，保障系统自带记录与用户自定记录优先

### 条目过滤

写入前会丢弃以下条目（订阅源常用注释标注失效节点）：

| 被丢弃的内容 | 说明 |
|---|---|
| `IP 域名 # Timeout`<br>`IP 域名 # 超时` | 源自己标记测速失败/失效 |
| `IP 域名 # Expired` / `# Dead` / `# Failed` | 其它失效标记 |
| `0.0.0.0  域名` | DNS 污染常见值 |
| `127.0.0.1 域名` | 环回，写进 hosts 会直接劫持 |
| `224.x` ~ `255.x` | 组播/保留段，不可路由 |
| `::` / `::1` | 空的/环回 IPv6 |

日志会记录跳过数量：`解析完成: github (有效行 40, 跳过失效/非法 2)`

## 使用方法

### Web界面

1. 登录Luci Web界面
2. 进入 `服务` → `Host Updater`
3. 配置基本设置和hosts源
4. 点击 `执行更新` 进行手动更新

### 命令行

```bash
# 查看帮助
hostupdater help

# 执行更新
hostupdater update

# 查看状态
hostupdater status

# 创建备份
hostupdater backup

# 还原到原始状态
hostupdater restore

# 手动重载 dnsmasq（让 hosts 立即生效）
hostupdater reload-dns
```

### 服务管理

```bash
# 启用服务
/etc/init.d/hostupdater enable

# 启动服务
/etc/init.d/hostupdater start

# 停止服务
/etc/init.d/hostupdater stop

# 重启服务
/etc/init.d/hostupdater restart

# 查看服务状态
/etc/init.d/hostupdater status
```

## 文件结构

```
/etc/config/hostupdater          # UCI配置文件
/etc/hostupdater/                # 软件包数据目录
├── sources.conf                 # hosts源配置文件
└── backup/                      # 备份目录
    └── hosts.original           # 原始hosts文件备份
/usr/bin/hostupdater             # 主程序
/etc/init.d/hostupdater          # 服务脚本
/var/log/hostupdater.log         # 日志文件
```

## 日志查看

```bash
# 查看实时日志
tail -f /var/log/hostupdater.log

# 查看最近100行日志
tail -n 100 /var/log/hostupdater.log
```

## 故障排除

### 常见问题

1. **更新后无订阅内容**
   - 检查订阅URL是否可访问
   - 查看`/tmp/hostupdater/*.tmp`是否有内容
   - 查看日志“解析完成 (有效行 N)”是否>0

2. **重复或冲突的解析**
   - 新版已启用“按出现顺序去重”，原始系统hosts优先

3. **Web界面无法访问**
   - 确保已安装luci-base和luci-compat
   - 重启rpcd与uhttpd

## 卸载

卸载软件包时会自动：
1. 停止并禁用服务
2. 还原原始hosts文件
3. 清理所有配置文件和数据

```bash
opkg remove hostupdater
```

## 许可证

本软件包采用MIT许可证。详见 [LICENSE](LICENSE) 文件。

## 贡献

欢迎提交Issue和Pull Request来改进这个软件包。

## 更新日志

### v1.0.0
- 初始版本发布
- 支持多hosts源管理
- 提供Web界面和命令行工具
- 支持定时和手动更新
- 自动备份和还原功能
- 去重合并最终hosts（原始hosts最高优先）
