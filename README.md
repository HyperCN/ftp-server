# 小米路由器 SSH 无法使用 SFTP 管理文件：问题排查与解决方案

## 一、核心问题汇总

1. **默认组件缺失**：小米路由器基于 OpenWrt 定制，默认未安装 `openssh-sftp-server`，导致 SSH 连接后无法通过 SFTP 管理文件；

2. **opkg 源失效**：尝试 `opkg install openssh-sftp-server` 在线安装时，出现源 404 错误，无法正常获取安装包；

3. **目录权限 / 空间问题**：

* 系统 `/overlay` 目录空间不足，且默认挂载为只读，无法写入新文件或安装包；

* 尝试挂载 `/data` 目录（可写）到 `/overlay` 后，`/data/usr` 目录可写，但核心系统目录 `/usr` 仍为只读，无法直接部署 `sftp-server`；

4. **架构不兼容**：初期从其他 OpenWrt 设备复制或解压的 `sftp-server` 可执行文件，与小米路由器 CPU 架构不匹配，无法执行。

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

2. 挂载 `/data/usr/libexec` 到系统 `/usr/libexec`（临时生效，重启后需重新挂载，或添加到开机自启）：

```
mount --bind /data/usr/libexec /usr/libexec
```

* 若需永久生效，编辑 `/etc/rc.local`，在 `exit 0` 前添加上述挂载命令。
* 重启后`/etc/rc.local`文件都复原，方法在[这里](https://www.right.com.cn/forum/thread-8340357-1-1.html)
  

### 步骤 3：获取并部署适配的 sftp-server

1. 下载对应架构的 `openssh-sftp-server` 安装包：

* OpenWrt 官方镜像站：[https://downloads.op](https://downloads.openwrt.org/)[enwrt](https://downloads.openwrt.org/)[.org/](https://downloads.openwrt.org/)

* 小米万兆路由器SFTP插件（需根据型号筛选）；
  
```
cd /data
curl -L openssh-sftp-server.ipk 'https://mirrors.ustc.edu.cn/openwrt/releases/18.06.0/packages/aarch64_generic/packages/openssh-sftp-server_7.7p1-2_aarch64_generic.ipk'
```

2. 解压安装包，提取 `sftp-server` 可执行文件：

* 若为 `.ipk` 包，可直接解压（`ipk` 本质是归档文件）：

```
tar xzf openssh-sftp-server.ipk -C /tmp
tar zxvf data.tar.gz -C /tmp
```

* 从解压后的 `tmp/usr/lib/` 目录中，复制 `sftp-server` 到 `/data/usr/libexec/`：

```
cp /tmp/usr/lib/sftp-server /data/usr/libexec/
```

3. 赋予执行权限：

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
```
最后将mount --bind /data/usr/libexec /usr/libexec添加进开机启动脚本，完毕SFTP成功连接

### 步骤 4：验证 SFTP 连接

* 从本地电脑通过 SSH 工具（如 Xshell、FileZilla）选择 SFTP 协议连接路由器：

  * 主机：路由器 LAN 地址（默认多为 192.168.31.1）；

  * 端口：22（默认 SSH 端口，若已修改则对应调整）；

  * 账号 / 密码：路由器 SSH 登录账号（需提前开启 SSH 功能）。

* 连接成功后，即可正常通过 SFTP 上传、下载、管理路由器文件。

## 四、关键补充说明

* 开机启动脚本
  1. 先创建自启动脚本
  ```
  vim /data/startup_script.sh
  ```
  2. 往脚本中添加内容
     
  ```
  #!/bin/sh
  install() {
        # Add script to system autostart docker
        uci set firewall.startup_script=include
        uci set firewall.startup_script.type='script'
        uci set firewall.startup_script.path="/data/startup_script.sh"
        uci set firewall.startup_script.enabled='1'
        uci commit firewall
        echo -e "\033[32m  startup_script complete. \033[0m"
         }
  uninstall() {
    # Remove scripts from system autostart
    uci delete firewall.startup_script
    uci commit firewall
    echo -e "\033[33m startup_script  has been removed. \033[0m"
    }

  startup_script() {
        # Put your custom script here.
        echo "Starting custom scripts..."
     }

  main() {
    [ -z "$1" ] && startup_script && return
    case "$1" in
    install)
        install
        ;;
    uninstall)
        uninstall
        ;;
    *)
        echo -e "\033[31m Unknown parameter: $1 \033[0m"
        return 1
        ;;
    esac
   }

  main "$@"
  ```
  
  3. 接着给予脚本执行权限
     
  ```
  cd /data
  chmod +x startup_script.sh
  ```
  4. 执行脚本安装命令，这将会设置防火墙指定启动脚本
     
  ```
  ./startup_script.sh install
  ```
  5. 执行完这句install命令，你会发现/etc/config/firewall最后会添加这样一段内容
     
  ```
  config include 'startup_script'
        option type 'script'
        option path '/data/startup_script.sh'
        option enabled '1'
  ```
  
 * 事实上，这和你手动编辑/etc/config/firewall内容效果是一样的，只不过脚本调用了系统的api设置防火墙方法。同理，你调用了uninstall 命令后，这段内容就会被删掉。
 * 如果不传入参数直接执行./startup_scrip.sh，就会执行脚本里面的startup_script函数。
 * 创建好这个通用自启动脚本后，你可以在startup_script里面添加任意你想要的启动命令。

  ```
startup_script() {
        # SFTP必要步骤
        (sleep 12; mount --bind /data/usr/libexec /usr/libexec) &
        # 终端历史记录
        (sleep 14; mount --bind /data/root /root) &
}
  ```

1. **永久挂载注意事项**：`mount --bind` 为临时挂载，重启路由器后失效。若需永久生效，除了修改 `/etc/rc.local`，也可添加到 `/etc/fstab`（需确保 `/data` 目录开机正常挂载）。

2. **架构兼容避坑**：若不确定架构，可直接在路由器上执行 `opkg print-architecture`，输出结果即为当前系统支持的架构，下载安装包时需严格匹配。

## 自用auto_start.sh脚本内容

```
#!/bin/sh
# auto_start.sh - 小米万兆路由器智能挂载脚本（无日志版）
# 功能：精准检测/data挂载后执行绑定，防重复挂载，适配路由启动特性

# ==================== 配置参数（可按需调整） ====================
TARGET_DIR="/data"          # 核心检测目录（小米路由持久化分区）
MAX_WAIT_TIME=30            # 最大等待时间（秒），超时则退出
CHECK_INTERVAL=2            # 检测间隔（秒）
# ================================================================

# 初始化等待计时
wait_seconds=0

# 智能轮询：等待/data分区挂载完成（双重检测：mount列表 + 可写权限）
while [ $wait_seconds -lt $MAX_WAIT_TIME ]; do
    # 检测1：/data在mount列表中；检测2：/data可写（避免挂载但不可用）
    if mount | grep -q "$TARGET_DIR" && [ -w "$TARGET_DIR" ]; then
        break  # 挂载成功，跳出等待循环
    fi
    sleep $CHECK_INTERVAL
    wait_seconds=$((wait_seconds + CHECK_INTERVAL))
done

# 超时保护：/data仍未挂载成功，直接退出（避免无效操作）
if [ $wait_seconds -ge $MAX_WAIT_TIME ]; then
    exit 1
fi

# SFTP目录绑定（防重复：仅当未绑定时报行执行）
if ! mount | grep -q "/data/usr/libexec /usr/libexec"; then
    mount --bind /data/usr/libexec /usr/libexec
fi

# 终端历史记录目录绑定（防重复）
if ! mount | grep -q "/data/root /root"; then
    mount --bind /data/root /root
fi

```
