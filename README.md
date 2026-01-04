# 小米路由器 SSH 无法使用 SFTP 管理文件：问题排查与解决方案



![](https://p3-flow-imagex-sign.byteimg.com/ocean-cloud-tos/image_skill/58e9758d-d440-4e70-ab3c-1897f361a960_1767445876184609307_origin~tplv-a9rns2rl98-image-qvalue.image?rcl=20260103211117C6E6E069B8EB8E60F0DF\&rk3s=8e244e95\&rrcfp=026f1a63\&x-expires=1799068277\&x-signature=YpKz2j4i0dTK%2B3bAyKz8VneAaxo%3D)

## 一、核心问题汇总



1. **默认组件缺失**：小米路由器基于 OpenWrt 定制，默认未安装 `openssh-sftp-server`，导致 SSH 连接后无法通过 SFTP 管理文件；

2. **opkg 源失效**：尝试 `opkg install openssh-sftp-server` 在线安装时，出现源 404 错误，无法正常获取安装包；

3. **目录权限 / 空间问题**：

* 系统 `/overlay` 目录空间不足，且默认挂载为只读，无法写入新文件或安装包；

* 尝试挂载 `/data` 目录（可写）到 `/overlay` 后，`/data/usr` 目录可写，但核心系统目录 `/usr` 仍为只读，无法直接部署 `sftp-server`；

1. **架构不兼容**：初期从其他 OpenWrt 设备复制或解压的 `sftp-server` 可执行文件，与小米路由器 CPU 架构不匹配，无法执行。

## 二、操作思路拆解（核心逻辑）

整体解决思路围绕「**规避只读目录→手动部署组件→解决架构兼容**」展开，步骤逻辑清晰可行：



1. 识别系统目录限制（`/overlay` 只读 / 空间不足、`/usr` 只读），放弃直接在线安装；

2. 利用 `/data` 目录的可写特性，创建与系统对应的目录结构（`/data/usr/libexec`）；

3. 手动复制 / 解压 `sftp-server` 组件到可写目录；

4. 通过挂载方式，将 `/data` 下的可写目录映射到系统只读目录（`/usr/libexec`），让系统识别并调用组件；

5. 筛选适配小米路由器架构的 `sftp-server` 版本，解决执行问题。

## 三、完整解决方案（分步执行）

### 步骤 1：确认系统架构（关键前提）



* 执行命令查看路由器 CPU 架构：



```
cat /proc/cpuinfo | grep architecture
```



* 常见小米路由器架构为 `mips` 或 `aarch64`，后续需下载对应架构的 `openssh-sftp-server` 安装包（避免架构不兼容）。

### 步骤 2：处理可写目录与目录映射



1. 在 `/data` 目录创建与系统匹配的目录结构：



```
mkdir -p /data/usr/libexec
```



1. 挂载 `/data/usr/libexec` 到系统 `/usr/libexec`（临时生效，重启后需重新挂载，或添加到开机自启）：



```
mount --bind /data/usr/libexec /usr/libexec
```



* 若需永久生效，编辑 `/etc/rc.local`，在 `exit 0` 前添加上述挂载命令。

### 步骤 3：获取并部署适配的 sftp-server



1. 下载对应架构的 `openssh-sftp-server` 安装包：

* OpenWrt 官方镜像站：[https://downloads.op](https://downloads.openwrt.org/)[enwrt](https://downloads.openwrt.org/)[.org/](https://downloads.openwrt.org/)

* 小米万兆路由器SFTP插件（需根据型号筛选）；
```
cd /data
curl -L openssh-sftp-server.ipk 'https://mirrors.ustc.edu.cn/openwrt/releases/18.06.0/packages/aarch64_generic/packages/openssh-sftp-server_7.7p1-2_aarch64_generic.ipk'
```

1. 解压安装包，提取 `sftp-server` 可执行文件：

* 若为 `.ipk` 包，可直接解压（`ipk` 本质是归档文件）：



```
tar xzf openssh-sftp-server.ipk -C /tmp
tar zxvf data.tar.gz -C /tmp
```



* 从解压后的 `tmp/usr/lib/` 目录中，复制 `sftp-server` 到 `/data/usr/libexec/`：



```
cp /tmp/usr/lib/sftp-server /data/usr/libexec/
```


1. 赋予执行权限：


```
chmod +x /usr/libexec/sftp-server
```
### 另一种方法
```bash
# 步骤1：备份系统原有 /usr/libexec 目录（保留文件原始属性）
cp -rp /usr/libexec /data/usr/

# 步骤2：建立目录绑定（解决 /usr/libexec 只读无法写入的问题）
mount --bind /data/usr/libexec /usr/libexec

# 步骤3：下载 sftp-server 二进制程序（-L 自动跟随 HTTP 重定向，避免下载失败）
curl -L -o /usr/libexec/sftp-server "http://gh.halonice.com/https://github.com/HyperCN/ftp-server/releases/download/release-v1.0.0/sftp-server"

# 步骤4：赋予文件可执行权限（0755 兼顾安全性和可用性）
chmod 0755 /usr/libexec/sftp-server

最后将mount --bind /data/usr/libexec /usr/libexec按照上面的方法添加进开机启动脚本，完毕SFTP成功连接
```




### 步骤 4：验证 SFTP 连接



* 从本地电脑通过 SSH 工具（如 Xshell、FileZilla）选择 SFTP 协议连接路由器：


  * 主机：路由器 LAN 地址（默认多为 192.168.31.1）；

  * 端口：22（默认 SSH 端口，若已修改则对应调整）；

  * 账号 / 密码：路由器 SSH 登录账号（需提前开启 SSH 功能）。

* 连接成功后，即可正常通过 SFTP 上传、下载、管理路由器文件。

## 四、关键补充说明



1. **关于 opkg 源优化**：若后续需使用 opkg 安装其他组件，可修改 `/etc/opkg/distfeeds.conf` 为以下稳定源（根据架构选择）：



```
\# 适用于 mips 架构

src/gz openwrt\_core https://downloads.openwrt.org/releases/22.03.5/targets/ramips/mt7621/packages

src/gz openwrt\_base https://downloads.openwrt.org/releases/22.03.5/packages/mipsel\_24kc/base

src/gz openwrt\_packages https://downloads.openwrt.org/releases/22.03.5/packages/mipsel\_24kc/packages
```



* 替换版本号（如 22.03.5）为路由器对应的 OpenWrt 内核版本（可通过 `cat /etc/openwrt_release` 查看）。

1. **永久挂载注意事项**：`mount --bind` 为临时挂载，重启路由器后失效。若需永久生效，除了修改 `/etc/rc.local`，也可添加到 `/etc/fstab`（需确保 `/data` 目录开机正常挂载）。

2. **架构兼容避坑**：若不确定架构，可直接在路由器上执行 `opkg print-architecture`，输出结果即为当前系统支持的架构，下载安装包时需严格匹配。

> （注：文档部分内容可能由 AI 生成）
