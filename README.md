手机SSH软件想要管理文件发现连接不上？
原因是小米路由默认没有安装SFTP
当我尝试通过正常的 opkg 安装 openssh-sftp-server 时，遇到了一个源 404 的错误。为了解决这个问题，我尝试通过修改 /etc/opkg/distfeeds.conf 文件，将多个源添加进去。然而，最终我发现即使我成功添加了这些源，系统的 /overlay 目录空间不足且只读，无法进行写操作。后来我尝试通过挂载 /data 目录到 /overlay，发现虽然 /data/usr 目录是可写的，但是 /usr 目录依然是只读的。
在探索的过程中，我在另一台 OpenWrt 设备上找到了一些启发。我发现 sftp 进程是通过 /usr/libexec/sftp-server 文件启动的，但是 /usr 目录只读。于是，我决定在 /data 目录创建一个相同的目录，并将 /usr/libexec 文件复制到 /data 目录。然后，我通过挂载的方式将 /data/usr/libexec 目录挂载到 /usr/libexec，并且将 sftp 的安装包解压提取出其中的 sftp-server 可执行文件，并拷贝到 /usr/libexec 目录中。然而，这些尝试都没有成功，因为发现 sftp 的可执行文件与当前系统的架构不符。最终，经过多次尝试，我找到了一个适用于当前系统的 sftp 版本，成功解决了问题。
cp -rp /usr/libexec /data/usr/
mount --bind /data/usr/libexec /usr/libexec
curl -L -o /usr/libexec/sftp-server "http://gh.halonice.com/https://raw.githubusercontent.com/HyperCN/ftp-server/refs/heads/main/sftp-server"
chmod 0755 /usr/libexec/sftp-server

最后将mount --bind /data/usr/libexec /usr/libexec按照上面的方法添加进开机启动脚本，完毕SFTP成功连接
