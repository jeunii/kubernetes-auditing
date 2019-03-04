# API Server configs

### Setup config directory

```bash
    volumeMounts:
    - mountPath: /var/lib/minikube/config/
      name: audit-config
  volumes:
  - hostPath:
     path: /var/lib/minikube/config/
     type: DirectoryOrCreate
    name: audit-config
```

### Enable audit logging to STDOUT.

Create file `/var/lib/minikube/config/audit-policy.yaml`.

```bash
    - --audit-policy-file=/var/lib/minikube/config/audit-policy.yaml
    - --audit-log-path=-
```

### Enable audit logging to file.

```bash
    volumeMounts:
    - mountPath: /var/lib/minikube/logs
      name: audit-logs
      readOnly: false
  volumes:
  - hostPath:
      path: /tmp
      type: Directory
    name: audit-logs
```

```bash
    - --audit-log-path=/var/lib/minikube/logs/audit.log
```

### Lets omit a stage

```bash
omitStages:
  - "RequestReceived"
```
Restart docker image !

### Falco !!!
Create namespace.
```bash
kubectl create ns audit-demo
```

Create config map.
```bash
kubectl create configmap falco-config --from-file=configs -n audit-demo
```
Create daemonset.
```bash
kubectl create -f . -n audit-demo
```

### Configure the Webhook 
```bash
    - --audit-webhook-config-file=/var/lib/minikube/config/webhook-config.yaml
    - --audit-webhook-batch-max-wait=5s
```