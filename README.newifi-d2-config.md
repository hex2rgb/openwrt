# Newifi D2 优化配置说明

## 硬件规格
- **CPU**: MediaTek MT7621A ver 1, eco 3 @ 880MHz
- **内存**: 512MB DDR3
- **Flash**: 32MB (Winbond W25Q256)
- **以太网**: MediaTek MT7530 Gigabit switch

## 配置目标
- ✅ **xray-core** - 现代协议 + 低 CPU 占用
- ✅ **WireGuard** - 快速安全的 VPN
- ✅ **PassWall2** - 成熟的透明代理方案
- ✅ **smartdns** - 智能 DNS 解析
- ✅ **性能优化** - 针对 MT7621A 优化
- ✅ **稳定性** - 完整的依赖支持

## 配置特点

### 1. 核心功能
- **xray-core + PassWall2**: 完整的透明代理解决方案
  - xray-geoip + xray-geosite 用于分流
  - dns2socks, ipt2socks, microsocks 支持

- **WireGuard**: 高性能 VPN
  - 内核级加密，性能优异
  - 完整的 LuCI 管理界面

### 2. 网络支持
- **iptables 完整支持**: 包含所有必需的模块
  - ✅ 内核模块：`kmod-ipt-tproxy` (TProxy 必需)
  - ✅ 用户空间模块：`iptables-mod-tproxy`
  - ✅ 连接跟踪：`kmod-ipt-conntrack`

- **DNS 解析**:
  - smartdns：智能分流
  - dnsmasq-full：完整 DNS 功能（替代基础版）

- **IPv6 支持**: odhcp6c + odhcpd-ipv6only

### 3. 性能优化
- **wpad-openssl**: 完整 Wi-Fi 支持（替代 wpad-basic）
- **ip-full**: 完整的路由工具
- **openssh-server**: 更好的远程维护

### 4. 空间优化
- 移除了 Docker（太重）
- 移除了不必要的开发工具
- 移除了 USB 存储支持（按需启用）
- 只保留必要的包

## 固件大小预估

| 组件 | 大小估算 |
|------|---------|
| 系统基础 | 8-10MB |
| xray-core + geo | 8-12MB |
| PassWall2 + 依赖 | 7-11MB |
| WireGuard | 0.5-1MB |
| smartdns | 0.5-1MB |
| iptables 全套 | 1-2.5MB |
| wpad-openssl | 0.4-0.8MB |
| openssh-server | 1-2MB |
| 固件预留 | 2-3MB |
| **总计** | **~29-33MB** |

⚠️ **注意**: 总计约 29-33MB，接近 32MB 上限，需要实际编译验证。

## 使用方法

### 方法 1: 直接使用配置文件
```bash
# 复制配置文件
cp .config.newifi-d2-optimized .config

# 更新 feeds（添加 PassWall2 源）
./scripts/feeds update -a
./scripts/feeds install -a

# 配置
make defconfig

# 编译
make -j$(nproc) V=s
```

### 方法 2: 通过 menuconfig 配置
```bash
# 更新 feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 打开配置界面
make menuconfig

# 手动选择以下选项：
# 1. Target System: ramips/mt7621
# 2. Target Profile: D-Team Newifi D2
# 3. 选择 PassWall2、xray-core、WireGuard 等相关包
```

## 关键配置项说明

### 必需的 Feeds
需要添加 PassWall2 源到 `feeds.conf.default`:
```
src-git passwall_packages https://github.com/xiaorouji/openwrt-passwall-packages.git;main
src-git passwall2 https://github.com/xiaorouji/openwrt-passwall2.git;main
```

### 重要的内核模块
以下内核模块**必须**包含，否则 TProxy 无法工作：
- `kmod-ipt-core`
- `kmod-ipt-conntrack`
- `kmod-nf-conntrack`
- `kmod-ipt-tproxy` ⚠️ **关键**
- `kmod-nf-tproxy`

### 如果固件超大小
如果编译后固件超过 32MB，可以考虑：
1. 移除 `smartdns`，使用 `dnsmasq-full` + PassWall2 内置 DNS
2. 移除 `luci-app-wireguard`，使用命令行配置 WireGuard
3. 移除 `openssh-server`，保留 `dropbear`

## 验证配置

编译完成后，检查固件大小：
```bash
ls -lh bin/targets/ramips/mt7621/
```

固件应小于 32MB。

## 性能优化建议

1. **CPU 调度**: MT7621A 双核，WireGuard 和 xray 会利用多核
2. **内存使用**: 512MB RAM 足够运行所有服务
3. **并发连接**: PassWall2 + xray 可处理大量并发连接
4. **Wi-Fi 性能**: wpad-openssl 提供完整的 Wi-Fi 功能

## 故障排除

### TProxy 不工作
- 检查 `kmod-ipt-tproxy` 是否正确安装
- 检查内核模块是否加载：`lsmod | grep tproxy`

### PassWall2 启动失败
- 检查 xray-core 是否正确安装
- 检查 iptables 内核模块是否完整

### WireGuard 无法启动
- 检查 `kmod-wireguard` 是否正确安装
- 检查加密模块：`lsmod | grep chacha20poly1305`

## GitHub Actions 自动编译

### Workflow 说明

已创建 GitHub Actions workflow 文件：`.github/workflows/build-newifi-d2.yml`

### 触发方式

1. **推送 main/master 分支（自动触发）** ⭐
   - **只有推送 main 或 master 分支时才会触发编译**
   - 其他分支推送不会触发编译
   - 避免不必要的编译，节省 GitHub Actions 时间

2. **手动触发（workflow_dispatch）**
   - 在 GitHub Actions 页面手动运行
   - 可选择是否创建 Release
   - 可以在任何分支上手动触发

3. **创建 Tag（自动 Release）**
   - 当创建新的 release tag 时自动触发
   - 自动创建 GitHub Release

### 使用步骤

1. **推送到 GitHub**
   ```bash
   git add .
   git commit -m "Add Newifi D2 build configuration"
   git push origin main
   ```

2. **查看编译进度**
   - 进入 GitHub 仓库
   - 点击 Actions 标签
   - 查看 "Build OpenWrt for Newifi D2" workflow

3. **下载固件**
   - 编译完成后，在 Actions 页面下载 artifacts
   - 或查看 Releases 页面（如果创建了 Release）

### Release 创建

#### 方法 1: 手动触发并创建 Release
1. 进入 GitHub Actions 页面
2. 选择 "Build OpenWrt for Newifi D2" workflow
3. 点击 "Run workflow"
4. 选择 "create_release: true"
5. 运行并等待完成

#### 方法 2: 通过 Tag 创建 Release
```bash
# 创建并推送 tag
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

#### 方法 3: 在 GitHub 界面创建 Release
1. 进入 Releases 页面
2. 点击 "Draft a new release"
3. 创建新 release 会自动触发编译 workflow

### Workflow 功能

- ✅ 自动安装编译依赖
- ✅ 使用 ccache 加速编译
- ✅ 更新并安装 feeds
- ✅ 自动编译固件
- ✅ 验证编译结果
- ✅ 上传 artifacts
- ✅ 自动创建 GitHub Release（可选）
- ✅ 包含固件信息（MD5、SHA256）

### 编译时间

- 首次编译：约 2-3 小时
- 后续编译（使用缓存）：约 1-2 小时

### 注意事项

1. **GitHub Actions 限制**
   - 免费账户：每月 2000 分钟
   - 单次编译约 120-180 分钟
   - 每月约可编译 10-15 次

2. **缓存策略**
   - 使用 ccache 加速编译
   - 缓存会自动保存和恢复

3. **固件大小检查**
   - Workflow 会自动验证固件大小
   - 如果超过 32MB 会报错

## 更新日志

- 2024-01-17: 初始配置
  - 集成 xray-core + PassWall2
  - 集成 WireGuard
  - 优化性能和稳定性
  - 添加 GitHub Actions 自动编译和 Release 生成
