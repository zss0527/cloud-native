### create linux servers via vmware  

1. use `sudo apt install net-tools openssh-server` to install basic tools, check ip address via `ifconfig`, run `sudo systemctl enable ssh`, use `sudo hostnamectl set-hostname node02` to reset hostname, then reboot;  
2. use `ssh larry@192.168.1.11` to connect to node01, and `sudo vim /etc/netplan/50-cloud-init.yaml`, copy below script:  

    ```bashshell
      network:
        version: 2
        ethernets:
          ens33:
            dhcp4: no
            addresses:
              - 192.168.1.21/24
            routes:
              - to: default
                via: 192.168.1.1
            nameservers:
              addresses:
                - 223.5.5.5
                - 223.6.6.6
    ```  

    then run `sudo netplan apply` and ssh login again;  

3. use `sudo vim /etc/hosts` and copy below scripts:  

    ```yaml
    127.0.0.1 localhost
    192.168.1.21 node01
    192.168.1.22 node02
    192.168.1.23 node03
    192.168.1.24 node04
    192.168.1.25 node05
    192.168.1.26 node06
    192.168.1.27 node07
    192.168.1.28 node08
    192.168.1.29 node09
    ```  

4. close swap and time sync  

    ```bashshell
    sudo swapoff -a
    sudo sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab

    sudo apt install -y chrony
    sudo systemctl enable chrony && sudo systemctl start chrony
    ```  

5. clone other servers from the basic one.  


### create k8s cluster via k3s  

1. create master node via `curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.31.5+k3s1 sh -s - server --cluster-init`, then check token via `sudo cat /var/lib/rancher/k3s/server/node-token`.  
2. create other master nodes and join the cluster via `curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.30.9+k3s1 sh -s - server --server https://192.168.1.21:6443 --token K1037f3ae8186f3a33a14d62789321d634f8aaf637ea73ce5f2c8d2216a835d7859::server:cfac9060c256b1a484cf088010178fa4`.  
3. create other worker nodes and join the cluster via `curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.30.9+k3s1 sh -s - agent --server https://192.168.1.21:6443 --token K1037f3ae8186f3a33a14d62789321d634f8aaf637ea73ce5f2c8d2216a835d7859::server:cfac9060c256b1a484cf088010178fa4`.  
4. check nodes status via `sudo kubectl get nodes`, give permisson via `sudo chmod 600 /etc/rancher/k3s/k3s.yaml`, and copy this kubeconfig file to local machine path `~/.kube/config`.  


### create loadbalancer via metallb  

1. `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml`.  
2. create manifest file:  
    ```yml
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: default-pool
      namespace: metallb-system
    spec:
      addresses:
      - 192.168.1.100-192.168.1.120
      autoAssign: true
    ---
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: default-l2-advertisement
      namespace: metallb-system
    spec:
      ipAddressPools:
      - default-pool
    ```  

    apply this file and check status: `kubectl apply -f metallb-ip-pool.yaml; kubectl get pods -n metallb-system`.  

### install and config rancher  

1. install helm via `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`  
2. add related repo: 

    ```bashshell
      helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
      helm repo add jetstack https://charts.jetstack.io
      helm repo update
    ```

3. install cert mananger:  

    ```
    sudo kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.0/cert-manager.crds.yaml
    helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace
    ```

4. install rancher:  

      ```
      sudo kubectl create namespace cattle-system

      sudo helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher.local --set bootstrapPassword=admin123
      #如果从repo中没法安装，可以通过fetch拉取二进制文件进行安装
      helm fetch rancher-stable/rancher --untar
      cd rancher
      helm install rancher . --namespace cattle-system --set hostname=rancher.local --set bootstrapPassword=admin123
      ```

### create redis cluster  

1. add redis-cluster repo via `helm repo add bitnami https://charts.bitnami.com/bitnami;
helm fetch bitnami/redis-cluster`.  
2. `tar -xf redis-cluster-9.0.5.tgz; cd redis-cluster/`, update value.yaml.  
    - 修改密码
    - 将service.type改为LoadBalancer
    - 将cluster.externalAccess.enabled改为true
3. install redis-cluster: `helm install redis-cluster . -n redis`, 之后执行提示的命令去配置externalip.  
4. 如果想卸载redis-cluster: 
    ```
    helm uninstall redis-cluster -n redis
    kubectl delete ns redis
    ```
5. create redisinsight service redisinsight.yaml: 
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata: 
      name: redisinsight
      namespace: redis
    spec:
      replicas: 1
      selector: 
        matchLabels:
          app: redisinsight
      template: 
        metadata: 
          labels: 
            app: redisinsight
        spec:
          containers:
          - name: redisinsight
            image: redislabs/redisinsight:1.12.1
            imagePullPolicy: IfNotPresent
            ports: 
            - containerPort: 8001
            volumeMounts: 
            - name: db
              mountPath:  /db
          volumes:
          - name: db
            emptyDir: {}
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: redisinsight-service
      namespace: redis
    spec:
      type: NodePort
      ports:
      - port: 80
        targetPort: 8001
        nodePort: 31888
      selector:
        app: redisinsight
    ``` 
    kubectl apply -f redisinsight.yaml  
    visit throw http://192.168.1.21:31888 in browser.  


### create kafka cluster  

1. add bitnami repo via `helm repo add bitnami  https://charts.bitnami.com/bitnami` and update repo `helm repo update bitnami`;  
2. pull kafka `helm pull bitnami/kafka` and `tar -xf kafka-23.0.1.tgz`;
3. cd kafak and update values.yaml file  
   - update controller:automountServiceAccountToken: true
   - update broker:automountServiceAccountToken: true and broker:replicaCount: 3
   - update service:type: LoadBalancer
   - update externalAccess:enabled: true and externalAccess:autoDiscovery:enabled: true
   - update rbac:create: true
4. create kafka namespace via `kubectl create ns kafka` and install kafka `helm install -n kafka kafka .`
5. verify status `kubectl get po -n kafka`
6. retrieve password `kubectl get secret kafka-user-passwords --namespace kafka -o jsonpath='{.data.client-passwords}' | base64 -d | cut -d , -f 1`,mXwFSO6biv
7. connection config properties file content:
    ```
    security.protocol=SASL_PLAINTEXT
    sasl.mechanism=SCRAM-SHA-256
    sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="user1" \
    password="mXwFSO6biv";
    ```
8. connection test
   ```
    producer:
    kafka-console-producer --topic test-topic --bootstrap-server 192.168.1.112:9094 --producer.config client.properties
    consumer:
    kafka-console-consumer --topic test-topic --bootstrap-server 192.168.1.112:9094 --consumer.config client.properties --from-beginning
   ```
9. uninstall kafka
    `helm delete kafka  -n kafka`  
    `kubectl delete ns kafka`  