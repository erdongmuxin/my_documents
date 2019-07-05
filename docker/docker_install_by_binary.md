### 安装二进制版本的docker,适用于Ubuntu,Centos,Fedora等各个版本的linux系统
  1. 以centos为例下载64位的docker文件

	`wget https://download.docker.com/linux/static/stable/x86_64/docker-18.09.7.tgz`

> docker文件可以在https://download.docker.com/linux/static/stable/找到自己想要的版本,复制链接地址,替换掉上面的链接即可

  2. 解压文件并拷贝文件到系统的PATH目录

    `tar -xzvf docker-18.09.7.tgz`
    `cp docker/* /usr/bin/`

  3. 启动docker

    `nohup dockerd &`

  4. 可以添加到rc.local用于开机自启(也可以添加docker.sevice,用systemctl来实现开机自启,这种操作更加复杂,这里不做介绍)

    `echo 'dockerd &' >> /etc/rc.d/rc.local`
