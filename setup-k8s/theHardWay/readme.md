<h1> Setup Multi-Master Kubernetes Cluster - THE HARD WAY </h1>
<p>This article will help you to setup and install Multi-Master kubernetes cluster THE HARD WAY using Vagrant and VirtualBox on Ubuntu:18.04LTS.</p>

***

<h2>Step:-1 - Setup VM for Master, worker, Loadbalancer and Admin </h2>

Make sure you are at location `github/imagincloud/kubernetes/setup-k8s/theHardWay`

COMMAND:- `vagrant up`

<h2>Step:-2 - Install kubectl in Admin VM</h2>

<h3>Step:-2.1 - SSH in Admin VM</h3>

COMMAND:- `vagrant ssh admin`
<p>You must use a kubectl version that is within one minor version difference of your cluster. 
For example, a v1.22 client can communicate with v1.21, v1.22, and v1.23 control planes. </p>

<h3>Step:-2.2 - Download kubectl</h3>

COMMAND:-  `wget https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kubectl`


<h3>Step:-2.3 - Change mode</h3>

COMMAND:-  `chmod +x kubectl`


<h3>Step:-2.4 - Move to executable dir</h3>

COMMAND:-  `sudo mv kubectl /usr/local/bin/`

<h3>Step:-2.5 - Verify kubectl version 1.22.0 or higher is installed</h3>

COMMAND:-  `kubectl version --client --short`

<h3>Step:-2.6 - verify if any config </h3>

COMMAND:-  `kubectl config view`

<h2>Step:-3 - Provisioning a CA and Generating TLS Certificates</h2>

<p>The kube-apiserver certificate requires all names that various components may reach it to be part of the alternate names. These include the different DNS names, and IP addresses such as the master servers IP address, the load balancers IP address, the kube-api service IP address etc.</p>

The openssl command cannot take alternate names as command line parameter.

<h3>Step:-3.1 - create a conf file for KubeAPIServer</h3>

<h4>Make sure you are in in admin VM</h4>

COMMAND:- `vagrant ssh admin` 

COMMAND:- `sudo sed -i '0,/RANDFILE/{s/RANDFILE/\#&/}' /etc/ssl/openssl.cnf` 


```
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 192.168.56.11
IP.3 = 192.168.56.12
IP.4 = 192.168.56.30
IP.5 = 127.0.0.1
EOF
```

<h3>Step:-3.2 - create a conf file for etcd</h3>

```
cat > openssl-etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = 192.168.56.11
IP.2 = 192.168.56.12
IP.3 = 127.0.0.1
EOF
```

<h3>Step:-3.3 - Generating CA and TLS Certificates</h3>

```
{
# Certificate Authority
openssl genrsa -out ca.key 2048
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000

# Admin Certificate
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out admin.crt -days 1000

# The Controller Manager Client Certificate
openssl genrsa -out kube-controller-manager.key 2048
openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager/O=system:kube-controller-manager" -out kube-controller-manager.csr
openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000

# The Kube Proxy Client Certificate
openssl genrsa -out kube-proxy.key 2048
openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy/O=system:kube-proxy" -out kube-proxy.csr
openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-proxy.crt -days 1000

# The Scheduler Client Certificate
openssl genrsa -out kube-scheduler.key 2048
openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler/O=system:kube-scheduler" -out kube-scheduler.csr
openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-scheduler.crt -days 1000

# The Kubernetes API Server Certificate
openssl genrsa -out kube-apiserver.key 2048
openssl req -new -key kube-apiserver.key -subj "/CN=kube-apiserver" -out kube-apiserver.csr -config openssl.cnf
openssl x509 -req -in kube-apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-apiserver.crt -extensions v3_req -extfile openssl.cnf -days 1000

# The ETCD Server Certificate
openssl genrsa -out etcd-server.key 2048
openssl req -new -key etcd-server.key -subj "/CN=etcd-server" -out etcd-server.csr -config openssl-etcd.cnf
openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd-server.crt -extensions v3_req -extfile openssl-etcd.cnf -days 1000

# The Service Account Certificate
openssl genrsa -out service-account.key 2048
openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 1000
}
```

<h3>Step:-3.4 - Moving certificates to both master VM</h3>

```
for instance in master01 master02; do
  scp ca.crt ca.key kube-apiserver.key kube-apiserver.crt \
    service-account.key service-account.crt \
    etcd-server.key etcd-server.crt \
    vagrant@${instance}:~/
done
```

<h2>Step:-4 - Generating Kubernetes Configuration Files For Authenticatio n- Admin VM</h2>
<p> Generating kubeconfig files for the controller manager, kube-proxy, scheduler clients and the admin user.</p>

 <h3>Kubernetes Public IP Address</h3>
  Each kubeconfig requires a Kubernetes API Server to connect to.
 To support high availability the IP address assigned to the load balancer will be used.
 In our case it is 192.168.56.30

<h3>Step:-4.1 - Setting  Env Variable For Loadbalancer VM - Admin VM</h3>

COMMAND:- `LOADBALANCER_ADDRESS=192.168.56.30` 

<h3>Step:-4.2 - Generate a kubeconfig file for the kube-proxy service - Admin VM</h3>

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${LOADBALANCER_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```


<h3>Step:-4.3 - Generate a kubeconfig file for the kube-controller-manager service - Admin VM</h3>

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```


<h3>Step:-4.4 - Generate a kubeconfig file for the kube-scheduler service - Admin VM</h3>

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

<h3>Step:-4.5 - Generate a kubeconfig file for the admin - Admin VM</h3>

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

<h3>Step:-4.6 - Distribute the Kubernetes Configuration Files for worker node - Admin VM</h3>

```
for instance in worker01 worker02; do
  scp kube-proxy.kubeconfig vagrant@${instance}:~/
done
```

<h3>Step:-4.7 - Distribute the Kubernetes Configuration Files for Master node - Admin VM</h3>

```
for instance in master01 master02; do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig vagrant@${instance}:~/
done
```

<h2>Step:-5 - Generating Encryption Key - Admin VM </h2>
<p>Kubernetes stores a variety of data including cluster state, application configurations, and secrets.
To store this data in encrypted format we have to generate the encreption key</p>

<h3>Step:-5.1 - Generate an encryption key - Admin VM </h3>

COMMAND:- `ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)` 

Verify ENCRYPTION_KEY

COMMAND:- `echo $ENCRYPTION_KEY` 

<h3>Step:-5.2 - Generate an encryption key config file - Admin VM </h3>

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

<h3>Step:-5.3 - Move encryption config to master servers  - Admin VM</h3>

```
for instance in master01 master02; do
  scp encryption-config.yaml vagrant@${instance}:~/
done
```

<h2>Step:-6 - Bootstrapping the etcd Cluster - All Masters</h2>

<p>NOTE:- Make sure you perform this step on only all Master VM</p>

<p>Kubernetes components are stateless and store cluster state in etcd,
we will bootstrap a two node etcd cluster and configure it.</p>

<h3>Step:-6.1 - Verify which etcd version is supported with kubernetes version - All Masters</h3>

<h3>Step:-6.1.1 - Download Kuberntes release binary of perticular version - All Masters</h3>

COMMAND:- `wget https://github.com/kubernetes/kubernetes/releases/download/v1.22.0/kubernetes.tar.gz` 

<h3>Step:-6.1.2 - Extract the Binary - All Masters</h3>

COMMAND:- `tar -xzvf kubernetes.tar.gz` 

<h3>Step:-6.1.3 - Read ETCD Version - All Masters</h3>

COMMAND:-  `cat kubernetes/hack/lib/etcd.sh | grep 'ETCD_VERSION=${ETCD_VERSION'`

Output:- ETCD_VERSION=${ETCD_VERSION:-3.5.1

<p>NOTE:- Now, as we know ETCD version which is compatible is 3.5.1 we will use it. </p>

<h3>Step:-6.2 - Download ETCD Binary - In All Masters </h3>

COMMAND:-  `wget -q --show-progress --https-only --timestamping "https://github.com/etcd-io/etcd/releases/download/v3.5.1/etcd-v3.5.1-linux-amd64.tar.gz"`

<h3>Step:-6.3 - Extract ETCD Binary and move to local bin - In All Masters </h3>

```
{
    tar -xvf etcd-v3.5.1-linux-amd64.tar.gz
    sudo mv etcd-v3.5.1-linux-amd64/etcd* /usr/local/bin/
}
```

<h3>Step:-6.4 - Create folder in lib dir to keep all the keys required for binary - In All Masters </h3>

```
{
    sudo mkdir -p /etc/etcd /var/lib/etcd
    sudo cp ca.crt etcd-server.key etcd-server.crt /etc/etcd/
}
```

<h3>Step:-6.5 - Extract private IP and hostname for etcd name, set as env variable- In All Masters </h3>

```
INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
echo $INTERNAL_IP
ETCD_NAME=$(hostname -s)
echo $ETCD_NAME
```
<h3>Step:-6.6 - Create the etcd.service systemd unit file - In All Masters </h3>
<p>All the configuration flags are present at https://etcd.io/docs/v3.4/op-guide/configuration/ </p>

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/etcd-server.crt \\
  --key-file=/etc/etcd/etcd-server.key \\
  --peer-cert-file=/etc/etcd/etcd-server.crt \\
  --peer-key-file=/etc/etcd/etcd-server.key \\
  --trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster master01=https://192.168.56.11:2380,master02=https://192.168.56.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

<h3>Step:-6.6 - Validate all env variable set correctly - In All Masters </h3>

COMMAND:- `cat /etc/systemd/system/etcd.service`

<h3>Step:-6.7 - Start ETCD server - In All Masters </h3>

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

<h3>Step:-6.9 - Verify -  ETCD service started correctly - In All Masters </h3>

COMMAND:- `sudo systemctl status etcd.service`

Make sure the status of etcd.service is `Active: active (running)`

<h3>Step:-6.10 - Optional - Check logs of etcd service - In All Masters </h3>

COMMAND:- `sudo journalctl -u etcd.service --no-pager --output cat -f`

<h3>Step:-6.8 - Verify -  List the etcd cluster members - In All Masters </h3>

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key
```

Output:-
```
1761bef04e125165, started, master01, https://192.168.56.11:2380, https://192.168.56.11:2379, false
d60f170a453bcaf4, started, master02, https://192.168.56.12:2380, https://192.168.56.12:2379, false
```

<h2>Step:-7 - Bootstrapping the Kubernetes Control Plane - In All Masters </h2>

<h3>Step:-7.1 - Create the Kubernetes configuration directory - In All Masters </h3>

COMMAND:- `sudo mkdir -p /etc/kubernetes/config`

<h3>Step:-7.2 - Download and Install the Kubernetes Controller Binaries - In All Masters </h3>

```
wget -q --show-progress --https-only --timestamping \
"https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kube-apiserver" \
"https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kube-controller-manager" \
"https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kube-scheduler" \
"https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kubectl"
```

<h3>Step:-7.3 - Change Mode and Move To Local Bin folder - In All Masters </h3>

```
{
    chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
    sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

<h3>Step:-7.4 - Create Kubernetes lib Dir And Move Certificates - In All Masters </h3>

```
{
    sudo mkdir -p /var/lib/kubernetes/
    sudo cp ca.crt ca.key kube-apiserver.crt kube-apiserver.key \
    service-account.key service-account.crt \
    etcd-server.key etcd-server.crt \
    encryption-config.yaml /var/lib/kubernetes/
}
```

<h3>Step:-7.5 - Get Internal IP address - In All Masters </h3>

```
INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
echo $INTERNAL_IP
```

<h3>Step:-7.6 - Setup Kube-APIserver service - In All Masters</h3>

```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.crt \\
  --enable-admission-plugins=NodeRestriction,ServiceAccount \\
  --enable-swagger-ui=true \\
  --enable-bootstrap-token-auth=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.crt \\
  --etcd-certfile=/var/lib/kubernetes/etcd-server.crt \\
  --etcd-keyfile=/var/lib/kubernetes/etcd-server.key \\
  --etcd-servers=https://192.168.56.11:2379,https://192.168.56.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \\
  --kubelet-client-certificate=/var/lib/kubernetes/kube-apiserver.crt \\
  --kubelet-client-key=/var/lib/kubernetes/kube-apiserver.key \\
  --runtime-config=api/all=true \\
  --service-account-key-file=/var/lib/kubernetes/service-account.crt \\
  --service-cluster-ip-range=10.96.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kube-apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/kube-apiserver.key \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account.key \\
  --service-account-issuer=api \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

<h3>Step:-7.7 - Setup Kubernetes Controller Manager service - In All Masters</h3>

<h4>Step:-7.7.1 - Move kube-controller manager config to shared dir in lib </h4>

COMMAND:- `sudo cp kube-controller-manager.kubeconfig /var/lib/kubernetes/`

<h4>Step:-7.7.1 - Setup kube-controller-manager service - In All Masters</h4>

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=192.168.56.0/24 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca.key \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.crt \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account.key \\
  --service-cluster-ip-range=10.96.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

<h3>Step:-7.7 - Setup Kube Schedular service - In All Masters</h3>

<h4>Step:-7.7.1 - Move kube-scheduler config to shared dir in lib </h4>

COMMAND:- `sudo cp kube-scheduler.kubeconfig /var/lib/kubernetes/`

<h4>Step:-7.7.1 - Setup kube-scheduler server - In All Masters</h4>

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --address=127.0.0.1 \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

<h3>Step:-7.8 - Enable and start the services - In All Masters</h3>

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```
<h3>Step:-7.9 - Validate Kubernetes Component Status - In All Masters</h3>

COMMAND:- `kubectl get componentstatuses --kubeconfig admin.kubeconfig`

Output:- 

```
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
controller-manager   Healthy   ok                              
scheduler            Healthy   ok                              
etcd-0               Healthy   {"health":"true","reason":""}   
etcd-1               Healthy   {"health":"true","reason":""}
```

<h2>Step:-8 - Setup Loadbalancer - In loadbalancer VM</h2>

<p>In loadbalancer (lb) Virtual machine we will install HA Proxy - high availability load balancer and proxy server for TCP and HTTP-based applications that spreads requests across multiple servers.</p>

<h3>Step:-8.1 - Install HAProxy - In loadbalancer VM</h3>

COMMAND:- `sudo apt-get update && sudo apt-get install -y haproxy`

<h3>Step:-8.2 - Create HA proxy configuration - In loadbalancer VM</h3>

```
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg 
frontend kubernetes
    bind 192.168.56.30:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master01 192.168.56.11:6443 check fall 3 rise 2
    server master02 192.168.56.12:6443 check fall 3 rise 2
EOF
```

<h3>Step:-8.3 - Start Service</h3>

COMMAND:- `sudo service haproxy restart`

<h3>Step:-8.4 - Verify Service status</h3>

COMMAND:- `sudo service haproxy status`

>Now we can use loadbalancer VM ip addrease to go for Master nodes of Kubernetes

<h3>Step:-8.5 - Verify HAProxy working - Make a HTTP request for the Kubernetes version info:</h3>

Command:- `curl  https://192.168.56.30:6443/version -k`

Output:-

```
{
  "major": "1",
  "minor": "22",
  "gitVersion": "v1.22.0",
  "gitCommit": "c2b5237ccd9c0f1d600d3072634ca66cefdf272f",
  "gitTreeState": "clean",
  "buildDate": "2021-08-04T17:57:25Z",
  "goVersion": "go1.16.6",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

<h2>Step:-9 - Setup Worker nodes</h2>

<h3>Step:-9.1 - Generate  keys - On Administrator node</h3>
<h4>Step:-9.1.1 - Create configuration files for worker certificate - On Administrator node</h4>

```
cat > openssl-worker01.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = worker01
IP.1 = 192.168.56.21
EOF

cat > openssl-worker02.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = worker02
IP.1 = 192.168.56.22
EOF
```

<h4>Step:-9.1.2 - Create Worker certificates - On Administrator node</h4>

```
{
# Create Certificates for worker01
openssl genrsa -out worker01.key 2048
openssl req -new -key worker01.key -subj "/CN=system:node:worker01/O=system:nodes" -out worker01.csr -config openssl-worker01.cnf
openssl x509 -req -in worker01.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out worker01.crt -extensions v3_req -extfile openssl-worker01.cnf -days 1000

# Create Certificates for worker02
openssl genrsa -out worker02.key 2048
openssl req -new -key worker02.key -subj "/CN=system:node:worker02/O=system:nodes" -out worker02.csr -config openssl-worker02.cnf
openssl x509 -req -in worker02.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out worker02.crt -extensions v3_req -extfile openssl-worker02.cnf -days 1000
}
```

<h3>Step:-9.2 - Generate kube-configuration file for workers - On Administrator node</h3>

```
LOADBALANCER_ADDRESS=192.168.56.30
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://192.168.56.30:6443 \
    --kubeconfig=worker01.kubeconfig

  kubectl config set-credentials system:node:worker01 \
    --client-certificate=worker01.crt \
    --client-key=worker01.key \
    --embed-certs=true \
    --kubeconfig=worker01.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:worker01 \
    --kubeconfig=worker01.kubeconfig

  kubectl config use-context default --kubeconfig=worker01.kubeconfig
}
```

```
LOADBALANCER_ADDRESS=192.168.56.30
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://192.168.56.30:6443 \
    --kubeconfig=worker02.kubeconfig

  kubectl config set-credentials system:node:worker02 \
    --client-certificate=worker02.crt \
    --client-key=worker02.key \
    --embed-certs=true \
    --kubeconfig=worker02.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:worker02 \
    --kubeconfig=worker02.kubeconfig

  kubectl config use-context default --kubeconfig=worker02.kubeconfig
}
```

<h3>Step:-9.3 - Move Certificates and config files to worker ndoes - On Administrator node</h3>

COMMAND:- `scp ca.crt worker01.crt worker01.key worker01.kubeconfig vagrant@worker01:~/`

COMMAND:- `scp ca.crt worker02.crt worker02.key worker02.kubeconfig vagrant@worker02:~/`

<h3>Step:-9.4 - Download kubect, kube-proxy and kubelet - ON EACH WORKER NODE</h3>

```
wget -q --show-progress --https-only --timestamping \
https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kubectl \
https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kube-proxy \
https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kubelet
```

<h3>Step:-9.5 - configure dirs - ON EACH WORKER NODE</h3>

```
sudo mkdir -p \
/etc/cni/net.d \
/opt/cni/bin \
/var/lib/kubelet \
/var/lib/kube-proxy \
/var/lib/kubernetes \
/var/run/kubernetes
```

```
{
  chmod +x kubectl kube-proxy kubelet
  sudo mv kubectl kube-proxy kubelet /usr/local/bin/
}
```

```
{
  sudo mv ${HOSTNAME}.key ${HOSTNAME}.crt /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.crt /var/lib/kubernetes/
}
```

<h3>Step:-9.5 - create kubelet-config - ON EACH WORKER NODE</h3>

```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.96.0.10"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
EOF
```

<h3>Step:-9.6 - create kubelet service - ON EACH WORKER NODE</h3>

```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service
[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.crt \\
  --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}.key \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

<h3>Step:-9.7 - create kubeproxy config - ON EACH WORKER NODE</h3>

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig

cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "192.168.56.0/24"
EOF
```

<h3>Step:-9.8 - create kubeproxy service - ON EACH WORKER NODE</h3>

```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

<h3>Step:-9.9 - Start kubelet and kube-proxy service - ON EACH WORKER NODE</h3>

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kubelet kube-proxy
  sudo systemctl start kubelet kube-proxy
}
```
<h3>Step:-9.10 - Validate service status and check logs  - ON EACH WORKER NODE</h3>

COMMAND:- `sudo systemctl status kubelet.service`

COMMAND:- `sudo journalctl -u kubelet.service --no-pager --output cat -f`

COMMAND:- `sudo systemctl status kube-proxy.service`

COMMAND:- `sudo journalctl -u kube-proxy.service --no-pager --output cat -f`

<h3>Step:-9.11 - Validate Nodes are listed now on each master node  - ON EACH MASTER NODE</h3>

COMMAND:- `kubectl get nodes --kubeconfig admin.kubeconfig`

Output:-

```
NAME       STATUS     ROLES    AGE     VERSION
worker01   NotReady   <none>   2m49s   v1.22.0
worker02   NotReady   <none>   2m46s   v1.22.0
```

<p>The status is NotReady as we are yet to setup the pod network which we will do in next steps</p>

<h2>Step:-10:- Configuring kubectl for Remote Access - On Admin Node</h2>

<h3>Step:-10.1:- configure kube configuration - On Admin Node</h3>

```
{
  KUBERNETES_LB_ADDRESS=192.168.56.30

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${KUBERNETES_LB_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

<h3>Step:-10.2:- Validate kube configuration - On Admin Node</h3>

COMMAND:- `kubectl get nodes`

Output:- 

```
NAME       STATUS     ROLES    AGE     VERSION
worker01   NotReady   <none>   8m2s    v1.22.0
worker02   NotReady   <none>   7m59s   v1.22.0
```

<h2>Step:-11:- Setup Container Netwoking </h2>

<h3>Step:-11.1:- Deploy Calico network - On Admin Node</h3>

COMMAND:- `kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.18/manifests/calico.yaml`


<h3>Step:-11.2:- Install CNI plugins - On ALL WORKER Nodes</h3>

<p>Download the CNI Plugins required for weave on each of the worker nodes</p>

COMMAND:- `wget https://github.com/containernetworking/plugins/releases/download/v1.0.1/cni-plugins-linux-amd64-v1.0.1.tgz`

<h3>Step:-11.3:- Extract it to /opt/cni/bin directory - On ALL WORKER Nodes</h3>

COMMAND:- `sudo tar -xzvf cni-plugins-linux-amd64-v1.0.1.tgz --directory /opt/cni/bin/`

<h3>Step:-12:- Validate Node Status Again - On Admin Node</h3>

COMMAND:- `kubectl get nodes`

Output:- 

```
NAME       STATUS   ROLES    AGE   VERSION
worker01   Ready    <none>   29m   v1.22.0
worker02   Ready    <none>   29m   v1.22.0
```

<h2>Step:-12:- Setup RBAC for Kubelet Authorization  - On Admin Node</h2>

<p>How the container or pod's log are been shown -> via kube-API -> kubelet

but we have not given kube-API server permission to get the logs from kubelet

You can validate this error  by getting logs from any pod Eg:-

</p>

```
vagrant@admin:~$ kubectl get pods -n kube-system
NAME              READY   STATUS    RESTARTS     AGE
weave-net-jjn76   2/2     Running   1 (8h ago)   8h
weave-net-w7rsk   2/2     Running   1 (8h ago)   8h
vagrant@admin:~$ kubectl logs weave-net-jjn76 -c weave-init -n kube-system
Error from server (Forbidden): Forbidden (user=kube-apiserver, verb=get, resource=nodes, subresource=proxy) ( pods/log weave-net-jjn76)

```

<h3>Step:-12.1:- configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node</h3>


```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF


cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kube-apiserver
EOF
```

<h3>Step:-12.2:- Validate if we are able to get the logs from pods</h3>

COMMAND:- `kubectl logs weave-net-jjn76 -c weave-init -n kube-system`

Output:- EMPTY - But No error, As required.

<h2>Step:-13:- Deploy the coredns cluster add-on  - On Admin Node</h2>

<p>Before installing coredns Lets check the dns lookup for kubernetes DNS</p>

<h3>Step:-13.1 :- Deploy a busybox image  - On Admin Node</h3>

COMMAND:- `kubectl run  busybox --image=busybox:1.28 --command -- sleep 3600`

```
kubectl exec -ti busybox -- nslookup kubernetes
Server:    10.96.0.10
Address 1: 10.96.0.10
.
.
and control did not return. That means it cannot resolve the 'kubernetes' DNS

# Important:- to show by default in all the pod's resolve.conf the nameserver is added 10.96.0.10
kubectl exec -ti busybox -- cat /etc/resolv.conf 
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local

```

<h3>Step:-13.2:- Deploy the coredns cluster add-on  - On Admin Node</h3>

<p>Reference:- https://kubernetes.io/docs/tasks/administer-cluster/coredns/#installing-coredns</p>


```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      serviceAccountName: coredns
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
      - name: coredns
        image: coredns/coredns:1.2.2
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.96.0.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
EOF
```

Output:- 

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```


<p>After installing coreDNS - in Admin VM now the DNS kuberntes is resolving to kubernetes.default.svc.cluster.local</p>

```
vagrant@admin:~$ kubectl exec -it busybox -- nslookup kubernetes
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```
