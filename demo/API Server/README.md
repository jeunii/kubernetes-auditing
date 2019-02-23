# API Server configs
Following are the additional flags
```bash
    - --audit-policy-file=/etc/ssl/certs/audit-policy.yaml
    - --audit-webhook-config-file=/etc/ssl/certs/webhook-config.yaml
    - --audit-log-path=/var/lib/minikube/logs/audit.log
    - --audit-webhook-batch-max-wait=5s
```
Following are the volume settings
```bash
    volumeMounts:
    - mountPath: /var/lib/minikube/logs
      name: logs
      readOnly: false
  volumes:
  - hostPath:
      path: /tmp
      type: Directory
    name: logs
```