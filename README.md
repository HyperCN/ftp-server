小米路由器部署 SFTP 服务问题排查与解决全流程
一、问题背景
手机 SSH 软件连接小米路由器后，仅能执行命令但无法管理文件，核心原因是小米路由器（基于 OpenWrt 定制）默认未预装 openssh-sftp-server 组件。
二、遇到的核心障碍
1. 包管理器源失效（opkg 404 错误）
通过常规命令安装 openssh-sftp-server 时，触发 404 错误：
bash
运行
opkg install openssh-sftp-server
原因是默认 /etc/opkg/distfeeds.conf 内的软件源失效或与设备版本不匹配。
2. 系统分区只读且空间不足
尝试修改 distfeeds.conf 添加多组软件源后，依然无法完成安装。
核心限制：系统 /overlay 目录空间不足且为只读挂载，这是定制 OpenWrt 设备的常见特性，系统分区（如 /usr）固化，不支持写入操作。
3. 挂载目录权限不匹配
尝试将可写的 /data 目录挂载到 /overlay 临时扩容，结果如下：
/data/usr 目录具备可写权限；
系统原生 /usr 目录仍为只读状态，无法写入 SFTP 相关执行文件。
4. 执行文件架构不兼容
参考另一台正常运行的 OpenWrt 设备，发现 SFTP 服务依赖 /usr/libexec/sftp-server 可执行文件，因此执行以下操作：
在 /data 目录创建同名目录 /data/usr/libexec；
将目标 sftp-server 文件拷贝到该目录，并尝试挂载到系统 /usr/libexec；
解压其他 openssh-sftp-server 安装包提取执行文件，替换后仍无法启动。
最终定位：提取的 sftp-server 文件与小米路由器的 CPU 架构不匹配。
三、解决思路与操作步骤
1. 核心方向
放弃 opkg 常规安装，转而手动匹配与当前设备架构兼容的 sftp-server 可执行文件。
2. 关键操作逻辑
确认设备架构：执行命令查看设备 CPU 架构，确定目标文件的架构类型（如 mipsel、aarch64 等）
bash
运行
uname -m
寻找匹配版本：从 OpenWrt 官方镜像站或第三方可靠源，下载与设备架构、系统版本对应的 openssh-sftp-server 相关包。
提取可执行文件：解压下载的安装包，提取核心文件 sftp-server。
挂载可写目录：将提取的 sftp-server 放入 /data/usr/libexec 目录，通过挂载命令将该目录映射到系统只读的 /usr/libexec：
bash
运行
mount --bind /data/usr/libexec /usr/libexec
配置并启动 SFTP：修改 SSH 配置文件，启用 SFTP 服务并指定执行文件路径，重启 SSH 服务验证连接。
四、最终结果
找到与小米路由器系统架构完全匹配的 sftp-server 版本后，完成挂载与配置，手机 SSH 软件成功通过 SFTP 协议连接路由器，实现文件管理功能。
五、总结
定制化 OpenWrt 设备（如小米路由器）存在系统分区只读的特性，常规 opkg 安装可能受限于源失效和分区权限。
手动部署软件时，架构匹配是核心前提，需先确认设备 CPU 架构再寻找对应文件。
mount --bind 命令是解决「只读系统目录」与「可写数据目录」映射的关键手段。
```bash
# 步骤1：备份系统原有 /usr/libexec 目录（保留文件原始属性）
cp -rp /usr/libexec /data/usr/

# 步骤2：建立目录绑定（解决 /usr/libexec 只读无法写入的问题）
mount --bind /data/usr/libexec /usr/libexec

# 步骤3：下载 sftp-server 二进制程序（-L 自动跟随 HTTP 重定向，避免下载失败）
curl -L -o /usr/libexec/sftp-server "http://gh.halonice.com/https://raw.githubusercontent.com/HyperCN/ftp-server/refs/heads/main/sftp-server"

# 步骤4：赋予文件可执行权限（0755 兼顾安全性和可用性）
chmod 0755 /usr/libexec/sftp-server

最后将mount --bind /data/usr/libexec /usr/libexec按照上面的方法添加进开机启动脚本，完毕SFTP成功连接
