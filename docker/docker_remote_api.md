# 【docker】开启remote api访问，并使用TLS加密

背景：

　　docker默认是能使用本地的socket进行管理，这个在集群中使用的时候很不方便，因为很多功能还是需要链接docker服务进行操作，docker默认也可以开启tcp访问，但是这就相当于把整个docker集群对外公开了，很不安全，需要假如TLS进行加密通信，操作如下：

1. TLS配置，生成key文件

    ```bash
    openssl genrsa -aes256 -out ca-key.pem 4096
    openssl req -new -x509 -days 3650 -key ca-key.pem -sha256 -out ca.pem
    openssl genrsa -out server-key.pem 4096
    openssl req -sha256 -new -key server-key.pem -out server.csr
    openssl x509 -req -days 3650 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem
    openssl genrsa -out key.pem 4096
    openssl req -new -key key.pem -out client.csr
    openssl x509 -req -days 3650 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem
    rm -v client.csr server.csr
    chmod -v 0400 ca-key.pem key.pem server-key.pem
    chmod -v 0444 ca.pem server-cert.pem cert.pem
    ```

2. 将ca.pem,server-cert.pem,server-key.pem复制到服务器的/root/.docker目录下

3. 对于二进制安装的docker服务,将开启docker的命令更改为

    ```bash
    nohup dockerd -H unix:///var/run/docker.sock -D -H tcp://0.0.0.0:2375 --tlsverify --tlscacert=/root/.docker/ca.pem -- tlscert=/root/.docker/server-cert.pem --tlskey=/root/.docker/server-key.pem &
    ```
    
    > 注意开机自启的命令也需要同步更改
    
4. 对于yum安装的docker服务,修改execstart配置,并重启服务

    ```bash
    vim  /lib/systemd/system/docker.service
    ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock -D -H tcp://0.0.0.0:2375 --tlsverify --tlscacert=/root/.docker/ca.pem --tlscert=/root/.docker/server-cert.pem --tlskey=/root/.docker/server-key.pem
    systemctl daemon-reload
    systemctl restart docker
    ```

5. 测试:将cert.pem,key.pem这两个文件复制到测试机上, curl中-k的意思是Allow connections to SSL sites without certs，不验证证书

    ```bash
    curl -k https://docker服务器IP:2375/info --cert ./cert.pem --key ./key.pem
    ```

    > 能返回docker信息则为成功,证书请妥善保管

