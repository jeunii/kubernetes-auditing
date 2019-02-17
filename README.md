# kubernetes-auditing
This document explains hows to setup kubernetes auditing in a minikube cluster.

# Pre Requisities
Spin up minikube with the latest k8s version.

```bash
minikube start --kubernetes-version=v1.13.2
```

Verify that the clutser is up and running and that the api-server is healthy.

```bash
kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
minikube   Ready    master   2d22h   v1.13.2

kubectl get po kube-apiserver-minikube -n kube-system
NAME                                        READY   STATUS      RESTARTS   AGE
kube-apiserver-minikube                     1/1     Running     0          23h
```

# Enable auditing
There are 2 essetial flags that needed to be added to the API Server manifests.
- `--audit-policy-file`
- `--audit-log-path`

Access the minikube vm first and switch over to root user.

```bash
minikube ssh
su - 
```

The manifest files can be found under `/etc/kubernetes/manifests/`

```bash
cd /etc/kubernetes/manifests/
ls -ltrh
total 24K
-rw-r----- 1 root root 1.4K Feb 14 16:07 addon-manager.yaml
-rw------- 1 root root 2.3K Feb 14 16:08 kube-controller-manager.yaml
-rw------- 1 root root 1.1K Feb 14 16:08 kube-scheduler.yaml
-rw------- 1 root root 2.0K Feb 14 16:08 etcd.yaml
-rw------- 1 root root 3.2K Feb 16 15:23 kube-apiserver.yaml
```

Use `vi` to edit the `kube-apiserver.yaml` file and add the flag `--audit-policy-file`. We need to give the path to the audit policy file which we will described in the next section.

By default when you specify the flag `--audit-policy-file`, the API Server will restart on its own. But whenever you edit the policy file, you will need to restart the docker container manually by running `docker restart <API-SERVER-CONTAINER-ID>`

### Log to standard out
If you want the audit logs to be written to standard out, use the following config
```bash
- --audit-log-path=-
```
Now you can view logs by just executing `kubectl logs <API-SERVER-POD>`
### Log to file
If you want the audit logs to be written to a file, use the following config
```bash
- --audit-log-path=/var/lib/minikube/logs/audit.log
```

## Make the audit log avaiable to minikube vm
**By default the log file will be created inside the container.** So to view it might be a hassle since you'll need to exec into the container and then view the logs by navigating to the correct directory. Instead change the manifest of the API server and mount the host `/tmp` to the container directory where the log is being generated.

e.g. If the flag is set to `--audit-log-path=/var/lib/minikube/logs/audit.log`, then your manifests Volume and VolumeMount sections should look like
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
Now when you execute `minikube ssh`, the audit log will be avaiable at `/tmp/audit.log`

# Audit Policy File
As per documention from the official k8s website 

> Audit policy defines rules about what events should be recorded and what data they should include. The audit policy object structure is defined in the audit.k8s.io API group. When an event is processed, it’s compared against the list of rules in order. The first matching rule sets the “audit level” of the event. The known audit levels are:
> - `None` - don’t log events that match this rule.
> - `Metadata` - log request metadata (requesting user, timestamp, resource, verb, etc.) but not request or response body.
> - `Request` - log event metadata and request body but not response body. This does not apply for non-resource requests.
> - `RequestResponse` - log event metadata, request and response bodies. This does not apply for non-resource requests.

## Log only Metadata for pods/log
The file content should look like
```bash
apiVersion: audit.k8s.io/v1 # This is required.
kind: Policy
rules:
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log"]
```
Running the below command 
```bash
kubectl logs coredns-869f847d58-j6r76 -n kube-system
```
will generate the audit logs

```javascript
{  
   "kind":"Event",
   "apiVersion":"audit.k8s.io/v1",
   "level":"Metadata",
   "auditID":"4a492c40-7192-4265-b983-f721e8979ad5",
   "stage":"ResponseStarted",
   "requestURI":"/api/v1/namespaces/kube-system/pods/coredns-869f847d58-j6r76/log",
   "verb":"get",
   "user":{  
      "username":"minikube-user",
      "groups":[  
         "system:masters",
         "system:authenticated"
      ]
   },
   "sourceIPs":[  
      "192.168.99.1"
   ],
   "userAgent":"kubectl/v1.13.3 (darwin/amd64) kubernetes/721bfa7",
   "objectRef":{  
      "resource":"pods",
      "namespace":"kube-system",
      "name":"coredns-869f847d58-j6r76",
      "apiVersion":"v1",
      "subresource":"log"
   },
   "responseStatus":{  
      "metadata":{  

      },
      "code":200
   },
   "requestReceivedTimestamp":"2019-02-17T14:57:35.748832Z",
   "stageTimestamp":"2019-02-17T14:57:35.757340Z",
   "annotations":{  
      "authorization.k8s.io/decision":"allow",
      "authorization.k8s.io/reason":""
   }
}
```
and
```javascript
{  
   "kind":"Event",
   "apiVersion":"audit.k8s.io/v1",
   "level":"Metadata",
   "auditID":"4a492c40-7192-4265-b983-f721e8979ad5",
   "stage":"ResponseComplete",
   "requestURI":"/api/v1/namespaces/kube-system/pods/coredns-869f847d58-j6r76/log",
   "verb":"get",
   "user":{  
      "username":"minikube-user",
      "groups":[  
         "system:masters",
         "system:authenticated"
      ]
   },
   "sourceIPs":[  
      "192.168.99.1"
   ],
   "userAgent":"kubectl/v1.13.3 (darwin/amd64) kubernetes/721bfa7",
   "objectRef":{  
      "resource":"pods",
      "namespace":"kube-system",
      "name":"coredns-869f847d58-j6r76",
      "apiVersion":"v1",
      "subresource":"log"
   },
   "responseStatus":{  
      "metadata":{  

      },
      "code":200
   },
   "requestReceivedTimestamp":"2019-02-17T14:57:35.748832Z",
   "stageTimestamp":"2019-02-17T14:57:35.761424Z",
   "annotations":{  
      "authorization.k8s.io/decision":"allow",
      "authorization.k8s.io/reason":""
   }
}
```

## Log only Requests for configMaps
```bash
apiVersion: audit.k8s.io/v1
kind: Policy
  - level: Request
    resources:
    - groups: ""
      resources: ["configmaps"]
```
Running the below command 
```bash
k get cm coredns -n kube-system
```
will generate the audit log
```javascript
{  
   "kind":"Event",
   "apiVersion":"audit.k8s.io/v1",
   "level":"Request",
   "auditID":"a7b4eb9a-d8d7-4679-842f-d20099b6b4f0",
   "stage":"ResponseComplete",
   "requestURI":"/api/v1/namespaces/kube-system/configmaps/coredns",
   "verb":"get",
   "user":{  
      "username":"minikube-user",
      "groups":[  
         "system:masters",
         "system:authenticated"
      ]
   },
   "sourceIPs":[  
      "192.168.99.1"
   ],
   "userAgent":"kubectl/v1.13.3 (darwin/amd64) kubernetes/721bfa7",
   "objectRef":{  
      "resource":"configmaps",
      "namespace":"kube-system",
      "name":"coredns",
      "apiVersion":"v1"
   },
   "responseStatus":{  
      "metadata":{  

      },
      "code":200
   },
   "requestReceivedTimestamp":"2019-02-17T15:09:25.645226Z",
   "stageTimestamp":"2019-02-17T15:09:25.646435Z",
   "annotations":{  
      "authorization.k8s.io/decision":"allow",
      "authorization.k8s.io/reason":""
   }
}
```
## Log Requests/Response for configMaps
```bash
  - level: RequestResponse
    users: ["minikube-user"]
    resources:
    - groups: ""
      resources: ["configmaps"]
```
Running the below command
```bash
kubectl get cm coredns -n kube-system -o yaml
```
will generate an audit log
```javascript
{  
   "kind":"Event",
   "apiVersion":"audit.k8s.io/v1",
   "level":"RequestResponse",
   "auditID":"a135fa09-26a3-4341-9290-1313962febd0",
   "stage":"ResponseComplete",
   "requestURI":"/api/v1/namespaces/kube-system/configmaps/coredns",
   "verb":"get",
   "user":{  
      "username":"minikube-user",
      "groups":[  
         "system:masters",
         "system:authenticated"
      ]
   },
   "sourceIPs":[  
      "192.168.99.1"
   ],
   "userAgent":"kubectl/v1.13.3 (darwin/amd64) kubernetes/721bfa7",
   "objectRef":{  
      "resource":"configmaps",
      "namespace":"kube-system",
      "name":"coredns",
      "apiVersion":"v1"
   },
   "responseStatus":{  
      "metadata":{  

      },
      "code":200
   },
   "responseObject":{  
      "kind":"ConfigMap",
      "apiVersion":"v1",
      "metadata":{  
         "name":"coredns",
         "namespace":"kube-system",
         "selfLink":"/api/v1/namespaces/kube-system/configmaps/coredns",
         "uid":"e3fd0492-3072-11e9-a3b1-080027a55f80",
         "resourceVersion":"193",
         "creationTimestamp":"2019-02-14T16:09:22Z"
      },
      "data":{  
         "Corefile":".:53 {\n    errors\n    health\n    kubernetes cluster.local in-addr.arpa ip6.arpa {\n       pods insecure\n       upstream\n       fallthrough in-addr.arpa ip6.arpa\n    }\n    prometheus :9153\n    proxy . /etc/resolv.conf\n    cache 30\n    loop\n    reload\n    loadbalance\n}\n"
      }
   },
   "requestReceivedTimestamp":"2019-02-17T15:14:59.711425Z",
   "stageTimestamp":"2019-02-17T15:14:59.712478Z",
   "annotations":{  
      "authorization.k8s.io/decision":"allow",
      "authorization.k8s.io/reason":""
   }
}
```
As you can see, `.responseObject.data` contains the Response metadata.