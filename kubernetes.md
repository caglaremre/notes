# kubernetes

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
- can pass args to pod with __spec.containers.args__ in the pod definition file
  - it is a __string list__
- can override entrypoint with __spec.container.command__ in the pod definition file
  - it is a __string list__
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
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: test
```
- can inject configmap via __spec.containers.envFrom__ with __configMapRef__
```yaml
spec:
  containers:
   - name: ubuntu
     image: ubuntu
     envFrom:
       - configMapRef:
           name: app-config
```