# kubernetes

## metada
- custom data for the definition file
<table border=1>
<tr>
<td>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-color
  labels:
    name: webapp-color
```
</td>
</tr>
</table>

## commands and arguments
- can pass args to pod with **spec.containers.args** in the pod definition file
  - it is a **string list**
- can override entrypoint with **spec.container.command** in the pod definition file
  - it is a **string list**
<table border=1>
<tr>
<td>

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
</td>
</tr>
</table>

## environment variables
- can pass env variables to pod with **spec.containers.env** in the pod definition file
  - every variable should have a **name** and **value**
  - it is a **string list**
- can pass values from configmap via **valueFrom** with **configMapKey**
- can pass values from configmap via **valueFrom** with **secretKeyRef**
<table border=1>
<tr>
<td>


```yaml
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      env:
       - name: "APP_COLOR"
         value: "GREEN"
```
</td>
<td>

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
</td>
</tr>
</table>

## configmap
- config maps are used to pass configuration data in the form of key value pairs in kubernetes
- create configmap then inject it to the pod
- in the yaml it **doesn't** have **spec**, insted have **data**
```bash
emre@home ~ → kubectl create configmap \
  app-config --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=test
```
<table border=1>
<tr>
<td>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: test
```
</td>
</tr>
</table>

- can inject configmap via **spec.containers.envFrom** with **configMapRef**
<table border=1>
<tr>
<td>

```yaml
spec:
  containers:
   - name: ubuntu
     image: ubuntu
     envFrom:
       - configMapRef:
           name: app-config
```
</td>
</tr>
</table>

## secrets
- used to store sensetive information
- stored in encoded format
- imperative way

`emre@home ~ → kubectl create secret generic <secret-name> --from-literal=<key>=<value>`

`emre@home ~ → kubectl create secret generic app-secret --from-file=app_secret.properties`

- use base64 encoded secrets
- declarative way

<table border=1>
<tr>
<td>

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
</td>
</tr>
</table>

- can inject configmap via **spec.containers.envFrom** with **secretRef**
<table border=1>
<tr>
<td>

```yaml
spec:
  containers:
   - name: ubuntu
     image: ubuntu
     envFrom:
       - secretRef:
           name: app-secret
```
</td>
</tr>
</table>

## init containers
- when a pod is first created the initcontainer is run, and the process in the initcontainer must run to a completion before the real container hosting the application starts.
- each init container is run one at a time in sequential order.

<table border=1>
<tr>
<td>

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
</td>
</tr>
</table>

## cluster maintenance
- default is 5 minute for pod eviction timeout
- drain nodes for os updates
  - `emre@home ~ → kubectl drain node-1`
- drained nodes becames unschedulable until uncordon
  - `emre@home ~ → kubectl uncordon node-1`
- make nodes unschedulable witouth terminating existing nodes
  - `emre@home ~ → kubectl cordon node-2`
- if there is app on node that isn't part any replicaset the node cannot be drained

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
