1. 下载
可以从官网<www.msys2.org>上下载，但下载速度较慢，推荐从国内镜像站的`msys2/distrib`目录中下载。

2. 安装
直接双击下载的可执行文件按提示操作即可，没有太多可说明的地方（注意下载PC对应的可执行文件）。

3. 配置
安装完成后，首次使用时需要运行安装目录下的`msys2_shell.cmd`脚本，让其对环境作初始化。
msys2使用的是pacman软件包管理器，其默认配置的是国外的镜像源，下载速度较慢，所以最好配置一下，使用国内的镜像源来加速更新过程。pacman对应的镜像源配置文件在`/etc/pacman.d`目录下，通常命名为mirrorlist.xxx，且该目录下应该有3个类似的文件，分别对应mingw32,mingw64和msys的镜像源配置。使用文本编辑工具在其中添加国内镜像源的地址,
如：在`mirrorlist.mingw64`中添加`Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/x86_64/`来完成mingw64镜像源的配置。配置完成以后，再使用`pacman -Sy`更新数据库，然后就可以对软件包进行更新操作了。

4. 使用
TODO
