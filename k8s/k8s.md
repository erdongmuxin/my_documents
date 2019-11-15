# K8S 学习指南

- ## K8S 安装

    > K8S有很多种安装方法,这里介绍一种比较方便的方法

    - **利用脚本easzup安装**

        > 脚本来源 [https://github.com/easzlab/kubeasz](https://github.com/easzlab/kubeasz) 感谢大佬

        1. 将所有节点加入ssh免密登录(包括自身)

        2. 下载脚本easzup,使用kubeasz版本2.0.3(版本可以根据上面的链接自行更新)

            ```shell
            #!/bin/bash
            export release=2.0.3
            curl -C- -fLO --retry 3 https://github.com/easzlab/kubeasz/releases/download/${release}/easzup
            chmod +x ./easzup
            cp -p ./easzup /usr/bin/
            ```

        3. 利用工具下载依赖

            ```shell
            easzup -D # 出现[INFO] Action successed : download_all则为成功
            easzup -S # 出现[INFO] Action successed : start_kubeasz_docker则为成功
            ```

        4. 利用容器工具在本机上安装k8s的master节点

            ```bash
            #!/bin/bash
            # 本地需要安装ansible才可以正确执行easzctl
            yum -y install ansible
            # 利用工具安装k8s到本机作为master节点
            # 此操作会创建easzctl软连接到/usr/bin/
            pip install netaddr # 这个依赖要先下载
            /etc/ansible/tools/easzctl start-aio
            # 将master节点设置为污节点(不会将pod资源部署在master上面)
            kubectl taint nodes $IP node-role.kubernetes.io/master=:NoSchedule
            ```

        5. 利用容器工具安装其他节点

            ```bash
            #!/bin/bash
            easzctl add-node $IP # 增加node节点
            easzctl del-node $IP # 删除node节点
            # 验证
            kubectl get node
            
            easzctl add-master $IP # 增加另一个master节点
            # 设置为污节点
            kubectl taint nodes $IP node-role.kubernetes.io/master=:NoSchedule
            # 删除同理 验证同理
            
            
            easzctl add-etcd $IP # 增加另一个etcd节点(etcd节点一般为奇数,部署在每一个master上面,需要根据提示给每个etcd节点取一个独立的名字)
            # 删除同理
            # 备份
            ansible-playbook /etc/ansible/23.backup.yml
            # 验证k8s能够识别新的etcd集群
            kubectl get cs
            # 如果验证失败,尝试重新配置运行apiserver，以让 k8s 集群能够识别新的etcd集群
            # ansible-playbook /etc/ansible/04.kube-master.yml -t restart master
            ```

        6. 升级k8s

            ```bash
            #!/bin/bash
            # 下载新的版本
            wget $LINK
            # 解压下载的tar.gz文件
            tar -xf $FILE
            # 替换文件
            cp -p $DIR/kube* /etc/ansible/bin/
            # 升级
            # 切换到待升级的集群(如果有多集群)
            easzctl checkout $CLUSTER_NAME
            # 执行升级
            easzctl upgrade
            ```

- ## **资源配置清单**

    > 可以动过kubectl explain命令查看具体字段信息,比如kubectl explain pod.spec.containers

    - 自主式Pod资源

        ```yaml
        aipVersion: v1 # 版本
        kind: Pod # 类型
        metadata: # 元数据
          name: pod-demo # 名称
          namespace: default # 命名空间
          labels: # 标签,可以定义多个
            app: myapp
            tier: frontend
          annotation: # 注释,用于给pod写一个注释
            erdongmuxin.com/created-by: "songda"
        spec: # 参数定义
          containers: # 容器定义,是一个个的列表
          - name: myapp # <string> 名字
            image: ikubernetes/myapp:v1 # <string> 镜像
            imagePullPlicy: IfNotPresent
            # <string: Always, Never, IfNotPresent> 下载规则: Always表示永远下载,在镜像标签为latest的时候Always是默认项,其他情况IfNotPresent是默认项,Never表示永远使用本地已经存在的镜像
            ports: <[]Object>
            - name: http
              containerPort: 80
              # 这里的暴露仅仅提供信息,让用户知道容器监听了哪个端口,而不是真正的开放端口,也不能阻止端口暴露
              protocol: TCP # default TCP
              hostIP: 0.0.0.0 # default 0.0.0.0
            command: <[]string>
            - "/bin/bash"
            args: <[]string>
            - "-c"
            - "sleep 3600"
            readinessProbe: # 就绪性监测
              httpGet: # http探针
                port: http
                path: /index.html
            livenessProbe: # 存活状态监测
              exec: # exec探针
                command: ["test", "-e", "/tmp/healthy"]
            lifecycle: # 生命周期,启动后命令以及结束前命令
              postStart: # 启动后命令
            exec:
              command: ["/bin/sh", "-c", "echo Home_Page >> /data/web/html/index.html"] # 系统启动的命令不应该强依赖这个命令的结果
              prestop: # 关闭前命令
            # 探针类型有三种: ExecAction, TCPSocketAction, HTTPGetAction
          nodeSelector: # 节点选择器
            disktype: ssd
          restartPolicy: # Always(default), OnFailure, Never # 重启策略
          
        ```

        ##### 使用`kubectl create -f $File`命令创建相关资源

    - Pod控制器

        > ReplicationController
        >
        > ReplicaSet: 副本集资源
        >
        > Deployment: 管理无状态的应用Pod,一个高级的replicaset资源
        >
        > DaemonSet: 确保一个节点只运行一个副本,比如说日志收集服务就可以应用这种控制器
        >
        > Job: 
        >
        > Cronjob: 
        >
        > StatefulSet: 管理有状态的应用Pod

    - ReplicaSet

        - 资源清单格式

            ```yaml
            apiVersion: apps/v1
            kind: ReplicaSet
            metadata:
              name: myapp
              namespace: default
            spec:
              replicas: 2 # 副本数量
              selector: # 标签选择器
                matchLabels:
                  app: myapp
                  release: canary
              template:
                metadata:
                  name: myapp-pod
                  labels:
                    app: myapp
                    release: canary
                    environment: qa
                spec:
                  containers:
                  - name: myapp-container
                    image: ikubernetes/myapp:v1
                    ports:
                    - name: http
                      containerPort: 80           
            ```

    - Deployment

        - 资源清单格式

            ```yaml
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: myapp-deploy
              namespace: default
            spec:
              replicas: 5
              selector:
                matchLabels:
                  app: myapp
                  release: canary
              template:
                metadata:
                  labels:
                    app: myapp
                    release: canary
                spec:
                  containers:
                  - name: myapp-container
                    image: ikubernetes/myapp:v1
                    ports:
                    - name: http
                      containerPort: 80
              strategy: # 更新策略
                rollingUpdate:
                  maxSurge: 1
                  maxUnavailable: 0
            ```

            > 使用`kubectl apply -f $File`命令创建或者更新
            >
            > 使用`kubectl patch $deploy_name -p $json_config`打补丁
            >
            > 使用`kubectl rollout histroy deployment $deploy_name` 来查看历史版本
            >
            > 使用`kubectl roullout undo deployment $deploy_name --to-revision=<int>`命令回滚到某个版本
            >
            > 使用`kubectl rollout pause deployment $deploy_name`来暂停更新(一般用于金丝雀发布)
            >
            > 使用`kubectl rollout resume deployment $deploy_name`来继续更新

    - DeamonSet

        - 资源清单格式

            ```yaml
            apiVersion: apps/v1
            kind: DaemonSet
            metadata:
              name: myapp-ds
              namespace: default
            spec:
              selector:
                matchLabels:
                  app: filebeat
                  release: stable
              template:
                metadata:
                  labels:
                    app: filebeat
                    release: stable
                spec:
                  containers:
                  - name: filebeat
                    image: ikubernetes/filebeat:5.6.5-alpine
                    env:
                    - name: REDIS_HOST
                      value: redis.default.svc.cluster.local # 服务名.名称空间.内建的本地域名后缀
                    - name: REDIS_LOG_LEVEL
                      value: info
                  updateStrategy:
                    type: rollingUpdate
                    rollingUpdate:
                      maxUnavailable: 
            ```

            > 使用`kubectl apply -f $File`创建更新

    - 多资源定义

        - 资源清单格式

            ```yaml
            apiVersion: ...
              # ...
            ---
            apiVersion: ...
              # ...
            ---
            apiVersion: ...
              # ...
            ```

            > StatefulSet相当复杂,挪到存储卷之后再研究

    - Service

        > - 工作模式: userspace, iptables, ipvs
        > - 类型:
        >     - ClusterIP: 仅用于集群内部通讯
        >     - NodePort: 外部可以通过nodeIP:Port来访问
        >     - LoadBalance:
        >     - ExternalName:
        >     - Ingress: 将域名转发到service,相当于nginx的功能
        > - 资源记录: 服务名.名称空间.内建的本地域名后缀(svc.cluster.local.)

        - 资源清单格式

            ```yaml
            apiVersion: v1
            kind: Service
            metadata:
              name: redis
              namespace: default
            spec:
              selector:
                app: redis
                role: logstor
              clusterIP: 10.68.216.97 # 设置为None的时候则为无头service,直接将域名解析到PodIp上,而不再解析为clusterIP
              sessionAffinity: ClientIp # 同一用户访问同一机器,相当于nginx的hash
              type: NodePort
              ports:
              - port: 6379
                targetPort: 6379
                nodePort: 36379 # 不指定则会随机建立,会在每个节点上开放这个端口,因此不允许冲突
            ```

        - Ingress资源清单格式

            ```yaml
            apiVersion: extensions/v1beta1
            kind: Ingress
            metadata:
              name: ingress-myapp
              namespace: default
            spec:
              rules:
              - host: edu.songda.com
                http:
                  paths:
                  - path:
                    backend:
                      serviceName: myapp
                      servicePort: 80
            ```

        - Ingress配置https(以tomcat为例)

            - 前期配置

                ```shell
                #!/bin/bash
                # 生成签名文件(已经有签名文件则可以跳过这一步)
                openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=traefik.svc.cluster.local"
                # 创建secret
                kubectl create -n kube-system secret tls traefik-cert --key=tls.key --cert=tls.crt
                # 创建traefik-controller,增加https相关配置
                kubectl apply -f /etc/ansible/manifests/ingress/traefik/tls/traefik-controller.yaml
                ```

            - 创建https ingress的yaml文件

                ```yaml
                apiVersion: extensions/v1beta1
                kind: Ingress
                metadata:
                  name: ingress-tomcat
                  namespace: default
                  annotations:
                    kubernetes.io/ingress.class: traefik
                spec:
                  rules:
                  - host: edu.songda.com
                    http:
                      paths:
                      - path:
                        backend:
                          serviceName: tomcat
                          servicePort: 8080
                  tls:
                  - secretName: traefik-cert
                ```

            - 在相应名称空间创建secret(也可以写入上面的yaml文件)

                ```shell
                #!/bin/bash
                kubectl create -n default secret traefik-cert --key=tls.key --cert=tls.crt
                
                ```

    - **Volumes**

        > Pod级别:定义在Pod中

        - Empytdir

            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: pod-myapp
              namespace: default
              labels:
                app: myapp
                tier: frontend
            spec:
              containers:
              - name: myapp
                image: ikubernetes/myapp:v1
                volumeMounts:
                - name: html
                  mountPath: /data/web/html/
              - name: busybox
                image: busybox
                imagePullPolicy: IfNotPresent
                command: ["/bin/sh", "-c", "$(date) >> /data/index.html"]
                volumeMounts:
                - name: html
                  mountPath: /data/
              volumes:
              - name: html
                emptyDir: {}
                
            ```

        - HostPath

            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: pod-vol-hostpath
              namespace: default
            spec:
              containers:
              - name: myapp
                image: ikubernetes/myapp:v1
                volumeMounts:
                - name: html
                  mountPath: /usr/share/nginx/html/
              volumes:
              - name: html
                hostPath:
                  path: /data/pod/volume1
                  type: DirectoryOrCreate
            ```

        - Nfs

            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: pod-vol-hostpath
              namespace: default
            spec:
              containers:
              - name: myapp
                image: ikubernetes/myapp:v1
                volumeMounts:
                - name: html
                  mountPath: /usr/share/nginx/html/
              volumes:
              - name: html
                nfs:
                  path: /data/volumes
                  server: 172.18.122.34
            ```

        - pv

            ```yaml
            apiVersion: v1
            kind: PersistentVolume
            metadata:
              name: pv-nfs-001
              labels:
                name: pv-nfs-001
                type: nfs
            spec:
              nfs:
                path: /data/volumes/v1
                server: node002  rwo rox rwx
              accessModes: ["RWX", "RWO", "ROM"]
              capacity:
                storage: 2G
            ```

        - pvc

            ```yaml
            apiVersion: v1
            kind: PersistentVolumeClaim
            metadata:
              name: mypvc
              namespace: default
            spec:
              accessModes: ["ReadWriteMany"]
              resources:
                requests:
                  storage: 6G
            ---
            apiVersion:
            kind: Pod
            metadata:
              name: pod-vol-pvc
              namespace: default
            spec:
              containers:
              - name: myapp
                image: ikubernetes/myapp:v1
                volumeMounts:
                - name: html
                  mountPath: /usr/share/nginx/html/
              volumes:
              - name: html
                persistentVolumeClaim:
                  claimName: mypvc
            ```

        - configmap

            ```yaml
            apiVersion: v1
            kind: ConfigMap
            metadata:
              name: nginx-config
              namespace: default
            data:
              nginx_prot: 80
            server_name: www.songda.com
            ```

            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: pod-cm-1
              namespace: default
              labels:
                app: myapp
                tier: frontend
              annotations:
                songda.com/created-by: songda
            spec:
              containers:
              - name: myapp
                image: ikubernetes/myapp:v1
                ports:
                - name: http
                  containerPort: 80
                env:
                - name: NGINX_SERVER_PORT
                  valueFrom:
                    configMapKeyRef: 
                      name: nginx-config
                      key: nginx_port
                - name: NGINX_SERVER_NAME
                  valueFrom:
                    configMapKeyRef:
                      name: nginx-config
                      key: server_name  
            ```

            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: pod-cm-2
              namespace: default
              labels:
                app: myapp
                tier: frontend
              annotations:
                songda.com/created-by: songda
            spec:
              containers:
              - name: myapp
                image: ikubernetes/myapp:v1
                ports:
                - name: http
                  containerPort: 80
                volumeMounts:
                - name: nginxconf
                  mountPath: /etc/nginx/config.d/
                  readOnly: true
              volumes:
              - name: nginxconf
                configMap:
                  name: nginx-config
                    
              # 会在容器内部/etc/nginx/config.d/下面生成两个链接文件,nginx_port和server_name,文件内容是80和www.songda.com
            ```

            ```yaml
            # 创建两个文件www1.conf和www.conf,内容如下
            # www1.conf
            server {
            	listen 80;
            	server_name www1.songda.com;
            	root /data/web/html;
            }
            # www2.conf
            server {
            	listen 8080;
            	server_name www2.songda.com;
            	root /data/web/html;
            }
            # 将文件加入configmap,文件名是键,文件内容是值
            kubectl create cm nginx-www --from-file=./
            ```

            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: pod-cm-3
              namespace: default
              labels:
                app: myapp
                tier: frontend
              annotations:
                songda.com/created-by: songda
            spec:
              containers:
              - name: myapp
                image: ikubernetes/myapp:v1
                ports:
                - name: http
                  containerPort: 80
                volumeMounts:
                - name: nginxconf
                  mountPath: /etc/nginx/conf.d/
                  readOnly: true
              volumes:
              - name: nginxconf
                configMap:
                  name: nginx-www
                  items: # nginx-www有两个key,分别是www1.conf,www2.conf,当你只想挂载一个时,需要用items指定
                  - key: www1.conf
                    path: www.conf # 将key的内容传递进这个文件名,这样挂载进容器之后的文件名是www.conf,内容是www1.conf的内容
            ```

        - secret

            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: pod-secret-1
              namespace: default
              labels:
                app: myapp
                tier: frontend
            spec:
              containers:
              - name: myapp
                image: ikubernetes/myapp:v1
                ports:
                - name: http
                  containerPort: 80
                env:
                - name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysql-root-password
                      key: password
            ```

    - statefulset

        > 需要headnessService,stateful控制器,volumeClaimTemplates

        - 资源清单格式

            ```yaml
            apiVersion: v1
            kind: Service
            metadata:
              name: myapp
              labels:
                app: myapp
            spec:
              ports:
              - port: 80
                name: web
              clusterIP: None # 无头服务必须指定为None
              selector:
                app: myapp-pod
            ---
            apiVersion: apps/v1
            kind: StatefulSet
            metadata:
              name: myapp
            spec:
              serviceName: myapp
              replicas: 3
              selector:
                matchLabels:
                  app: myapp-pod
              template:
                metadata:
                  labels:
                    app: myapp-pod
                spec:
                  containers:
                  - name: myapp
                    image: ikubernetes/myapp:v1
                    ports:
                    - containerPort: 80
                      name: web
                    volumeMounts:
                    - name: myappdata
                      mountPath: /usr/share/nginx/html/
              volumeClaimTemplates:
              - metadata:
                  name: myappdata
                spec:
                  accessModes: ["ReadWriteOnce"]
                  resources:
                    requests:
                      storage: 1G
              updateStrategy:
                rollingUpdate:
                  partition: 3 # 指定最多更新的个数
            ```

        - 动态pv(storageclass)
        
            > 在statefulset的时候,手动创建pv和pvc较为繁琐,因此有动态pv
        
            1. 修改/etc/ansible/roles/cluster-storage/defaults/main.yml
        
                ```yaml
                storage: # 可以开启多个,但是注意class和name不能有冲突
                  nfs:
                    enabled: "yes" # 开启nfs动态pv
                    server: "$IP" # nfs服务器地址,我在机器上创建了目录/home/pv并且做成了nfs服务器
                    server_path: "/home/pv" # 共享出去的目录
                    storage_class: "class-clota" # class名称,很重要,pvc根据这个名称申请资源
                    provisioner_name: "nfs-provisioner-01" # pod名字
                
                    # aliyun_nas 参数
                  aliyun_nas:
                    enabled: "no"
                    server: "xxxxxxxxxxx.cn-hangzhou.nas.aliyuncs.com"
                    server_path: "/"
                    storage_class: "class-aliyun-nas-01"
                    controller_name: "aliyun-nas-controller-01"
                ```
        
            2. 创建pv
        
                ```shell
                ansible-playbook /etc/ansible/roles/cluster-storage/cluster-storage.yml
                # 验证
                kubectl get pod --all-namespaces | grep nfs-prov # 阿里云则grep aliyun-nas
                # 输出: kube-system nfs-provisioner-01-一串字符 1/1 Running
                # 这里的nfs-provisioner-01就是上面配置里面的provisioner_name
                kubectl get storageclass # 查看创建成功的动态pv
                ```
        
    - k8s认证与授权
    
        - 查看常见信息
    
            ```shell
            kubectl config --help
            ```
    
        - API
    
            ```yaml
            ObjectURL: /apis/<GROUP>/<VERSION>/namespaces/<NAMESPACE_NAME>/<KIND>/<OBJECT_ID>
            ```
        
        - 私钥证书认证
        
            1. 创建私钥与证书
    
                ```shell
                (umask 077; openssl genrsa --out songda.key 2048)
                openssl req -new -key songda.key -out songda.csr -subj "/CN=songda"
                openssl x509 -req -in songda.csr -CA /etc/kubernetes/ssl/ca.pem -CAkey /etc/kubernetes/ssl/ca-key.pem -CAcreateserial -out songda.crt -days 365
                ```
    
            2. 利用证书创建用户
    
                ```shell
                kubectl config set-credentials songda --client-certificate=./songda.crt --client-key=./songda.key --embed-certs=true
                ```
    
            3. 认证用户,让其可以访问集群
    
                ```shell
                kubectl config set-context context-cluster1-songda --cluster=cluster1 --user=songda 
                # cluseter的名字通过kubectl config view得到
                ```
        
            4. 切换用户
        
                ```shell
                kubectl config use-context context-cluster1-songda 
                kubectl get node # 可以看到,虽然做了认证,但是没有授权,这个新的用户并没有权限做任何操作
                Error from server (Forbidden): nodes is forbidden: User "songda" cannot list resource "nodes" in API group "" at the cluster scope
                ```
        
        - RBAC授权
        
            - role授权(名称空间级别,只对当前名称空间有效)
        
                1. 创建role,并赋予role权限
        
                    ```yaml
                    apiVersion: rbac.authorization.k8s.io/v1
                    kind: Role
                    metadata:
                      name: pods-reader
                      namespace: default
                    rules:
                    - apiGroups: # 设置群组
                      - ""
                      resources: # 设置资源
                      - pods
                      verbs: # 设置权限
                      - get
                      - list
                      - watch
                    ```
        
                2. 创建rolebinding,将用户绑定到role上
        
                    ```yaml
                    apiVersion: rbac.authorization.k8s.io/v1
                    kind: RoleBinding
                    metadata:
                      name: songda
                      namespace: default
                    roleRef:
                      apiGroup: rbac.authorization.k8s.io
                      kind: Role
                      name: pods-reader # 设置绑定的role
                    subjects:
                    - apiGroup: rbac.authorization.k8s.io # 设置绑定的用户
                      kind: User
                      name: songda
                    ```
        
                3. 用户对pod有了权限,但是对其他资源仍然无法访问(记得先切换用户)
        
                    ![1568190105128](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1568190105128.png)
        
            - clusterrole授权(集群级别,对所有名称空间有效,注意切回admin用户)
        
                1. 创建clusterrole
        
                    ```yaml
                    apiVersion: rbac.authorization.k8s.io/v1
                    kind: ClusterRole
                    metadata:
                      name: cluster-reader
                    rules:
                    - apiGroups:
                      - ""
                      resources:
                      - pods
                      verbs:
                      - get
                      - list
                      - watch
                    ```
        
                2. 创建clusterrolebinding,绑定clusterrole
        
                    ```yaml
                    apiVersion: rbac.authorization.k8s.io/v1beta1
                    kind: ClusterRoleBinding
                    metadata:
                      name: songda-all-ns
                    roleRef:
                      apiGroup: rbac.authorization.k8s.io
                      kind: ClusterRole
                      name: cluster-reader
                    subjects:
                    - apiGroup: rbac.authorization.k8s.io
                      kind: User
                      name: songda
                    ```
        
                3. 所有名称空间都有了pod查看权限,但是其他资源则没有(记得先切换用户)
        
                    ![1568190462651](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1568190462651.png)
        
                4. 利用rolebiding绑定clusterrole,可以将clusterrole的权限降级为名称空间级别(删除之前的规则)
        
                    ```yaml
                    apiVersion: rbac.authorization.k8s.io/v1
                    kind: RoleBinding
                    metadata:
                      name: rolebiding-clusterrole
                    roleRef:
                      apiGroup: rbac.authorization.k8s.io
                      kind: ClusterRole
                      name: cluster-reader
                    subjects:
                    - apiGroup: rbac.authorization.k8s.io
                      kind: User
                      name: songda
                    ```
                    
        5. 验证(切换用户验证)
                
            ![1568191024574](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1568191024574.png)
                
        6. 用系统的admin权限为每个名称空间设置管理员(切换回admin操作)
                
            ```yaml
                    apiVersion: rbac.authorization.k8s.io/v1
                    kind: RoleBinding
                    metadata:
                      creationTimestamp: null
                      name: default-admin
                    roleRef:
                      apiGroup: rbac.authorization.k8s.io
                      kind: ClusterRole # 权限设置为admin的clusterrole
                      name: admin
                    subjects:
                    - apiGroup: rbac.authorization.k8s.io
                      kind: User
                      name: songda
                    ```
                
        7. 切回songda用户,可以看到拥有了所有的权限
                
            ![1568191613350](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1568191613350.png)
                
        8. 使用`openssl x509 -in /etc/kubernetes/ssl/admin.pem -text -noout`查看admin的分组
            
        









