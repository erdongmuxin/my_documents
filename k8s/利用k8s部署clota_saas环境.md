# 利用k8s部署clota_saas环境

> 基于K8S部署公司的环境,一步一步了解K8S相关知识

- ### 基础架构图

    - 基于阿里云的服务器架构

        ![](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587217915.png)
        
        > 自建服务器需要自制负载均衡,这个写在了另一个学习文档

- ### 安装k8s并把所有节点加入集群

    - 机器:

        - 172.18.122.41 2C16G(master节点,主要管理节点,可以连接外网)
        - 172.18.122.42 2C16G(master节点)
        - 172.18.122.43 2C16G(master节点)
        - 172.18.122.44 2C16G(node节点)
        - 172.18.122.45 2C16G(node节点)

    - 负载均衡IP: 47.106.46.31

    - 安装k8s(首先远程连接至172.18.122.41)

        ```shell
        #!/bin/bash
        
        # 将所有节点加入免密登录(包括自身)
        ssh-keygen
        ssh-copy-id 172.18.122.41 && ssh-copy-id 172.18.122.42 && ssh-copy-id 172.18.122.43 && ssh-copy-id 172.18.122.44 && ssh-copy-id 172.18.122.45
        
        # 先执行以下操作安装一个依赖
        pip install netaddr
        
        # 下载脚本easzup,使用kubeasz版本2.0.3(版本可以根据https://github.com/easzlab/kubeasz自行更新)
        export release=2.0.3
        curl -C- -fLO --retry 3 https://github.com/easzlab/kubeasz/releases/download/${release}/easzup
        chmod +x ./easzup
        cp -p ./easzup /usr/bin/
        
        # 利用easzup工具下载并启动依赖
        easzup -D # 出现[INFO] Action successed : download_all则为成功
        easzup -S # 出现[INFO] Action successed : start_kubeasz_docker则为成功
        
        # 安装单节点的k8s
        yum -y install ansible
        /etc/ansible/tools/easzctl start-aio
        ```

        ```shell
        # 将其他机器加入集群(为了保证速度,建议开多终端分别执行)
        easzctl add-etcd 172.18.122.42 && easzctl add-master 172.18.122.42 # 会提示你为新增的etcd取一个名字,默认已经有了一个名字为etcd-0的资源,取名只要不和他冲突就好,查看已有的etcd可以用命令kubectl get cs
        easzctl add-master 172.18.122.43 
        easzctl add-node 172.18.122.44 && easzctl add-node 172.18.122.45
        easzctl add-etcd 172.18.122.43 # 两个etcd同时进行似乎有点问题,这个放到最后等其他的都执行完了再执行,同样不要和已有的etcd名字冲突了
        ```

        

        > 如果你要复制上面的内容到shell脚本中,请使用vi而不是vim,因为vim在注释行换行之后会默认加上注释符号#,导致第一个注释行之后全都是注释行

    - 修改master节点的调度策略,让pod可以在master节点上调度

        - 执行`kubectl get node`,可以看到有两个master几点是不允许被调度的

            ![1567516224039](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587152371.png)

        - 执行`kubectl edit node 172.18.122.42`,将unschedulabel键值修改为false,保存退出(43机器同理)

            ![1567516280204](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587184029.png)

        - 再次执行`kubectl get node`,各个节点都已经可以被调度了

            ![1567516318273](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587191389.png)

            > 在实际生产环境中,应该用三个配置稍低的master节点,并且不让一般应用的pod调度在master节点上,测试环境机器有限,只好全部让调度了

        - 执行`kubectl get cs`,可以看到etcd也是三个都部署好了

            ![1567339718846](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587240037.png)

- ### 安装harbor(可选)

    > 用于存储自定义的镜像,如果已经有了私有镜像仓库,或者想直接使用docker hub(在更新镜像的时候会比较慢),这一步就可以跳过,我这里使用了阿里云的容器镜像服务以及docker hub的服务者两种并存,想redis,mysql这些不常更新的使用了docker hub,而自己项目的镜像则用了阿里云
    
- ### 制作镜像

    > 根据需要制作需要的镜像,并上传到harbor(或者docker hub),这里我只贴出Dockerfile,制作以及上传的命令请自行学习

    1. 制作redis镜像(需要一个可以使用nslookup命令的redis)(最新更新已经不需要制作这个镜像,可以跳过这一步)

        ```dockerfile
        FROM redis
        MAINTAINER songda <erdongmuxin@163.com>
        RUN  apt-get update && apt-get install -y dnsutils
        CMD ["redis-server"]
        ```

- ### 配置基础服务(我首先创建了clota目录,之后所有的操作都在这个目录下面进行)

    - #### 配置动态pvc(baseservice/)

        > 动态pvc,也就是storage-class,在部署有状态应用比如mysql,redis等具有重要的作用,他会根据你设置的参数,自动向pv请求合适的持久存储空间,pv会自动分配一个存储卷以供使用
        >
        > 这里是利用nfs来做动态pv实现持久化存储,这种方法k8s本身不支持,因此需要额外启动一个pod来实现这个功能
        >
        > 方法一,是比较详细的版本,简单版是方法二,在下面,如果不想深入研究,可以使用方法二
        >
        > 如果你有其他的分布式存储系统,则不需要这样复杂,可以直接创建class-storage资源,具体操作参考[官方文档](https://kubernetes.io/docs/concepts/storage/storage-classes/#introduction)

        1. 下载官方项目(当然,首先应该有个nfs服务,这里不多做介绍)

            ```shell
            git clone https://github.com/kubernetes-incubator/external-storage.git
            ```

        2. 修改相关配置

            ```shell
            cd external-storage/nfs-client/deploy/
            vim deployment.yaml
            vim class.yaml
            ```

            - deployment.yaml

                ```yaml
                apiVersion: v1 # 给pv设置权限
                
                
                kind: ServiceAccount
                metadata:
                  name: nfs-client-provisioner
                ---
                kind: Deployment
                apiVersion: extensions/v1beta1
                metadata:
                  name: nfs-client-provisioner # 必须和上面的权限的name一致
                spec:
                  replicas: 1
                  strategy:
                    type: Recreate # 更新策略,表示所有的pod同时重建,但是这里副本数是1,其实更新策略没有意义
                  template: # pod模板
                    metadata:
                      labels:
                        app: nfs-client-provisioner
                    spec:
                      serviceAccountName: nfs-client-provisioner # 必须和上面的权限name保持一致
                      containers:
                        - name: nfs-client-provisioner
                          image: quay.io/external_storage/nfs-client-provisioner:latest
                          volumeMounts:
                            - name: nfs-client-root  # 将名为nfs-client-root的存储卷挂在到/persistentvolums下 这个目录不允许更改
                              mountPath: /persistentvolumes
                          env:
                            - name: PROVISIONER_NAME # 服务名,可以更改,但是下面的value必须和class名字对应,class名在下一个配置文件设置
                              value: fuseim.pri/ifs 
                            - name: NFS_SERVER
                              value: 172.18.122.41 # 这里填上nfs服务器的地址
                            - name: NFS_PATH
                              value: /home/pv # 这里填上nfs服务器的远程共享目录
                      volumes: # 创建存储卷,名为nfs-client-root,这个存储卷在上面24行被挂载,这两个name必须一致
                        - name: nfs-client-root
                          nfs:
                            server: 172.18.122.41 # 这里填上nfs服务器的地址
                            path: /home/pv # 这里填上nfs服务器的远程共享目录
                      # 注意两个需要自己填写地方没有区别,一模一样,但是意义不一样: 
                      # 上面的是将配置信息写入环境变量,这样"quay.io/external_storage/nfs-client-provisioner:latest"这个容器便能够在nfs服务器上自动创建目录来支持其他容器的持久化存储
                      # 而下面的则是自己利用nfs为自己做了一个持久化存储
              # 因此下面的可以采用别的方式,比如emptydir,hostPath之类的,但是上面的一定要配置好
                ```

            - class.yaml
            
                ```yaml
                apiVersion: storage.k8s.io/v1
                kind: StorageClass
                metadata:
                  name: managed-nfs-storage
                provisioner: fuseim.pri/ifs # 和上一个配置文件的服务名对应
                parameters:
                  archiveOnDelete: "true" # 这里默认是false,不会有备份,当你的pvc被删除时,pv会被直接删除掉,数据则会丢失
                  # 修改为true,当你的pvc被删除时,pv上面的数据会自动备份然后删除
                ```
        
        3. 启动pv
    
            ```shell
        kubectl apply -f rbac.yaml -f deployment.yaml -f class.yaml # 注意三个文件的顺序不要写反了
            ```

        4. 验证
    
            ```shell
            kubectl apply -f test-claim.yaml -f test-pod.yaml # 注意顺序
            
            kubectl get pod -w
            NAME		READY   STATUS      RESTARTS   AGE
            test-pod	0/1		Completed   0          137m
            
            # 当test-pod的状态为Completed的时候,去到nfs服务器上面的/home/pv
            ssh nfs_server
            cd /home/pv
            
            ls -lh
            default-test-claim-pvc-c16af798-c553-4bdf-ab3e-e951c5a2518f # 可以看到目录由"命名空间-pvc_name-pvc-随机字符"组成
            
            ls -lh default-test-claim-pvc-c16af798-c553-4bdf-ab3e-e951c5a2518f/
            -rw-r--r-- 1 root root 0 8月  28 11:26 SUCCESS # 看到SUCCESS的文件则为验证通过
            
            exit # 退出nfs_server
            kubectl delete -f test-pod.yaml -f test-claim.yaml # 删除测试的pvc和pod,注意顺序
            ssh nfs_server
            cd /home/pv
            
            ls -lh
            archived-default-test-claim-pvc-c16af798-c553-4bdf-ab3e-e951c5a2518f # 可以看到备份目录由"archived-命名空间-pvc_name-pvc-和之前相同的随机字符"组成,相当于只加了一个前缀"archived-"
            
            ls -lh archived-default-test-claim-pvc-c16af798-c553-4bdf-ab3e-e951c5a2518f 
            -rw-r--r-- 1 root root 0 8月  28 11:26 SUCCESS
            
            rm -rf archived-* #清理掉测试的目录
        exit # 退出nfs_server
            ```

    - #### 配置pv简单版(方法二,支持自建nfs与阿里云nas,推荐使用)

        1. 修改/etc/ansible/roles/cluster-storage/defaults/main.yml
    
            ```yaml
            storage: # 可以开启多个,但是注意class和name不能有冲突
              nfs:
                enabled: "yes"
                server: "172.18.122.41" # nfs服务器地址,我在128机器上创建了目录/home/pv并且做成了nfs服务器
                server_path: "/home/pv" # 
                storage_class: "class-clota"
                provisioner_name: "nfs-provisioner-01"
            
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
            # 会有一个报错,是因为如果你只启用了nfs,则执行aliyun_nas会报错,反之亦然,所以不用管,看到ok=1,则表示成功了(2.0.3版本已经修复)
            # 验证
            kubectl get pod --all-namespaces | grep nfs-prov # 阿里云则grep aliyun-nas
            # 输出: kube-system nfs-provisioner-01-一串字符 1/1 Running
            # 这里的nfs-provisioner-01就是上面配置里面的provisioner_name
            ```
        
            ![1567314731574](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587274375.png)
        
        3. 通过`kubectl get storageclass`查看创建成功的class
        
            ![1567415932568](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587280271.png)
        
        4. 验证(创建baseservices/pv/test_claim.yml)(首先创建了一个baseservices目录)
        
            ```yml
            kind: PersistentVolumeClaim
            apiVersion: v1
            metadata:
              name: test-claim
              annotations:
                volume.beta.kubernetes.io/storage-class: "class-clota" #注意这里的class要对应main.yml中的class, 可以通过
            命令kubectl get storageclass查看已有的class
            spec:
              accessModes:
                - ReadWriteMany
              resources:
                requests:
                  storage: 1Mi
            ```
        
            ```shell
            kubectl apply -f test_claim.yml
            
            kubectl get pvc
            # 输出:
            NAME         STATUS   VOLUME                        ...
            test-claim   Bound    pvc-af8f3de9-8ef6-4cd0-83c5-      ...
            #状态为Bound则表示成功了
            
            kubectl get pv
            # 可以看到自动创建了一个pv
            
            ```
        
            ![1567314936373](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587298191.png)
        
        5. 进一步验证,链接到172.18.122.41,进入/home/pv目录,可以看到自动为你新建了一个目录
        
            ![1567339814784](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587302546.png)
        
        6. 利用`kubectl delete -f test_claim.yml`删除测试用的pvc
        
            ![1567339955873](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587308110.png)
    
- ### 配置zookeeper集群(完全参考[官方文档](https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/))
  
    1. ##### zookeeper_service(baseservice/zookeeper/01.zookeeper_service.yml)
    
        ```yaml
        apiVersion: v1
        kind: Service # 无头服务,给每个pod分配单独的域名,保证zk之间可以通过唯一但是不同的标识来进行通讯,无头服务是statefulsets(有状态应用)资源必须用到的服务,一般来讲,mysql,redis,zookeeper这种有主从关系,主节点和从节点需要分开处理的应用,都叫做有状态应用,而nginx这类的,只需要保证数量的,称之为无状态应用,一般用deployment资源进行部署
        metadata:
          name: zk-hs
          labels:
            app: zk
        spec:
          ports:
          - port: 2888
            name: server
          - port: 3888
            name: leader-election
          clusterIP: None
          selector:
            app: zk # 通过selector把服务绑定到含有app标签,并且值等于zk的Pod上
        ---
        apiVersion: v1
        kind: Service # clusterIP(默认)服务,保证内网通过2181端口来访问所有的zookeeper集群,访问地址zk-cs:2181
        metadata:
          name: zk-cs
          labels:
            app: zk
        spec:
          ports:
          - port: 2181
            name: client
          selector:
            app: zk # 通过selector把服务绑定到含有app标签,并且标签值等于zk的Pod上
        ```
        
    2. zookeeper_statefulsets(baseservice/zookeeper/02.zookeeper_statefulsets.yml)
    
        ```yaml
        apiVersion: policy/v1beta1
        kind: PodDisruptionBudget # 控制器,配置主动删除pod的策略时候
        metadata:
          name: zk-pdb
        spec:
          selector:
            matchLabels:
              app: zk
          minAvailable: 3 # 保证主动删除pod的时候不会低于3个副本,配置的意义在于,如果你手动删除一个pod会导致副本数量低于3个,控制器会先额外启动一个副本,然后再删除,否则,控制器会先删除pod,再启动一个新的副本,注意两种模式会有先后顺序的不同,另一个参数是maxUnavailable,保证失效的副本不会超过一个定值
        ---
        apiVersion: apps/v1
        kind: StatefulSet # 不同于deployment,statefulsets资源为有状态的应用集,宕机再启动会根据不同的配置进行不同的操作,statefulsets启动的时候,会根据无头服务给每个pod不同的域名和主机名,从0开始依次网上增加,宕机重启之后,ip改变,但是域名和主机名不会变,域名和ip的映射会随之更新
        metadata:
          name: zk
        spec:
          selector:
            matchLabels:
              app: zk # 监控标签app的值为zk的pod数量,使其保持设置的数量
          serviceName: zk-hs # 匹配前面设置的无头服务的名字,给每个pod一个独特的标识
          replicas: 3 # 副本的数量
          updateStrategy: # 设置更新策略为滚动更新
            type: RollingUpdate
          podManagementPolicy: Parallel # 控制策略,表示在手动删除statfulsets的时候,pod同时删除,默认值是orderedReady,表示pod从后往前依次删除,此策略和上面的PodDisruptionBudget不会会产生冲突,要注意的是,此策略只在删除Statefulset的时候生效,直接删除pod是不生效的
          template: # 设置pod模板
            metadata:
              labels:
                app: zk # 设置标签app,并且赋值zk给标签app,必须和上面的matchLabels保持一致,不然statefulset会因为pod的数量不够(因为无法匹配)而一直启动新的pod
            spec:
              affinity: # affinity 尽量避免多个pod部署在一个node上,除非node数量少于pod数量
                podAntiAffinity:
                  requiredDuringSchedulingIgnoredDuringExecution:
                    - labelSelector:
                        matchExpressions:
                          - key: "app"
                            operator: In
                            values:
                            - zk
                      topologyKey: "kubernetes.io/hostname"
              containers: # 容器信息
              - name: kubernetes-zookeeper
                imagePullPolicy: IfNotPresent # 下载镜像的策略,表示如果没有镜像则下载
                image: guiaiy/zookeeper-cluster
                resources: # 限制zk资源利用率
                  requests:
                    memory: "1Gi"
                    cpu: "0.5"
                ports: # 注意这里的ports设置并不能实际暴露容器的端口,容器的端口暴露由命令决定,这里只是方便查看信息
                - containerPort: 2181
                  name: client
                - containerPort: 2888
                  name: server
                - containerPort: 3888
                  name: leader-election
                command:
                - sh
                - -c
                - "start-zookeeper \
                  --servers=3 \
                  --data_dir=/var/lib/zookeeper/data \
                  --data_log_dir=/var/lib/zookeeper/data/log \
                  --conf_dir=/opt/zookeeper/conf \
                  --client_port=2181 \
                  --election_port=3888 \
                  --server_port=2888 \
                  --tick_time=2000 \
                  --init_limit=10 \
                  --sync_limit=5 \
                  --heap=512M \
                  --max_client_cnxns=60 \
                  --snap_retain_count=3 \
                  --purge_interval=12 \
                  --max_session_timeout=40000 \
                  --min_session_timeout=4000 \
                  --log_level=INFO"
                readinessProbe: # 就绪性监测,如果失败,则服务不会转发到这个pod上面
                  exec:
                    command:
                    - sh
                    - -c
                    - "zookeeper-ready 2181"
                  initialDelaySeconds: 10 # 在初始化之后10秒开始监测
                  timeoutSeconds: 5
                livenessProbe: # 存活性监测,如果失败,则pod会被删除,statefulset会另起一个新的pod
                  exec:
                    command:
                    - sh
                    - -c
                    - "zookeeper-ready 2181"
                  initialDelaySeconds: 10
                  timeoutSeconds: 5
                volumeMounts:
                - name: zookeeper # 将名为zookeeper的pvc存储卷挂在到/var/lib/zookeeper上
                  mountPath: /var/lib/zookeeper
              securityContext: # 设置启动用户与组(当你不想以root用户运行的时候)
                runAsUser: 1000
                fsGroup: 1000
          volumeClaimTemplates: # pvc模板,当pod启动的时候,他会自动创建pvc,而pvc则会根据模板的信息向pv发送请求,pv会分配一个合适大小的存储卷以便pod使用
          - metadata:
              name: zookeeper # pvc的名字,statefulset根据这个名字进行调用
              annotations:
                volume.beta.kubernetes.io/storage-class: "class-clota" # 这里的设置是固定的,名字必须对应已有的class名字,详情请查看上面的动态pvc创建
            spec:
              accessModes: [ "ReadWriteOnce" ] # 只允许一次挂载,当被挂载之后,会拒绝其他的挂载请求, 还有ReadWriteMany,ReadOnlyOnce,注意没有ReadOnlyMany
              resources:
                requests:
                  storage: 3Gi # 实际是根据pv的容量走的,所以其实数据超出3G也不会有问题,这里的设置意义不大
        ```
    
    3. 利用`kubectl apply -f 01.zookeeper_service.yml -f 02.zookeeper_statefulsets.yml`创建zookeeper集群
    
    4. 通过`kubectl get  pod -w -l app=zk`查看集群启动过程
    
        ![1567519980373](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587319349.png)
    
    5. 等到3个zk都是running状态时候,查看集群各类信息
    
        ```shell
        kubectl get sts | grep zk # 查看集群整体状态
        kubectl get pod | grep zk # 查看各个Pod的状态
        kubectl get svc | grep zk # 查看服务状态
        kubectl get pvc | grep zk # 查看存储卷请求的状态
        kubectl get pv | grep zk # 查看存储卷状态
        ```
    
        ![1567341161356](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587324688.png)
    
- ### 配置一主两从mysql集群(参考[官方文档](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)):
  
    > 由于官网的mysql集群使用的是root用户同步数据并且没有设置密码,安全性较低,这里做了稍稍的改进(当然,直接购买阿里云RDS更加方便)
    
    1. 配置mysql_service(baseservice/mysql/01.mysql_service.yml)
    
        ```yaml
        apiVersion: v1 # 无头服务,用于标识每一个mysql服务
        kind: Service
        metadata:
          name: mysql
          labels:
            app: mysql 
        spec:
          ports:
          - name: mysql
            port: 3306
          clusterIP: None
          selector:
            app: mysql # 服务会绑定所有拥有app标签,并且值为mysql的pod
        ---
        apiVersion: v1 # clustIP服务,用于连接mysql进行读操作,想要进行写操作,需要单独连接服务器mysql-0.mysql:3306(如果使用mycat或者其他中间件进行读写分离,则不需要设置这个服务)
        kind: Service
        metadata:
          name: mysql-read
          labels:
            app: mysql
        spec:
          ports:
          - name: mysql
            port: 3306
          selector:
            app: mysql
        ---
        apiVersion: v1 # 这个服务单独给mysql主节点暴露了一个地址,将负载均衡转发到集群任意主机的23306端口可以访问mysql主节点,我主要用来初始化数据,初始化数据之后,我会删除这个service
        kind: Service
        metadata:
          name: mysql-out
          labels:
            app: mysql
        spec:
          ports:
          - name: mysql
            port: 3306
            nodePort: 23306
          type: NodePort
          selector:
            statefulset.kubernetes.io/pod-name: mysql-0 
          
        ```
    
    2. 配置mysql_config(baseservice/mysql/02.mysql_config.yml)
    
        ```yaml
        apiVersion: v1
        kind: ConfigMap # 一个特殊的存储类型,将"键-值"存进来,将来可以以"文件名-文件内容"的形式挂载出去
        metadata:
          name: mysql
          labels:
            app: mysql
        data:
          master.cnf: | # mysql_master配置
            [mysqld]
            log-bin
          slave.cnf: | # mysql_slave配置
            [mysqld]
            super-read-only
        ---
        apiVersion: v1
        kind: Secret # 通过base64加密存储的一个键值对
        metadata:
          name: mysqlrootpass
        data:
          password: U29uZ2RhQDIwMTk= # base64编码之后的值,虽然对于专业人员用处不大,但是还是提高了门槛
        ```
    
    3. 配置mysql_statefulsets(baseservice/mysql/03.mysql_statefulsets.yml)
    
        ```yaml
        apiVersion: apps/v1
        kind: StatefulSet
        metadata:
          name: mysql
        spec:
          selector:
            matchLabels:
              app: mysql
          serviceName: mysql # 填写上面设置的无头服务的名字
          replicas: 3 # 副本数量
          template:
            metadata:
              labels:
                app: mysql
            spec:
              affinity: # affinity 尽量避免多个pod部署在一个node上,除非node数量少于pod数量
                podAntiAffinity:
                  requiredDuringSchedulingIgnoredDuringExecution:
                    - labelSelector:
                        matchExpressions:
                          - key: "app" # 标签名
                            operator: In
                            values:
                            - mysql # 标签值
                      topologyKey: "kubernetes.io/hostname"
              initContainers: # 初始化容器,会在完成初始化之后被删除掉
              - name: init-mysql
                image: mysql:5.7
                env:
                - name: MYSQL_ROOT_PASSWORD # 将上面secret中的密码解码后赋值给环境变量
                  valueFrom:
                    secretKeyRef: # name要匹配secret的name,而key则匹配secret中的某个data的名字
                      name: mysqlrootpass
                      key: password
                command:
                - bash
                - "-c"
                - |
                  set -ex
                  # statefulset类型的资源会根据metadata中的name生成主机名,依次为mysql-0,mysql-1,mysql-2,这里匹配结尾,并且k8s会往dns中注册相应的域名,规则是hostname.servicename, servicename在第9行设置,比如主节点的地址就是mysql-0.mysql
                  [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
                  ordinal=${BASH_REMATCH[1]} # bash_rematch会获取上面的匹配值,依次是-0, -1, -2,而bash_rematch[1]则会获取0, 1, 2
                  echo [mysqld] > /mnt/conf.d/server-id.cnf
                  # 将server-id写入配置文件
                  echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
                  # 将之前的配置从configMap里面复制到/mnt/conf.d
                  if [[ $ordinal -eq 0 ]]; then
                    # 尾数为0则表示是主节点
                    cp /mnt/config-map/master.cnf /mnt/conf.d/
                  elif [[ $ordinal -eq 1 ]]; then
                    cp /mnt/config-map/slave.cnf /mnt/conf.d/
                    # 此操作是创建同步用户,官方文档中直接用root用户同步因此没有这一步,是我额外加上的一步,又因为init-container是先于pod启动的,所以,在主节点的initcontainer启动的时候,mysql的主节点还没有启动,无法执行这一步的操作,因此,在第一个从节点启动(也就是尾数为1)的时候,上一个节点也就是主节点已经启动,才能执行这一步操作
                    mysql -h mysql-0.mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "grant replication slave,replication client,select on *.* to repli@'%' identified by 'Repli@2019'"
                    # 给repli加上select权限是为了之后进行健康检查
                  else
                    cp /mnt/config-map/slave.cnf /mnt/conf.d/
                  fi
                volumeMounts: # 注意这里,上面我们把config-map的配置文件复制到了/mnt/conf.d下面,执行上面复制操作的意义在于,config-map中的配置是只读文件系统,我们只能通过控制台kubectl edit cm(configmap简写) mysql来修改配置,因此复制出来,才能在容器中用命令进行修改
                - name: conf # 将名为conf的emptydir类型的存储卷挂载到/mnt/conf.d下,然后将config-map挂载出来的文件复制到/mnt/conf.d目录下,文件就被存放到了存储卷上
                  mountPath: /mnt/conf.d
                - name: config-map # 将名为config-map的configMap中的键值以文件名文件内容的方式挂载到/mnt/config-map下,有了这步的挂载,上一步的cp操作才能顺利完成
                  mountPath: /mnt/config-map
              - name: clone-mysql # 这个初始化的容器用于复制主节点的数据
                image: guiaiy/xtrabackup
                imagePullPolicy: IfNotPresent # latest标签的镜像下载规则默认是Alwasy,总会重新下载,修改这个规则,当本地有镜像的时候就不会重新下载了
                command:
                - bash
                - "-c"
                - |
                  set -ex
                  [[ -d /var/lib/mysql/mysql ]] && exit 0 # 数据已经存在则跳过
                  [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
                  ordinal=${BASH_REMATCH[1]}
                  [[ $ordinal -eq 0 ]] && exit 0 # 本身是主节点则跳过
                  # 克隆数据
                  ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
                  # 备份
                  xtrabackup --prepare --target-dir=/var/lib/mysql
                volumeMounts: # 注意,第一个mysqlpvc是动态pvc,和上面的zookeeper一样,系统会自动分配一个合适的大小,第二个conf类型是emptydir,由于在上一个initcontainer将conf挂在到了/mnt/conf.d,并且复制了文件进去,意味着conf下面已经有了文件,然后上一个initcontainer执行完成之后会自动被删掉,但是conf这个持久存储并不会被删掉,此时再挂在到/etc/mysql/conf.d,就相当于把配置从configMap中复制到了/etc/mysql/conf.d,这两个存储的具体设置在yml文件的最后
                - name: mysqlpvc # 申请一个动态pvc用于存储mysql数据
                  mountPath: /var/lib/mysql
                  subPath: mysql
                - name: conf # 将名为conf的emptydir类型的存储卷挂在到/etc/mysql/conf.d下面,注意在上一个initcontainer:init-mysql中,我们已经往这个存储卷cp了内容
                  mountPath: /etc/mysql/conf.d
              # initcontainer会在启动并执行完命令后退出并被删除
              containers: # 真正启动的容器,会在initcontainer启动完成且没有错误之后被启动
              - name: mysql
                image: mysql:5.7
                env:
                - name: MYSQL_ROOT_PASSWORD # 通过环境变量设置root的初始密码
                  valueFrom:
                    secretKeyRef:
                      name: mysqlrootpass
                      key: password
                - name: MYSQL_USER # 设置同步用的账号
                  value: repli
                - name: MYSQL_PASSWORD
                  value: Repli@2019
                ports:
                - name: mysql
                  containerPort: 3306e
                volumeMounts:
                - name: mysqlpvc
                  mountPath: /var/lib/mysql
                  subPath: mysql
                - name: conf
                  mountPath: /etc/mysql/conf.d
                resources:
                  requests:
                    cpu: 100m
                    memory: 1Gi
                livenessProbe: # 美中不足的是,还没有设置主从同步的状态监测,以后有时间改进
                  exec:
                    command: ["mysqladmin", "-h", "127.0.0.1", "-urepli", "-pRepli@2019", "ping"]
                  initialDelaySeconds: 30
                  periodSeconds: 10
                  timeoutSeconds: 5
                readinessProbe:
                  exec:
                    command: ["mysql", "-h", "127.0.0.1", "-urepli","-pRepli@2019", "-e", "SELECT 1"]
                  initialDelaySeconds: 5
                  periodSeconds: 2
                  timeoutSeconds: 1
              - name: xtrabackup # 参考官方的建议,通过xtrabackup容器设置主从同步
                image: guiaiy/xtrabackup
                imagePullPolicy: IfNotPresent
                env:
                - name:  MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysqlrootpass
                      key: password
                ports:
                - name: xtrabackup
                  containerPort: 3307
                command: # 通过xtrabackup设置主从同步,他和mysql容器,共享存储卷,我对xtrabackup不是特别了解,因此直接将官网的同步命令复制过来了
                - bash
                - "-c"
                - |
                  set -ex
                  cd /var/lib/mysql
        
                  # Determine binlog position of cloned data, if any.
                  if [[ -f xtrabackup_slave_info && "x$(<xtrabackup_slave_info)" != "x" ]]; then
                    # XtraBackup already generated a partial "CHANGE MASTER TO" query
                    # because we're cloning from an existing slave. (Need to remove the tailing semicolon!)
                    cat xtrabackup_slave_info | sed -E 's/;$//g' > change_master_to.sql.in
                    # Ignore xtrabackup_binlog_info in this case (it's useless).
                    rm -f xtrabackup_slave_info xtrabackup_binlog_info
                  elif [[ -f xtrabackup_binlog_info ]]; then
                    # We're cloning directly from master. Parse binlog position.
                    [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
                    rm -f xtrabackup_binlog_info xtrabackup_slave_info
                    echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                          MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
                  fi
        
                  # Check if we need to complete a clone by starting replication.
                  if [[ -f change_master_to.sql.in ]]; then
                    echo "Waiting for mysqld to be ready (accepting connections)"
                    
                    # 原本这里没有密码,而我们设置了初始密码,因此这个地方需要修改
                    until mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1"; do sleep 1; done
        
                    echo "Initializing replication from clone position"
                    
                    # 在这里设置自定义的主从信息
                    mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} \
                          -e "$(<change_master_to.sql.in), \
                                  MASTER_HOST='mysql-0.mysql', \
                                  MASTER_USER='repli', \
                                  MASTER_PASSWORD='Repli@2019', \
                                  MASTER_CONNECT_RETRY=10; \
                                START SLAVE;" || exit 1
                    # In case of container restart, attempt this at-most-once.
                    mv change_master_to.sql.in change_master_to.sql.orig
                  fi
        
                  # Start a server to send backups when requested by peers.
                  # 这里也需要加上密码
                  exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
                    "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root --password=${MYSQL_ROOT_PASSWORD}"
                volumeMounts:
                - name: mysqlpvc
                  mountPath: /var/lib/mysql
                  subPath: mysql
                - name: conf
                  mountPath: /etc/mysql/conf.d
                resources:
                  requests:
                    cpu: 100m
                    memory: 100Mi
              volumes: # 这里是静态的存储卷设置,可以看到conf是emptyDir类型,而config-map是configMap类型,他对应的是名为mysql的configMap,是我们上一步设置的,这个configMap有两个key,分别是master.cnf和slave.cnf,而他们的值则是上面设置的内容,当configMap类型的存储卷被挂载的时候,会生成两个文件,文件名则是key(master.cnf和slave.cnf),而文件内容则是key的值
              - name: conf
                emptyDir: {}
              - name: config-map
                configMap:
                  name: mysql
          updateStrategy: # 设置更新策略为滚动更新
            type: RollingUpdate
          volumeClaimTemplates:
          - metadata:
              name: mysqlpvc
              annotations:
                volume.beta.kubernetes.io/storage-class: "class-clota"
            spec:
              accessModes: ["ReadWriteOnce"]
              resources:
                requests:
                  storage: 3Gi
        ```
    
        4. 通过`kubectl apply -f 01.mysql_service.yml -f 02.mysql_config.yml -f 03.mysql_statefulsets.yml`创建mysql集群
    
        5. 通过`kubectl get pod -l app=mysql -w`查看创建过程
    
            ![1567520676280](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587347806.png)
    
        6. 查看集群状态(和zk一样,但是因为mysql的statefulsets一个pod有两个容器,所以可以看到2/2)
    
            ![1567520773540](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587353169.png)
            
        7. 使用`kubectl exec -it mysql-1 /bin/bash`登录其中一个mysql检查一下主从状态
        
            ![1567520912445](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587360849.png)
    
- 配置mycat读写分离服务(购买RDS服务可以忽略mycat)

    1. 配置mycat_service(baseservice/mycat/01.mycat_service.yml)

        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: mycat
          labels:
            app: mycat
        spec:
          ports:
          - port: 8066
            name: server
          selector:
            app: mycat
        ```

    2. 配置mycat_config(baseservice/mycat/02.mycat_config.yml)

        ```yaml
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: mycat
          labels:
            app: mycat
        data:
          schema.xml: |
            <?xml version="1.0"?>
            <!DOCTYPE mycat:schema SYSTEM "schema.dtd">
            <mycat:schema xmlns:mycat="http://io.mycat/">
                <schema name="clota" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1"></schema>
                <schema name="report" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn2"></schema>
                <dataNode name="dn1" dataHost="localhost" database="clota" />
                <dataNode name="dn2" dataHost="localhost" database="report" />
                <dataHost name="localhost" maxCon="1000" minCon="10" balance="3" writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                    <heartbeat>select user()</heartbeat>
                    <!-- 可以配置多个主从 -->
                    <writeHost host="hostM1" url="mysql-0.mysql:3306" user="sunac" password="hqpw@sunac1918">
                        <!-- 可以配置多个从库 -->
                        <readHost host="hostS2" url="mysql-1.mysql:3306" user="sunac" password="hqpw@sunac1918" />
                    </writeHost>
                </dataHost>
            </mycat:schema>
          server.xml: |
            <?xml version="1.0" encoding="UTF-8"?>
            <!DOCTYPE mycat:server SYSTEM "server.dtd">
            <mycat:server xmlns:mycat="http://io.mycat/">
                <system>
                <property name="nonePasswordLogin">0</property>
                <property name="useHandshakeV10">1</property>
                <property name="useSqlStat">0</property>
                <property name="useGlobleTableCheck">0</property>
                <property name="sequnceHandlerType">2</property>
                <property name="subqueryRelationshipCheck">false</property>
                <property name="processorBufferPoolType">0</property>
                <property name="handleDistributedTransactions">0</property>
                <property name="useOffHeapForMerge">1</property>
                <property name="memoryPageSize">64k</property>
                <property name="spillsFileBufferSize">1k</property>
                <property name="useStreamOutput">0</property>
                <property name="systemReserveMemorySize">384m</property>
                <property name="useZKSwitch">false</property>
                <property name="strictTxIsolation">false</property>
                <property name="useZKSwitch">true</property>
                </system>
                <user name="sunac" defaultAccount="true">
                <property name="password">sunac@mycat2019</property>
                <property name="schemas">clota,report</property>
                </user>
                <user name="sunacread">
                <property name="password">sunac@mycat2019</property>
                <property name="schemas">clota,report</property>
                <property name="readOnly">true</property>
                </user>
            </mycat:server>
        ```

    3. 配置mycat_deployment(baseservice/mycat/03.mycat_deployment.yml)

        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: mycat
        spec:
          replicas: 2
          selector:
            matchLabels:
              app: mycat
          template:
            metadata:
              labels:
                app: mycat
            spec:
              nodeSelector:
                roleA: BaseService
              affinity: # affinity 尽量避免多个pod部署在一个node上,除非node数量少于pod数量
                podAntiAffinity:
                  requiredDuringSchedulingIgnoredDuringExecution:
                    - labelSelector:
                        matchExpressions:
                          - key: "app" # 标签名
                            operator: In
                            values:
                            - mycat # 标签值
                      topologyKey: "kubernetes.io/hostname"
              containers:
              - name: mycat
                image: guiaiy/mycat:1.6.6.1
                command:
                - bash
                - "-c"
                - |
                  set -ex
                  cp /mnt/config-map/* /usr/local/mycat/conf/
                  /usr/local/mycat/bin/mycat console
                volumeMounts:
                - name: mycatconfig
                  mountPath: /mnt/config-map
              volumes:
              - name: mycatconfig
                configMap:
                  name: mycat
        ```

- #### 配置一主两从redis(这里我们使用先配置一主三从,然后在缩减的方法,可以体验一下缩放配置的方便性)
  
    1. 配置redis_service(baseservice/redis/01.redis_service.yml)
    
        > 网上的redis配置大多是deployment,并且没有健康状态的检测,因此我对他做了稍微的改进,这里其实有两种版本,一种是用deployment做,一种是我下面的方法,利用statefulset做,因为statefulset才可以申请动态pvc,而deployment则需要自己创建pvc,本着方便的原则,我用了statefulset
        
        ```yaml
        apiVersion: v1 # 无头服务,用于标识每一个redis服务
        kind: Service
        metadata:
          name: redis
          labels:
            app: redis
        spec:
          ports:
          - name: redis
            port: 6379
          clusterIP: None
          selector:
            app: redis
        ---
        apiVersion: v1 # clustIP服务,用于连接redis进行读操作,想要进行写操作,需要单独连接服务器redis-0.redis:6379.
        kind: Service
        metadata:
          name: redis-read
          labels:
            app: redis
        spec:
          ports:
          - name: redis
            port: 6379
          selector:
            app: redis
        ---
        apiVersion: v1 # 这个服务用与单独为主节点设置一个clusterIP,因为如果主节点如果挂掉,他的podIP是会变得,会导致主从同步失效,因此,设置一个service链接到主节点,这个clusterIP是永恒不变的,因此现在进行写操作也可以连接到redis-master:6379,master有了两个连接地址
        kind: Service
        metadata:
          name: redis-master
          labels:
            app: redis
        spec:
          ports:
          - name: redis
            port: 6379
          selector:
            statefulset.kubernetes.io/pod-name: redis-0
         
        ```
        
    2. 配置redis_config(baseservice/redis/02.redis_config.yml)
    
        ```yaml
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: redisconf
        data: # <以下数据请根据实际情况修改>
          redis-master.conf: |
            daemonize no
            port 6379
            timeout 0
            loglevel debug
            logfile /var/log/redis.log
            databases 16
            save 900 1
            save 300 10
            save 60 10000
            rdbcompression yes
            dbfilename dump.rdb
            dir /data
            requirepass %redispass%
            appendonly yes
            appendfsync everysec
            no-appendfsync-on-rewrite no
            auto-aof-rewrite-percentage 100
            auto-aof-rewrite-min-size 64mb
            slowlog-log-slower-than 10000
            slowlog-max-len 1024
          redis-slave.conf: |
            daemonize no
            port 6379
            timeout 0
            loglevel debug
            logfile /var/log/redis.log
            databases 16
            save 900 1
            save 300 10
            save 60 10000
            rdbcompression yes
            dbfilename dump.rdb
            dir /data
            requirepass %redispass%
            appendonly yes
            appendfsync everysec
            no-appendfsync-on-rewrite no
            auto-aof-rewrite-percentage 100
            auto-aof-rewrite-min-size 64mb
            slowlog-log-slower-than 10000
            slowlog-max-len 1024
            slaveof %master-ip% %master-port%
            masterauth %redispass%
        ---
        apiVersion: v1 # 设置redis的密码
        kind: Secret
        metadata:
          name: redispass
        data:
          password: U29uZ2RhQDIwMTk= # base64编码之后的值
        ```
    
    3. 配置redis_statefulsets(baseservice/redis/03.redis_statefulsets.yml)
    
        ```yaml
        apiVersion: apps/v1
        kind: StatefulSet
        metadata:
          name: redis
        spec:
          selector:
            matchLabels:
              app: redis
          serviceName: redis
          replicas: 4 # 先建四个副本,一主三从
          template:
            metadata:
              labels:
                app: redis
            spec:
              affinity: # affinity 尽量避免多个pod部署在一个node上,除非node数量少于pod数量
                podAntiAffinity:
                  requiredDuringSchedulingIgnoredDuringExecution:
                    - labelSelector:
                        matchExpressions:
                          - key: "app" # 标签名
                            operator: In
                            values:
                            - redis # 标签值
                      topologyKey: "kubernetes.io/hostname"
              containers: 
              - name: reids
                image: guiaiy/redis
                imagePullPolicy: IfNotPresent
                ports:
                - name: redis
                  containerPort: 6379
                env:
                - name: GET_HOSTS_FROM # 这个环境变量可以获取k8s的service的一些变量,在这里设置是为了获取之前redis-master服务的IP地址
                  value: env
                - name: REDIS_PASS
                  valueFrom:
                    secretKeyRef:
                      name: redispass
                      key: password
                volumeMounts:
                - name: redispvc
                  mountPath: /data
                  subPath: data
                - name: conf
                  mountPath: /home
                - name: config-map
                  mountPath: /mnt/config-map
                command: 
                - bash
                - "-c"
                - |
                  set -ex
                  [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
                  ordinal=${BASH_REMATCH[1]}
                  # 如果是主节点
                  if [[ $ordinal -eq 0 ]]; then
                    # 因为configmap不可操作,因此只能将其复制出来再进行操作
                    cat /mnt/config-map/redis-master.conf > /home/redis.conf
                    # REDIS_PASS是上面的env定义的密码
                    sed -i "s/%redispass%/${REDIS_PASS}/" /home/redis.conf
                    redis-server /home/redis.conf
                  else
                  # 如果不是主节点
                    cat /mnt/config-map/redis-slave.conf > /home/redis.conf
                    # 下面的${REDIS_MASTER_SERVICE_HOST}就是从上面的环境变量中获取的,redis_master就是服务的名称,也就是redis-master,系统将-自动转换成了_,同理,service_port则是服务的port,也就是6379,其实如果不用上面的环境变量,直接用IP和6379也可以的,只不过,后续如果修改了主节点的端口和ip这里也要修改不够方便,我之前不了解这个知识,因此使用了REDIS_MASTER_HOST=`nslookup redis-master | grep -oP "(?<=s\: )(\d+.)+\d+"`来获取IP,因此才傻傻的做了一个有nslookup命令的redis
                    sed -i "s/%master-ip%/${REDIS_MASTER_SERVICE_HOST}/" /home/redis.conf
                    sed -i "s/%master-port%/${REDIS_MASTER_SERVICE_PORT}/" /home/redis.conf
                    sed -i "s/%redispass%/${REDIS_PASS}/" /home/redis.conf
                    redis-server /home/redis.conf
                  fi
                resources:
                  requests:
                    cpu: 100m
                    memory: 1Gi
                readinessProbe: # 就绪性检测
                  exec:
                    command: 
                    - bash
                    - "-c"
                    - |
                      set -ex
                      result=`redis-cli -a ${REDIS_PASS} ping`
                      [[ $result == "PONG" ]] || exit 1 
                  initialDelaySeconds: 30
                  periodSeconds: 10
                  timeoutSeconds: 5
                livenessProbe: # 存活性检测
                  exec:
                    command: 
                    - bash
                    - "-c"
                    - |
                      set -ex
                      result=`redis-cli -a ${REDIS_PASS} ping`
                      [[ $result == "PONG" ]] || exit 1 
                  initialDelaySeconds: 5
                  periodSeconds: 2
                  timeoutSeconds: 5
              volumes:
              - name: conf
                emptyDir: {}
              - name: config-map
                configMap:
                  name: redisconf
          updateStrategy: # 设置更新策略为滚动更新
            type: RollingUpdate
          volumeClaimTemplates:
          - metadata:
              name: redispvc
              annotations:
                volume.beta.kubernetes.io/storage-class: "class-clota"
            spec:
              accessModes: ["ReadWriteOnce"]
              resources:
                requests:
                  storage: 1Gi
        ```
        
    4. 创建,查看(同上)
       
          ![1567521361255](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587400996.png)
          
        ![1567346398118](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587408860.png)
        
        
        
        
        
    5. 利用`kubectl scale sts redis --replicas=2`将redis集群缩减为2个副本(可以看到,按顺序终止了redis-3和redis-2)
    
          ![1567521672959](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587415023.png)
    
    6. 使用`kubectl get pod -o wide`查看pod的分布状态
    
          ![1567521706660](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587421167.png)
          
    
- ### 配置后端

    1. 后端镜像Dockerfile(后期应该通过jenkins或者gitlab/cicd自动制作镜像,包的名字可以根据实际情况修改)

        ```dockerfile
        FROM guiaiy/jdk
        MAINTAINER songda <erdongmuxin@163.com>
        COPY app.jar /usr/src/app/app.jar
        WORKDIR /usr/src/app/
        CMD ["java","-jar","app.jar"]
        ```
        
        ```dockerfile
        FROM guiaiy/tomcat
        MAINTAINER songda <erdongmuxin@163.com>
        RUN rm -rf /usr/local/tomcat/webapps/*
        COPY webapps/api.war webapps/ROOT.war /usr/local/tomcat/webapps/
        CMD ["catalina.sh","run"]
        ```
    2. 配置后端service(01.backend_service.yml)
    
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: api
        spec:
          selector:
            app: tomcat
          type: ClusterIP
          ports:
          - port: 9000
            targetPort: 8080
        ---
        apiVersion: v1
        kind: Service
        metadata:
          name: manager
        spec:
          selector:
            app: clota-admin
          type: ClusterIP
          ports:
          - port: 9011
            targetPort: 9011
        ```
        
    3. 配置后端deployment
    
        1. biz-app(backend/02.biz_deployment.yml)
    
            ```yml
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: backend-biz
            spec:
              replicas: 2 # 配置副本数量
              selector: # 配置deployment监视的pod特征
                matchLabels:
                  app: biz
              template: # 配置deployment使用的pod模板
                metadata:
                  labels: # 标签要全包含上面的selector,这样deployment才能够监视
                    app: biz
                spec:
                  affinity: # affinity 尽量避免多个pod部署在一个node上,除非node数量少于pod数量
                    podAntiAffinity:
                      requiredDuringSchedulingIgnoredDuringExecution:
                        - labelSelector:
                            matchExpressions:
                              - key: "app" # 标签名
                                operator: In
                                values:
                                - biz # 标签值
                          topologyKey: "kubernetes.io/hostname"
                  initContainers: # 首先检测zk,在zk启动完成之后再启动biz
                  - name: waitforzookeeper
                    image: busybox:1.31.0
                    command: ['sh', '-c', 'until nslookup zk-cs; do echo waiting for zk; sleep 2; done']
                  - name: waitformysql # 然后检测mysql
                    image: busybox:1.31.0
                    command: ['sh', '-c', 'until nslookup mysql-read; do echo waiting for mysql; sleep 2; done']
                  containers:
                  - name: biz-app
                    image: registry-vpc.cn-shenzhen.aliyuncs.com/clota/biz-app:v1.1.1
                    readinessProbe: # 健康监测
                      exec:
                        command:
                        - bash
                        - '-c'
                        - |
                          set -ex
                          result=`ps aux | grep -oP "(?<=java -)jar(?= app)"`
                          [[ $result == "jar" ]] || exit 1
                      initialDelaySeconds: 30
                      periodSeconds: 10
                      timeoutSeconds: 5
                    livenessProbe:
                      exec:
                        command:
                        - bash
                        - '-c'
                        - |
                          set -ex
                          sleep 20
                          result=`/usr/share/zookeeper/bin/zkCli.sh -server zk-cs:2181 ls /dubbo/com.quick.clota.face.members.MemberAccountApi | grep -o providers`
                          [[ $result == "providers" ]] || exit 1
                      initialDelaySeconds: 30
                      periodSeconds: 10
                      timeoutSeconds: 5
                    resources:
                      requests:
                        cpu: 200m
                        memory: 1.5Gi
              strategy: # 更新策略
                rollingUpdate: # 滚动更新
                  maxSurge: 1 # 最多超出的副本数量
                  maxUnavailable: 0 # 最少低于的副本数量
                  # 按照以上配置,deployment会先启动一个新的pod,然后干掉一个旧的pod,再启动一个新的,再干掉一个旧的,知道所有副本都被更新
              # nodeSelector: 指定服务能够使用的节点,测试环境机器不足,因此不做配置,直接分配到所有机器上面
              #   nodeUse: app
            
            ```
        
        2. job(backend/03.job_deployment.yml)
        
            ```yml
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: backend-job
            spec:
              replicas: 2 # 配置副本数量
              selector: # 配置deployment监视的pod特征
                matchLabels:
                  app: job
              template: # 配置deployment使用的pod模板
                metadata:
                  labels: # 标签要全包含上面的selector,这样deployment才能够监视
                    app: job
                spec:
                  affinity: # affinity 尽量避免多个pod部署在一个node上,除非node数量少于pod数量
                    podAntiAffinity:
                      requiredDuringSchedulingIgnoredDuringExecution:
                        - labelSelector:
                            matchExpressions:
                              - key: "app" # 标签名
                                operator: In
                                values:
                                - job # 标签值
                          topologyKey: "kubernetes.io/hostname"
                  initContainers:
                  - name: waitforbiz
                    image: guiaiy/zookeeper-cluster
                    command: 
                    - bash
                    - '-c'
                    - |
                      set -ex
                      result=`zkCli.sh -server zk-cs:2181 ls /dubbo/com.quick.clota.face.members.MemberAccountApi | grep -o providers`
                      until [[ $result == "providers" ]]
                      do
                      echo waiting for biz
                      result=`zkCli.sh -server zk-cs:2181 ls /dubbo/com.quick.clota.face.members.MemberAccountApi | grep -o providers`
                      sleep 2
                      done
                  containers:
                  - name: job-app
                    image: registry-vpc.cn-shenzhen.aliyuncs.com/clota/job-app:v1.1.1
                    readinessProbe: # 健康监测
                      exec:
                        command:
                        - bash
                        - '-c'
                        - |
                          set -ex
                          result=`ps aux | grep -oP "(?<=java -)jar(?= app)"`
                          [[ $result == "jar" ]] || exit 1
                      initialDelaySeconds: 30
                      periodSeconds: 10
                      timeoutSeconds: 5
                    livenessProbe:
                      exec:
                        command:
                        - bash
                        - '-c'
                        - |
                          set -ex
                          result=`/usr/share/zookeeper/bin/zkCli.sh -server zk-cs:2181 ls /dubbo/com.quick.clota.face.members.MemberAccountApi | grep -o providers`
                          [[ $result == "providers" ]] || exit 1
                      initialDelaySeconds: 30
                      periodSeconds: 10
                      timeoutSeconds: 5
                    resources:
                      requests:
                        cpu: 200m
                        memory: 1.5Gi
              strategy: # 更新策略
                rollingUpdate: # 滚动更新
                  maxSurge: 1 # 最多超出的副本数量
                  maxUnavailable: 0 # 最少低于的副本数量
                      # 按照以上配置,更新时,deployment会先启动一个新的pod,然后干掉一个旧的pod,再启动一个新的,再干掉一个旧的,知道所有副本都被更新
                  # nodeSelector: 指定服务能够使用的节点,测试环境机器不足,因此不做配置,直接分配到所有机器上面
                  #   nodeUse: app
            ```
        
        3. tomcat(backend/04.tomcat_deployment.yml)
        
            ```yml
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: backend-tomcat
            spec:
              replicas: 2 # 配置副本数量
              selector: # 配置deployment监视的pod特征
                matchLabels:
                  app: tomcat
              template: # 配置deployment使用的pod模板
                metadata:
                  labels: # 标签要全包含上面的selector,这样deployment才能够监视
                    app: tomcat
                spec:
                  affinity: # affinity 尽量避免多个pod部署在一个node上,除非node数量少于pod数量
                    podAntiAffinity:
                      requiredDuringSchedulingIgnoredDuringExecution:
                        - labelSelector:
                            matchExpressions:
                              - key: "app" # 标签名
                                operator: In
                                values:
                                - tomcat # 标签值
                          topologyKey: "kubernetes.io/hostname"
                  containers:
                  - name: tomcat
                    image: registry-vpc.cn-shenzhen.aliyuncs.com/clota/tomcat:v1.1.1
                    ports:
                    - name: api
                      containerPort: 8080
                    readinessProbe: # 健康监测
                      tcpSocket:
                        port: 8080
                    livenessProbe:
                      tcpSocket:
                        port: 8080
                    resources:
                      requests:
                        cpu: 200m
                        memory: 1.5Gi
              strategy: # 更新策略
                rollingUpdate: # 滚动更新
                  maxSurge: 1 # 最多超出的副本数量
                  maxUnavailable: 0 # 最少低于的副本数量
                  # 按照以上配置,更新时,deployment会先启动一个新的pod,然后干掉一个旧的pod,再启动一个新的,再干掉一个旧的,知道所有副本都被更新
              # nodeSelector: 指定服务能够使用的节点,测试环境机器不足,因此不做配置,直接分配到所有机器上面
              #   nodeUse: app
            
            ```
            
        4. 使用`kubectl apply -f 01.backend_service.yml -f 02.biz_deployment.yml -f 03.job_deployment.yml -f 04.tomcat_deployment.yml`创建后端deployment


- ### 配置前端(具体域名与端口设置请根据实际情况修改)  

  
  1. 配置nginx_dockerfile
  
      ```dockerfile
      FROM nginx
      MAINTAINER songda <erdongmuxin@163.com>
      ADD . /data/
      CMD ["nginx","-g","daemon off;"]
      ```
      
  2. 配置nginx_service(frontend/01.frontend_service.yml)
  
      ```yaml
      apiVersion: v1
      kind: Service
      metadata:
        name: nginx
      spec:
        selector:
          app: nginx
        type: NodePort
        ports:
        - port: 80 # service的port
          targetPort: 80 # 容器暴露的port
          nodePort: 20080 # 通过SLB将80或者443请求转发到任意服务器的20080端口即可访问到这个service对应pod的80端口,客户端-前端使用https链接,而前端-后端依然使用http链接
      ---
      ```
  
  3. 配置nginx_config(用于修改nginx的配置,frontend/02.nginx_config.yaml)
  
      ```yaml
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: nginxconf
      data: # <以下数据请根据实际情况修改>
        clota.conf: |
          gzip on;
          gzip_buffers 4 16k;
          gzip_comp_level 5;
          gzip_types text/plain application/javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
          server {
            listen       80;
            server_name  www.erdongmuxin.cn;
            location / {
              root   /data/web;
              index index.html;
            }
            location /static {
              alias /data/web/static;
            }
            location ~ /mmbiz_(.*)/ {
              proxy_pass         http://www.erdongmuxin.cn;
              proxy_set_header   Host             "mmbiz.qpic.cn";
              proxy_set_header   Referer          "";
            }
             location /admin {
              root /data/admin-web;
              index index.html;
            }
            location /admin/static {
              alias /data/clota-admin/static;
            }
            location /api {
              proxy_pass http://api.default.svc.cluster.local:9000;
              proxy_redirect off;
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header X-Forwarded-Proto $scheme;
            }
            sub_filter "http://mmbiz.qpic.cn" "";
            sub_filter_once off;
          }
          server {
            listen 80;
            server_name api.erdongmuxin.cn;
            location / {
              proxy_pass http://api.default.svc.cluster.local:9000;
              proxy_redirect off;
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header X-Forwarded-Proto $scheme; 
            }
          }
      ```
  
  4. 配置nginx_deployment(frontend/03.nginx_deployment.yml)
  
      ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: frontend-nginx
      spec:
        replicas: 2 # 配置副本数量
        selector: # 配置deployment监视的pod特征
          matchLabels:
            app: nginx
        template: # 配置deployment使用的pod模板
          metadata:
            labels: # 标签要全包含selector,这样deployment才能够监视
              app: nginx
          spec:
            affinity: # affinity 尽量避免多个pod部署在一个node上,除非node数量少于pod数量
              podAntiAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  - labelSelector:
                      matchExpressions:
                        - key: "app" # 标签名
                          operator: In
                          values:
                          - nginx # 标签值
                    topologyKey: "kubernetes.io/hostname"
            containers:
            - name: nginx
              image: registry-vpc.cn-shenzhen.aliyuncs.com/clota/nginx:v1.1.1
              ports:
              - name: http
                containerPort: 80
              volumeMounts:
              - name: nginxconf
                mountPath: /etc/nginx/conf.d/
              readinessProbe:
                tcpSocket:
                  port: 80
              livenessProbe:
                tcpSocket:
                  port: 80
              resources:
                requests:
                  cpu: 200m
                  memory: 1.5Gi
            volumes:
            - name: nginxconf
              configMap:
                name: nginxconf
        strategy: # 更新策略
          rollingUpdate: # 滚动更新
            maxSurge: 2 # 最多超出的副本数量
            maxUnavailable: 0 # 最少低于的副本数量
            # 按照以上配置,更新时,deployment会先启动一个新的pod,然后干掉一个旧的pod,再启动一个新的,再干掉一个旧的,知道所有副本都被更新
        # nodeSelector: 指定服务能够使用的节点,测试环境机器不足,因此不做配置,直接分配到所有机器上面
        #   nodeUse: app
      ```
  
  5. 使用`kubectl apply -f 01.frontend_service.yml -f 02.nginx_config.yml-f 03.nginx_deployment.yml`创建前端
  
- ### 查看最终成果

    ```shell
    kubectl get pod -o wide
    kubectl get svc -o wide
    ```

    ![1567504918233](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587438395.png)

- ### 自动化部署(参考[这个文档])

    > 主要流程就是提交代码-->打包-->制作镜像-->上传镜像-->更新Pod,具体流程请看上面的文档

- 回滚


    - 通过`kubectl rollout --help`查看回滚主要命令
    
        ![1567562155185](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587447483.png)
    
    - 通过`kubectl rollout history`查看历史版本(目前有两个版本)
    
        ![1567562034684](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587453005.png)
    
    - 通过`kubectl rollout undo --to-reversion=`回滚到指定版本(不指定或者指定为0回到上一个版本)
    
        ![1567562225018](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587463440.png)
    
        ![1567562998207](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587473013.png)
    
        可以看到立刻启动了一个新的pod进行滚动更新,启动一个新的然后再干掉一个老的
    
        ![1567563062006](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587479005.png)

- 高可用验证

    > 通过干掉某个节点,或者pod验证高可用
    
    - 干掉pod
    
        > 通过干掉redis来验证redis高可用
    
        - 验证redis主从(主节点存数据,从节点读数据,可以看到主从还是完好的)
    
            ![1567565065614](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587484958.png)
    
        - 通过`kubectl delete pod redis-0`干掉redis主节点并验证重启过程
    
            ![1567565258446](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587489257.png)
    
            ![1567565293808](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587495260.png)
    
        - 再次验证redis主从(可以看到数据没有丢失,新的数据也能同步)
    
            ![1567565408331](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587500193.png)
    
    - 干掉节点
    
        > 直接干掉某一台机器,模拟机器死机的结果
    
        - 链接到172.18.122.44(mysql主节点),执行关机操作,并查看所有pod的流程
    
            ![1567566186150](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587508056.png)
    
            ![1567569879237](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587533328.png)
    
            可以看到,biz最终是在41上重启了一个,但是44机器上的始终没有被干掉,而mysql和redis干脆就没有重启,查了资料发现,stateful类型的资源在节点宕机之后是无法自动重建的,而deployment类型的则可以,statefulset类型的资源在宕机之后需要手动删除或者恢复节点
    
            因此,我写了一个脚本来监控,如果发现有宕机的节点,那么就会强制的去干掉所有正在terminating的pod,然后加入了三个主节点的时间任务,每5分钟运行一次,干掉之后,k8s就会在另外的机器重启pod了
    
            ```shell
            #!/bin/bash
            notReadyNum=`kubect get node | grep -i not`
            if [[ notReadyNum != "" ]];then
            	for i in `kubectl get pod | awk  '/Terminating/{print $1}'`
            	do
            		kubectl delete pod $i --force --grace-period=0
            	done
            fi
            ```
        
        - 刚刚的节点上正好有个mysql主库,等到k8s重新启动一个主库之后,我们登陆从库,看一看主从有没有被断开
        
            ![1567586851973](https://erdongmuxin.oss-cn-shenzhen.aliyuncs.com/小书匠/1567587513505.png)
        
            可以看到,非常完美,虽然我还是建议直接购买RDS的服务
            
            
            
        - 到了这里,开发人员在项目里配置以下参数,打包发布,基本服务就已经可以通过www.erdongmuxin.cn或者https://www.erdongmuxin.cn从外网访问了
        
            ```yaml
            mysql:
              写库: mysql-0.mysql
              读库: mysql-read 
              账号密码需要自定义设置之后交付给开发人员
            redis:
              写库: redis-0.redis或者redis-master
              读库: redis-read
            zookeeper:
              地址: zk-cs
            前端打包指定的api地址: api.erdongmuxin.cn # 设置在nginx配置里,需要将这个地址解析到负载均衡ip
            ```
        
            
        
        
        
        

