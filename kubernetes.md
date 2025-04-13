# kubernetes

## tips
- executing a command inside pod `kubectl exec webapp -- cat /log/app.log`

## installing
### installing kubeadm, kubelet, kubectl
- follow this [link](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) for installation
- for installing specific version on the ubuntu after adding kubernetes repository

```bash
apt-get update
apt-cache madison kubeadm # for to see kubeadm versions
apt-get install kubelet=<version> kubeadm=<version> kubectl=<version>
```

- add ipv4 ip forwarding with `sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf`
- add ipv6 ip forwarding with `sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/' /etc/sysctl.conf`
- to enable ip forwarding without restart, run `sysctl -p /etc/sysctl.conf`

### installing container runtime
- select one of the runtimes from the [link](https://kubernetes.io/docs/setup/production-environment/container-runtimes) and follow the installation
#### containerd
- to install containerd on ubuntu, run `apt install -y containerd`
- if `/etc/containerd` doesn't exit, run `mkdir -p /etc/containerd`
- to create default config, run `containerd config default > /etc/containerd/config.toml`

### control groups (cgroup)
- when you want to limit resource on containers **cgroup** is used under the hood
- there are two driver to choose, **cgroupfs** and **systemd**
- if init system is **systemd** on the host, you need use **systemd**.
- to check the init system, run `ps -p 1`
- if the **kubelet** version v1.22 or above, default configuration is **systemd**
- container runtime configuration for **cgroup** is mandatory
- enable **systemd** in containerd configuration with `sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml`
  - then restart **containerd** with `systemctl restart containerd`

### initializing control-plane
- to use control-plane with **high availability** pass `--control-plane-endpoint <virtual_ip>` with **kubeadm**
- to assign ip pools for pods pass `--pod-network-cidr "10.0.0.0/16"` with **kubeadm**
- to choose which network endpoint pass `--apiserver-advertise-address <ip_address>` with **kubeadm**
- to upload certificates as secrest pass `--upload-certs` with **kubeadm**
`kubeadm init --apiserver-advertise-address=192.168.1.77 --pod-network-cidr "10.244.0.0/16" --upload-certs`
- after initializing copy the generated kubectl config to `%HOME/.kube/config` for using **kubectl**
- `kubeadm token` is when you forgot your token

### installing network addon (cni)
- follow this [link](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy) for selecting and installing cni
- **flannel** needs **br_netfilter** module, enable it with `modprobe br_netfilter`

## cluster upgrade
- none of the components' version can higher than `kube-apiserver`
- controller-manager and `kube-scheduler` can be **one** version lower than `kube-apiserver`
- `kubelet` and `kube-proxy` can be **two** version lower than `kube-apiserver`
- `kubectl` can be **one** higher or **one** lower than `kube-apiserver`
- kubernetes support only support recent **three** minor version
- recommended path is update minor versions one at a time without skipping

### kubeadm - upgrade
  - for minor version upgrades edit package manager's kubernetes version
    - for example: for ubuntu edit `/etc/apt/sources.list.d/kubernetes.list` and replace desired version
    - before upgrading the kubeadm list versions and select the patch version for installation for `kubeadm` and `kubelet`
  - only `drain` the node will be upgraded then upgrade `kubelet` and `kubeadm` with the package manager
  - `emre@home ~ → kubeadm upgrade plan <version>`
  - upgrade the `kubeadm` command first with package manager
  - `emre@home ~ → kubeadm upgrade apply <version>`
  - upgrade the `kubelet` command first with package manager and restart the `kubelet` service
  - update the node configuration for the new `kubelet` version
    - `emre@home ~ →  kubeadm upgrade node #config --kubelet-version <version>`
  - restart the `kubelet` service
  - `uncordon` the node when the upgrade is completed
  - repeat for all the nodes

## metada
- custom data for the definition file

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-color
  labels:
    name: webapp-color
```

## commands and arguments
- can pass args to pod with **spec.containers.args** in the pod definition file
  - it is a **string list**
- can override entrypoint with **spec.container.command** in the pod definition file
  - it is a **string list**

```yaml
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command:
    - "sleep"
    args:
    - "1200"
```

## environment variables
- can pass env variables to pod with **spec.containers.env** in the pod definition file
  - every variable should have a **name** and **value**
  - it is a **string list**

```yaml
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    env:
    - name: "APP_COLOR"
      value: "GREEN"
```

- can pass values from configmap via **valueFrom** with **configMapKey**
- can pass values from configmap via **valueFrom** with **secretKeyRef**

```yaml
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: webapp-config-map
          key: APP_COLOR
```

## configmap
- config maps are used to pass configuration data in the form of key value pairs in kubernetes
- create configmap then inject it to the pod
- in the yaml it **doesn't** have **spec**, insted have **data**
```bash
emre@home ~ → kubectl create configmap \
  app-config --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=test
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: test
```

- can inject configmap via **spec.containers.envFrom** with **configMapRef**

```yaml
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    envFrom:
    - configMapRef:
        name: app-config
```

## secrets
- used to store sensetive information
- stored in encoded format
- imperative way

`emre@home ~ → kubectl create secret generic <secret-name> --from-literal=<key>=<value>`

`emre@home ~ → kubectl create secret generic app-secret --from-file=app_secret.properties`

- use base64 encoded secrets
- declarative way


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: mysql
  #DB_HOST: bXlzcWw=
  DB_User: root
  DB_Password: password
```

- can inject configmap via **spec.containers.envFrom** with **secretRef**

```yaml
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    envFrom:
    - secretRef:
        name: app-secret
```

## init containers
- when a pod is first created the initcontainer is run, and the process in the initcontainer must run to a completion before the real container hosting the application starts.
- each init container is run one at a time in sequential order.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

## cluster maintenance
- default is 5 minute for pod eviction timeout
- drain nodes for os updates
  - `emre@home ~ → kubectl drain node-1`
- drained nodes becames unschedulable until uncordon
  - `emre@home ~ → kubectl uncordon node-1`
- make nodes unschedulable witouth terminating existing nodes
  - `emre@home ~ → kubectl cordon node-2`
- if there is app on node that isn't part any replicaset the node cannot be drained


## etcd
- `etcdctl` is the client for interacting with **etcd**
- before using `etcdctl` export api version the environment variable
  - `emre@home ~ → export ETCDCTL_API=3`
- for TLS enabled etcd database `--cacert=`, `--cert=`, `--endpoints=<host>:<port>`, `--key=` are **MANDATORY**
- check the cluster members with `emre@home ~ → etcdctl member list`

## backup & restore
- resource configuration
  - store in the source code repository
  - query to **kube-apiserver** with `kubectl` and save the output
    - `emre@home ~ → kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml`
- etcd cluster
  - stores information about state of the cluster itself
  - --data-dir directory can be backup
  - backup a snapshot of etcd with `etcdctl`
    - `emre@home ~ → ETCDCTL_API=3 etcdctl snapshot save snapshot.db`
    - `emre@home ~ → ETCDCTL_API=3 etcdctl snapshot status snapshot.db`
    - for restoring the snapshot
      1. stop the **kube-apiserver** service with `emre@home ~ → service kube-apiserver stop`
      2. `emre@home ~ → ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup`
      3. new data directory should be updated on **etcd.service** as well
      4. reload the **daemon** with `emre@home ~ → systemctl deamon-reload` to read the updated configuration
      5. then restart the **etcd** service with `emre@home ~ → systemctl etcd restart`
      6. lastly start the **kube-apiserver** service with `emre@home ~ → service kube-apiserver start`
  - for the all **etcdctl** commands you need to specify endpoint, cacert, cert, key files
    - `emre@home ~ → ETCDCTL_API=3 etcdctl --endpoint=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key`
- persistent volumes

## security
### secure hosts
- password based authentication disabled
- ssh key based authentication
- root access disabled
- other related infra security and hardening

### authentication
- kubertetes not manage user accounts
- can create service accounts
- all user access managed by kube-apiserver
- auth mechanisims
  1. static password files
    - from csv file with password, username, userid, optionally groupid
    - need to add `--basic-auth-file=<filename>` to kube-apiserver server configuration
    - specify user and password when running commands with curl
    - **not recommended and deprecated**
  2. static token files
    - from csv file with token, username, userid, optionally groupid
    - need to add `--token-auth-file=<filename>` to kube-apiserver server configuration
    - specify the token with `Authentication: Bearer <token>` in header
    - **not recommended and deprecated**
  3. certificates
  4. identity services

## tls in kubernetes

### kubernetes server components
- each components have their own pair certificates
- kube-apiserver
- etcd
- kubelet

### kubernetes client components
- each components have their own pair certificates
- admin (kubectl, kubeadm)
- kube-scheduler
- kube-controller-manager
- kube-proxy
- kube-apiserver to talk to etcd
- kube-apiserver to talk to kubelet

## api groups
 /metrics /healthz /version /api /apis /logs
- core and named two categories
- core group is where all core func exists
- grouped based on purpose


## authorization mechanisms
- authorization mode selected with kube-apiserver's starting args.
- `--authorization-mode=<mode>` is used for selecting the one of the mode below
- if nothins selected **AlwaysAllow** is the default
- can accept multiple with comma seperated list
- node
  - access within the cluster
- abac (attribute based access controls)
  - difficult to manage because of you need manually change every file and restart kube-apiserver
  - `{"kind": "Policy", "spec": {"user": "dev-user", "namespace": "*", "resource": "pods", "apiGroup": "*"}}`
- rbac (role based access controls)
  - insted of user or group we define role and assosiate with users
- webhook
  - manage the authorization with 3rd party tool
  - exp: open policy agent
- AlwaysAllow and AlwaysDeny two extras

### rbac
- create a role object
- for core group `apiGroups` is empty

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
```

- create the role with `kubectl create -f <filename>.yaml`
- link the user with the role using role binding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metada:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

- create the role bindings with `kubectl create -f <filename>.yaml`
- to get roles, run `kubectl get role`
- to get role bindings, run `kubectl get rolebindings`
- use `decribe` command to get details
- to check access, run `kubectl auth can-i <the command you want to check>`
- to impersonate to another user `kubectl auth can-i <command to check> --as <user to impersonate>`
- you can add `--namespace` to impersonating the user
- you can add `resourceNames` to the role object to give permission only for specific pods

### cluster roles
- cluster roles are system wide and not namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
name: cluster-administrator
rules:
- apiGroups: [""]
  resources: [“nodes"]
  verbs: ["list“, "get", “create“, “delete"]
```


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

## service accounts
- service accounts used by machines
- when service account created, a **token** generated automatically
- token is stored in **secrets**
- a default service account always mounted pods
- default service account limited to basic queries to kube-apiserver
- default service account mounted at **/var/run/secrets/kubernetes.io/serviceaccount** folder
- default account mount can be disabled via `automountServiceAccountToken: false` in definition file
- to create service account, run `kubectl create serviceaccount <account name>`
- to create token for service account, run `kubectl create token <account name>`
- to get service accounts, run `kubectl get serviceaccount`
- cannot edit running pod's service accounts
- you can edit deployment's service accounts
- to create a token associated with a secret object:

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: myserviceaccount
```

## image security
- to login private registery we create a secret with docker-registry

`kubectl create secret docker-registry regcred --docker-server= --docker-username --docker-password --docker-email`
- after creating this we specify this secret in the pod definition file

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: private-registry.io/apps/internap-app-name
  imagePullSecrets:
  - name: regrecd
```

## security context
- to run containers with specific user

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
   runAsUser: 1000
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
```

- capabilities are only supported at the container level

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      capabilities:
        add: ["MAC_ADMIN"]
```

## network
### network policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
    ports:
    - protocol: TCP
      port: 3306
```

- from outside of the cluster

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
```

- to outside of the cluster

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 80
```

## certificates api
- creating csr object

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: Jane
spec:
  expirationSeconds: 600
  usages:
  - digital signature
  - key encipherment
  - server auth
  request:
  - <base64 csr>
```

- `kubectl get csr` to get requests
- `kubectl certificate approve <name>` to sign the request
- the signed certificate will be in csr object
- `kubectl get csr -o yaml` to see signed base64 certificate
- controller manager is doign the signing

## kube config
- the default config file is $HOME/.kube/config
- clusters, users, context(which user used to access to which cluster)
- uses the existing users
- `kubectl config view` to view config file in use
- `kubectl config use-context <context>` to change context
- use `certificate-authority-data` instead `certificate-authority` for directly adding base64 data to config file
- use `KUBECONFIG` environment variable to export custom config file
- example:

```yaml
apiVersion: v1
kind: Config
current-context: my-kube-admin@my-kube-playground
clusters:
- name:  my-kube-playground
  cluster:
    certificate-authority: ca.crt
    server: https://my-kube-playground:6443
context:
- name: my-kube-admin@my-kube-playground
  context:
    cluster: my-kube-playground
    user: my-kube-admin
    namespace: finance
users:
- name: my-kube-admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

## volumes
- directory to directory mapping
- to create and mount a volume on host to pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - image: alpine
    name: alpine
    volumeMounts:
    - mountPath: /opt
      name: data-volume
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```

- do not use the host without a nfs. data will not persist bettween nodes.

## persistent volumes
- to create persitent volumes

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes: # defines how a volume should be mounted on host
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath: # defines volume type do not use this on production replace this with storage solutions
    path: /tmp/data
```

- persistent volumes one-to-one with persistent volume claims
- unclaimed storage will not be used by other claims
- to use persistent volume, create persistent volume claim

```yaml
apiVersion: v1
kind: PersistenctVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 505Mi
```

- to add pvc to any pod, deployment or replica sets add below to under volumes

```yaml
spec:
  volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myclaim
```

- when pvc deleted, volume will be not deleted and will not claimed by other pvc by default
- if you want to delete pv with pvc, use `persistentVolumeReclaimPolicy: Delete` when creating pv
- if you want to recycle pv for new pvc, use `persistentVolumeReclaimPolicy: Recycle` when creating pv

## storage class
- with storage class you provision dynamicly with cloud vendors

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
```

- with storage class we don't need to create persistent volume anymore.
- to use storage class, add it to pvc definition

```yaml
spec:
  storageClassName: google-storage
```

## network
- networks managed by plugins
- network plugins installed on `/opt/cni/bin`
- which plugin and how to use it configurations stored in `/etc/cni/net.d`

### cni plugin responsibilities
- must support arguments add/del/check
- must support parameters conainer id, network ns, etc...
- must manage ip address assignment to pods
- must return results in a specific format

### service network
- services is hosted across the cluster
- `clusterip` is when you create a service accessible from all pods on the cluster
- `nodeport` is when you need to expose application on node port to access from external

### dns
- whenever a service created, dns service creates a record for the service and map the service name to the ip address
- all pod and services for a namespace grouped together with a subdomain in the name of the namespace
- all the services grouped together in another subdomain called svc
- all the services and pods are grouped together, into a route domain for the cluster, which is set to cluster.local by default.
- records for pod are not created by default.

### ingress
- for ingress to work must deploy an ingress controller
- ingress resource comes under the namespace scoped
- multi-tenacy is not supported (different paths manages by different teams)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

### gateway api
- like ingress must deploy a controller
- infra team configure the `gateway class`
  - defines what is the underlying network would be (nginx, traefik, etc.)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
spec:
  controllerName: example.com/gateway-controller
```

- cluster team configure the `gateway`
  - which are instances of the `gateway class`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

- app team configure `http route`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metada:
  name: example-httproute
spec:
  parentRefs:
  - name: example-gateway
    namespace: example-namespace
  hostname:
  - "www.example.com"
  rules:
  - matches:
    - path:
        type: Prefix
        value: /login
    backendRefs:
    - name: example-svc
      port: 8080
```

- unlike `ingress` you can use other routes like `tcp route`, `grpc route`
