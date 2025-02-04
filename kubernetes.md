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
- can pass args to pod with __spec.containers.args__ in the pod definition file
  - it is a __string list__
- can override entrypoint with __spec.container.command__ in the pod definition file
  - it is a __string list__
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
- can pass env variables to pod with __spec.containers.env__ in the pod definition file
  - every variable should have a __name__ and __value__
  - it is a __string list__
- can pass values from configmap via __valueFrom__ with __configMapKey__
- can pass values from configmap via __valueFrom__ with __secretKeyRef__
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
- in the yaml it __doesn't__ have __spec__, insted have __data__
```bash
kubectl create configmap \
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

- can inject configmap via __spec.containers.envFrom__ with __configMapRef__
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

`kubectl create secret generic <secret-name> --from-literal=<key>=<value>`

`kubectl create secret generic app-secret --from-file=app_secret.properties`

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

- can inject configmap via __spec.containers.envFrom__ with __secretRef__
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