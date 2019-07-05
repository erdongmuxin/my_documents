### 安装二进制版本的docker,适用于Ubuntu,Centos,Fedora等各个版本的linux系统
1. 以centos为例下载64位的docker文件
	
	```shell
	wget https://download.docker.com/linux/static/stable/x86_64/docker-18.09.7.tgz
	```
2. 解压文件并拷贝文件到系统的PATH目录
	```bash
       tar -xzvf docker-18.09.7.tgz
       cp docker/* /usr/bin/
   ```
3. 启动docker

     ```bash
     nohup dockerd &
     ```
4. 可以添加到rc.local用于开机自启(也可以添加docker.sevice,用systemctl来实现开机自启,这种操作更加复杂,这里不做介绍)

     ```shell
     echo '/usr/bin/dockerd &' >> /etc/rc.d/rc.local
     ```
5. docker的日志文件有时候会很大,而阿里云初始磁盘往往只有40G,但是可以通过以下操作更改磁盘,例如,已经将一个100G的硬盘挂到了/opt
     1. 停止docker服务
     2. 转移docker文件
     ```shell
     cd /var/lib
     mv docker/* /opt/docker/
     rm -rf docker
     ln -s /opt/docker/ /var/lib/docker
     ```
     3. 重启docker服务



     


