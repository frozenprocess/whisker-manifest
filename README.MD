# How to enable
# Requirements
- Docker
- K3d
- Internet

```bash
k3d cluster create \
  my-calico-manifest-cluster \
  -s 1 -a 2 \
  --k3s-arg '--flannel-backend=none@server:*' \
  --k3s-arg '--disable-network-policy@server:*' \
  --k3s-arg '--disable=traefik@server:*' \
  --k3s-arg '--cluster-cidr=192.168.0.0/16@server:*'
```

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/calico-typha.yaml
```

🔐 CA Key and Certificate
```bash
openssl genrsa -out certs/ca.key 2048

openssl req -new -x509 -key certs/ca.key \
    -subj '/CN=frozen-tutorials/O=We love Calico/C=US' \
    -days 1024 -out certs/ca.crt

openssl req \
  -newkey rsa:2048 -nodes -sha256 -keyout certs/typha-client.key \
  -subj '/CN=typha-client/O=We love Calico/C=US' \
  -out certs/typha-client.csr
```

📥 Typha Client Certificate

```bash
openssl x509 -req \
    -in certs/typha-client.csr \
    -CA certs/ca.crt -CAkey certs/ca.key \
    -out certs/typha-client.crt \
    -days 1024

openssl req \
  -newkey rsa:2048 -nodes -sha256 -keyout certs/typha-server.key \
  -subj '/CN=typha-server/O=We love Calico/C=US' \
  -out certs/typha-server.csr
```

📥 Typha Server Certificate

```bash
openssl x509 -req \
    -in certs/typha-server.csr \
    -CA certs/ca.crt -CAkey certs/ca.key \
    -out certs/typha-server.crt \
    -days 1024

openssl req \
  -newkey rsa:2048 -nodes -sha256 -keyout certs/goldmane.key \
  -subj '/CN=goldmane/O=We love Calico/C=US' \
  -addext "subjectAltName=DNS:goldmane,DNS:goldmane.kube-system,DNS:goldmane.kube-system.svc,DNS:goldmane.kube-system.svc.cluster.local" \
  -out certs/goldmane.csr
```

📥 Goldmane Certificate with SAN

```bash
openssl x509 -req \
    -in certs/goldmane.csr \
    -CA certs/ca.crt -CAkey certs/ca.key \
    -out certs/goldmane.crt \
    -days 1024

openssl req \
  -newkey rsa:2048 -nodes -sha256 -keyout certs/whisker-backend.key \
  -subj '/CN=whisker-backend/O=We love Calico/C=US' \
  -addext "subjectAltName=DNS:whisker-backend,DNS:whisker-backend.kube-system,DNS:whisker-backend.kube-system.svc,DNS:whisker-backend.kube-system.svc.cluster.local" \
  -out certs/whisker-backend.csr
```


📥 Whisker Backend Certificate with SAN

```bash
openssl x509 -req \
    -in certs/whisker-backend.csr \
    -CA certs/ca.crt -CAkey certs/ca.key \
    -out certs/whisker-backend.crt \
    -days 1024 \
    -sha256 -extensions v3_req -extfile certs/req-whisker.conf

openssl x509 -req \
    -in certs/goldmane.csr \
    -CA certs/ca.crt -CAkey certs/ca.key \
    -out certs/goldmane.crt \
    -days 1024 \
    -sha256 -extensions v3_req -extfile certs/req-goldmane.conf
```

```
kubectl create configmap goldmane-ca-bundle \
  --namespace=kube-system \
  --from-file=ca-bundle.crt=./certs/ca.crt \
  --from-file=tigera-ca-bundle.crt=./certs/ca.crt \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret tls goldmane-key-pair \
  --cert=certs/goldmane.crt \
  --key=certs/goldmane.key \
  --namespace=kube-system \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret tls typha-client-key-pair \
  --cert=certs/typha-client.crt \
  --key=certs/typha-client.key \
  --namespace=kube-system \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret tls typha-server-key-pair \
  --cert=certs/typha-server.crt \
  --key=certs/typha-server.key \
  --namespace=kube-system \
  --dry-run=server -o yaml | kubectl apply -f -

kubectl create secret tls whisker-backend-key-pair \
  --cert=certs/whisker-backend.crt \
  --key=certs/whisker-backend.key \
  --namespace=kube-system \
  --dry-run=server -o yaml | kubectl apply -f -
```

```bash
        - name: FELIX_FLOWLOGSGOLDMANESERVER
          value: goldmane.kube-system.svc:7443
        - name: FELIX_FLOWLOGSFLUSHINTERVAL
          value: "15"
```

```bash
kubectl patch deployment calico-typha -n kube-system --type=strategic -p='
{
  "spec": {
    "template": {
      "spec": {
        "volumes": [
          {
            "name": "config",
            "configMap": {
              "name": "goldmane",
              "defaultMode": 420
            }
          },
          {
            "name": "goldmane-ca-bundle",
            "configMap": {
              "name": "goldmane-ca-bundle",
              "defaultMode": 420
            }
          },
          {
            "name": "typha-server-key-pair",
            "secret": {
              "secretName": "typha-server-key-pair",
              "defaultMode": 420
            }
          }
        ],
        "containers": [
          {
            "name": "calico-typha",
            "env": [
              {
                "name": "TYPHA_CAFILE",
                "value": "/etc/pki/tls/certs/tigera-ca-bundle.crt"
              },
              {
                "name": "TYPHA_SERVERCERTFILE",
                "value": "/typha-server-key-pair/tls.crt"
              },
              {
                "name": "TYPHA_SERVERKEYFILE",
                "value": "/typha-server-key-pair/tls.key"
              },
              {
                "name": "TYPHA_CLIENTCN",
                "value": "typha-client"
              }
            ],
            "volumeMounts": [
              {
                "name": "goldmane-ca-bundle",
                "mountPath": "/etc/pki/tls/certs",
                "readOnly": true
              },
              {
                "name": "typha-server-key-pair",
                "mountPath": "/typha-server-key-pair",
                "readOnly": true
              }
            ]
          }
        ]
      }
    }
  }
}'
```


```bash
kubectl patch ds calico-node -n kube-system --type=strategic -p='
{
  "spec": {
    "template": {
      "spec": {
        "volumes": [
          {
            "name": "config",
            "configMap": {
              "name": "goldmane",
              "defaultMode": 420
            }
          },
          {
            "name": "goldmane-ca-bundle",
            "configMap": {
              "name": "goldmane-ca-bundle",
              "defaultMode": 420
            }
          },
          {
            "name": "typha-client-key-pair",
            "secret": {
              "secretName": "typha-client-key-pair",
              "defaultMode": 420
            }
          }
        ],
        "containers": [
          {
            "name": "calico-node",
            "env": [
              {
                "name": "FELIX_TYPHACAFILE",
                "value": "/etc/pki/tls/certs/tigera-ca-bundle.crt"
              },
              {
                "name": "FELIX_TYPHACERTFILE",
                "value": "/typha-client-key-pair/tls.crt"
              },
              {
                "name": "FELIX_TYPHAKEYFILE",
                "value": "/typha-client-key-pair/tls.key"
              },
              {
                "name": "FELIX_TYPHAK8SNAMESPACE",
                "value": "kube-system"
              },
              {
                "name": "FELIX_TYPHACN",
                "value": "typha-server"
              }
            ],
            "volumeMounts": [
              {
                "name": "goldmane-ca-bundle",
                "mountPath": "/etc/pki/tls/certs",
                "readOnly": true
              },
              {
                "name": "typha-client-key-pair",
                "mountPath": "/typha-client-key-pair",
                "readOnly": true
              }
            ]
          }
        ]
      }
    }
  }
}'
```

```bash
kubectl create -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: goldmane
  namespace: kube-system
EOF
```

```
kubectl create -f -<<EOF
apiVersion: v1
kind: Service
metadata:
  name: goldmane
  namespace: kube-system
spec:
  ports:
  - port: 7443
    protocol: TCP
    targetPort: 7443
  selector:
    k8s-app: goldmane
  sessionAffinity: None
  type: ClusterIP
EOF
```

```
https://github.com/projectcalico/calico/blob/95c9579475bfdefe8850feb3324016cfe3553034/felix/collector/dpstatshelper.go#L69C1-L71C29
```

```bash
kubectl create -f -<<EOF
apiVersion: v1
data:
  config.json: '{"emitFlows":false}'
kind: ConfigMap
metadata:
  name: goldmane
  namespace: kube-system
EOF
```

```bash
kubectl create -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: goldmane
    k8s-app: goldmane
  name: goldmane
  namespace: kube-system
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: goldmane
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/name: goldmane
        k8s-app: goldmane
      name: goldmane
    spec:
      containers:
      - env:
        - name: LOG_LEVEL
          value: INFO
        - name: PORT
          value: "7443"
        - name: SERVER_CERT_PATH
          value: /goldmane-key-pair/tls.crt
        - name: SERVER_KEY_PATH
          value: /goldmane-key-pair/tls.key
        - name: CA_CERT_PATH
          value: /etc/pki/tls/certs/tigera-ca-bundle.crt
        - name: PUSH_URL
          value: https://guardian.kube-system.svc.cluster.local:443/api/v1/flows/bulk
        - name: FILE_CONFIG_PATH
          value: /config/config.json
        - name: HEALTH_ENABLED
          value: "true"
        image: docker.io/calico/goldmane:v3.30.2
        imagePullPolicy: IfNotPresent
        livenessProbe:
          exec:
            command:
            - /health
            - -live
          failureThreshold: 3
          periodSeconds: 60
          successThreshold: 1
          timeoutSeconds: 5
        name: goldmane
        readinessProbe:
          exec:
            command:
            - /health
            - -ready
          failureThreshold: 3
          periodSeconds: 30
          successThreshold: 1
          timeoutSeconds: 5
        resources: {}
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          privileged: false
          runAsGroup: 10001
          runAsNonRoot: true
          runAsUser: 10001
          seccompProfile:
            type: RuntimeDefault
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /config
          name: config
          readOnly: true
        - mountPath: /etc/pki/tls/certs
          name: goldmane-ca-bundle
          readOnly: true
        - mountPath: /goldmane-key-pair
          name: goldmane-key-pair
          readOnly: true
      dnsPolicy: ClusterFirst
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: goldmane
      serviceAccountName: goldmane
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
      - key: CriticalAddonsOnly
        operator: Exists
      volumes:
      - configMap:
          defaultMode: 420
          name: goldmane
        name: config
      - configMap:
          defaultMode: 420
          name: goldmane-ca-bundle
        name: goldmane-ca-bundle
      - name: goldmane-key-pair
        secret:
          defaultMode: 420
          secretName: goldmane-key-pair
EOF
```

```bash
kubectl patch ds calico-node -n kube-system --type=strategic -p='
{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "calico-node",
            "env": [
              {
                "name": "FELIX_FLOWLOGSGOLDMANESERVER",
                "value": "goldmane.kube-system.svc:7443"
              },
              {
                "name": "FELIX_FLOWLOGSFLUSHINTERVAL",
                "value": "15"
              }
            ]
          }
        ]
      }
    }
  }
}'
```


```bash
kubectl create -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: whisker
    k8s-app: whisker
  name: whisker
  namespace: kube-system
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: whisker
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/name: whisker
        k8s-app: whisker
      name: whisker
    spec:
      containers:
      - env:
        - name: LOG_LEVEL
          value: INFO
        - name: CALICO_VERSION
          value: v3.30.2
        - name: NOTIFICATIONS
          value: Enabled
        image: docker.io/calico/whisker:v3.30.2
        imagePullPolicy: IfNotPresent
        name: whisker
        resources: {}
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          privileged: false
          runAsGroup: 10001
          runAsNonRoot: true
          runAsUser: 10001
          seccompProfile:
            type: RuntimeDefault
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      - env:
        - name: LOG_LEVEL
          value: INFO
        - name: PORT
          value: "3002"
        - name: GOLDMANE_HOST
          value: goldmane.kube-system.svc.cluster.local:7443
        - name: TLS_CERT_PATH
          value: /whisker-backend-key-pair/tls.crt
        - name: TLS_KEY_PATH
          value: /whisker-backend-key-pair/tls.key
        image: docker.io/calico/whisker-backend:v3.30.2
        imagePullPolicy: IfNotPresent
        name: whisker-backend
        resources: {}
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          privileged: false
          runAsGroup: 10001
          runAsNonRoot: true
          runAsUser: 10001
          seccompProfile:
            type: RuntimeDefault
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /whisker-backend-key-pair
          name: whisker-backend-key-pair
          readOnly: true
        - mountPath: /etc/pki/tls/certs
          name: goldmane-ca-bundle
          readOnly: true
      dnsPolicy: ClusterFirst
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: whisker
      serviceAccountName: whisker
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
      - key: CriticalAddonsOnly
        operator: Exists
      volumes:
      - name: whisker-backend-key-pair
        secret:
          defaultMode: 420
          secretName: whisker-backend-key-pair
      - configMap:
          defaultMode: 420
          name: goldmane-ca-bundle
        name: goldmane-ca-bundle
EOF
```

```bash
kubectl create -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: whisker
  namespace: kube-system
EOF
```

```bash
kubectl create -f -<<EOF
apiVersion: v1
kind: Service
metadata:
  name: whisker
  namespace: kube-system
spec:
  ports:
  - port: 8081
    protocol: TCP
    targetPort: 8081
  selector:
    k8s-app: whisker
  sessionAffinity: None
  type: LoadBalancer
EOF
```

```bash
kubectl patch ds calico-node -n kube-system --type=strategic -p="\
{
  \"spec\": {
    \"template\": {
      \"spec\": {
        \"dnsPolicy\": \"ClusterFirstWithHostNet\"
      }
    }
  }
}"
```
