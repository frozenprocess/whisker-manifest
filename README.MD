# How to enable Calico Whisker in a Manifest based install

The whole purpose of this tutorial is to show you the behind the scene magic that happens when you are using Tigera-operator to install Calico.

Keep in mind that everything that you read here can be done using the `tigera-operator` and two resources. Basically, in an operator installation method (our recommended way) you just create these two resources and tigera-operator takes care of everything else.

```bash
apiVersion: operator.tigera.io/v1
kind: Goldmane
metadata:
  name: default
---
apiVersion: operator.tigera.io/v1
kind: Whisker
metadata:
  name: default
```

# Requirements

While this tutorial can be read like your favorite cat syfi, if you wish to replicate the same test in your environment there are a bunch of requirements that you need to install/have.
- Docker
- K3d
- OpenSSL (To issue and sign certificates)
- Internet

## Setup your cluster

Ok, just like the matrix you have chosen the red pill and now I'm going to show you how deep the rabbit hole goes. For this tutorial we are going to use `k3d`, and `docker` (hence being a requirement). This will allow us to spin up a `k3s` cluster super fast.

```bash
k3d cluster create \
  my-calico-manifest-cluster \
  -s 1 -a 2 \
  --k3s-arg '--flannel-backend=none@server:*' \
  --k3s-arg '--disable-network-policy@server:*' \
  --k3s-arg '--disable=traefik@server:*' \
  --k3s-arg '--cluster-cidr=192.168.0.0/16@server:*'
```

## Install Calico and Typha in your cluster

Now that we have the cluster ready it is time to install Calico, but this time we are going to install Calico and Typha using a manifest instead of the operator.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/calico-typha.yaml
```

**Note:** Currently, you need to have at least one instance of Typha running in order to use Goldmane and Whisker.

You can use the following command to verify if `calico-node` pods are running.
```bash
kubectl rollout status -n kube-system ds/calico-node
```
Keep in mind that a manifest install requires you (the administrator) to configure every part of Calico.
While this approach is highly flexible it is not with out pain of things like missing YAML indentations or adding a value which was deprecated or is not supported in the current version of Calico.

This is why we recommend you to use `Tigera-operator`, the operator is a dedicated operator (its to nice I'm using it twice!) that is tuned to configure and maintain your Calico installation so you can spend your time on the things that matters the most! (ehm ehm, vacations)

## Generating certificates

By default, the manifest install doesn't secure Calico components, and it is your (the administrator) responsibility to generate certificate authority, certificates and sign them and rotate them on the desired period. Keep in mind that these are all automated in an operator based install.

### 🔐 Certificate Authority (CA)

To secure calico components we need a CA to issue and sign our certificates. While this can be a valid  certificate authority that issues certificates to the masses, we can also create our own CA to sign and issue these resources.


Use the following command to establish a CA and generate its keys:
```bash
openssl genrsa -out certs/ca.key 2048

openssl req -new -x509 -key certs/ca.key \
    -subj '/CN=tigera-ca-bundle/O=We love Calico/C=US' \
    -days 1024 -out certs/ca.crt
```

### 📥 Generating a Certificate for calico-typha (Typha Server)

Calico typha is a middle layer, key value caching system designed to sit between `calico-node` requests and your `kube api-server`. This allows Typha to query the cluster api-server once and provide `calico-node` with answers to all its queries. if any changes occurs within your Kubenretes cluster then Typha will initiate another query to update its cache.

**Note:** Remember the `CN=` name here and in the next step, these two are used in the tunning typha and calico-node section as an additional verification check.

Use the following command to generate a certificate request for Typha:
```bash
openssl req \
  -newkey rsa:2048 -nodes -sha256 -keyout certs/typha-server.key \
  -subj '/CN=typha-server/O=We love Calico/C=US' \
  -out certs/typha-server.csr
```

Use the following command to sign and generate a certificate for Typha:
```bash
openssl x509 -req \
    -in certs/typha-server.csr \
    -CA certs/ca.crt -CAkey certs/ca.key \
    -out certs/typha-server.crt \
    -days 1024
```

### 📥 Generating a Certificate for Calico-Node (Typha Client)

In order to generate a certificate for the calico-node pod first you need to create a certificate request, this is due the fact that your CA needs to issue and sign all these certificates.

Use the following command to generate a ceritificat request:
```bash
openssl req \
  -newkey rsa:2048 -nodes -sha256 -keyout certs/typha-client.key \
  -subj '/CN=typha-client/O=We love Calico/C=US' \
  -out certs/typha-client.csr
```

Use the following command to sign and generate the client certificates:
```bash
openssl x509 -req \
    -in certs/typha-client.csr \
    -CA certs/ca.crt -CAkey certs/ca.key \
    -out certs/typha-client.crt \
    -days 1024
```

### 📥 Generating Certificates for Goldmane and Whisker

Goldmane and Whisker certificates are a little bit different. Since these two components hold sensitive (Network Flow Logs) information we are going to assign roles to their certificates. On top of that since these certificates are going to be used to emmit flow logs from other workloads (calico-node in this example) we need to assign Subject Alternative Name (SAN) to these certificates.

Use the following command to generate a certificate `request` for Goldmane:
```bash
openssl req \
  -newkey rsa:2048 -nodes -sha256 -keyout certs/goldmane.key \
  -subj '/CN=goldmane/O=We love Calico/C=US' \
  -addext "subjectAltName=DNS:goldmane,DNS:goldmane.kube-system,DNS:goldmane.kube-system.svc,DNS:goldmane.kube-system.svc.cluster.local" \
  -out certs/goldmane.csr
```
Use the following command to generate a certificate `request` for Whisker:
```bash
openssl req \
  -newkey rsa:2048 -nodes -sha256 -keyout certs/whisker-backend.key \
  -subj '/CN=whisker-backend/O=We love Calico/C=US' \
  -addext "subjectAltName=DNS:whisker-backend,DNS:whisker-backend.kube-system,DNS:whisker-backend.kube-system.svc,DNS:whisker-backend.kube-system.svc.cluster.local" \
  -out certs/whisker-backend.csr
```

### Finishing the Certificate Requests and Signing Certificates with our CA

Great, now that we have two certificate requests it is time to sign them with our CA. This allows any workload that uses these certificates to verify their authenticity.

Use the following command to generate and sing your Goldmane certificate:
```bash
openssl x509 -req \
    -in certs/goldmane.csr \
    -CA certs/ca.crt -CAkey certs/ca.key \
    -out certs/goldmane.crt \
    -days 1024 \
    -sha256 -extensions v3_req -extfile certs/req-goldmane.conf
```
Do the same for Whisker:
```bash
openssl x509 -req \
    -in certs/whisker-backend.csr \
    -CA certs/ca.crt -CAkey certs/ca.key \
    -out certs/whisker-backend.crt \
    -days 1024 \
    -sha256 -extensions v3_req -extfile certs/req-whisker.conf
```

### Importing Certificates into the cluster

Now that we have all our certificates signed and ready let's import them into our Kubernetes cluster as a configmap/secret. This is a best practice in Kubernetes since it allows us to change these certificates easily by replacing these configmaps.

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

## Tunning Typha to use certificates

Our next step is to assign certificates to Typha. In order to do that you need to modify the `calico-typha` manifest and add a VolumeMount object to connect the previous generated secrets to the pods, add a volume to mount them in the workloads, and add a bunch of environment variables to let Typha know its certificate time.

Use the following command to modify your typha deployment:
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

**Note**: We've also shared the CA certificate (not the key), this is important since our CA is not a "valid/public" CA we use the `TYPHA_CAFILE` environment variable to inject the CA certificate into the Typha processes. 

## Tunning calico-node to use certificates

Now we need to do the same for `calico-node` daemonset,

**Note:** Since `calico-node` is going to be a client we are setting the `FELIX_TYPHACN` to `typha-server` so it matches the certificate that was issued earlier to the server, this will provide `calico-node` with an additional verification check.

Use the following command to adjust `calico-node` daemonset to use certificates for its communication with Typha: 
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

Before starting the next step use the following command to make sure that your changes are picked up by Calico and Typha.

```bash
kubectl rollout status -n kube-system ds/calico-node deployment/calico-node
```

## Deploying Goldmane

Now that we have our Calico and Typha workloads running and secured, let's start running Goldmane and Whisker.

First we need a service account, service accounts are used to give workloads an identity so the administrator can manage their reach.

Use the following command to generate a service account for Goldmane:
```bash
kubectl create -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: goldmane
  namespace: kube-system
EOF
```

Next, we need to generate a service to point to the goldmane ingestion port, this will be used by any emitter (in this case calico-node) to send flows to the Goldmane server.

Use the following command to generate the service:
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

Goldmane binary, looks for a config file to be present, and we need to supply that file to our Goldmane when its starting.

Use the following command to generate the config file as a ConfigMap.
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

### Tunning Goldmane Deployment

Goldmane binary looks for certain indications to change its behavior, since certificates have finite life time we are using environment variables to tell Goldmane which certificates it should use.

Examine the the following block, it is an example that illustrate the required changes in Goldmane deployment that you will deploy later.
```bash
        - name: SERVER_CERT_PATH
          value: /goldmane-key-pair/tls.crt
        - name: SERVER_KEY_PATH
          value: /goldmane-key-pair/tls.key
        - name: CA_CERT_PATH
          value: /etc/pki/tls/certs/tigera-ca-bundle.crt
```

Similar to the previous parts we need to mount the VolumeMounts to the workload so they an be accessed inside the workload.
```bash
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
```

Finally, we need to mount these VolumeMounts in their respective path inside the workload.
```bash
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
```

**Note:** The following gist file has all the changes that we examined in the above.

Use the following command to deploy Goldmane:
```bash
kubectl create -f https://gist.githubusercontent.com/frozenprocess/5555ec3266133e53510b4bda59f38e42/raw/003335e822557b8bd56b0180418ec1bf6f7232cb/goldmane.yaml
```

## Deploying Whisker UI

Now that we have our Calico and Typha workloads running and secured, let's start running Whisker and Whisker.

First we need a service account, service accounts are used to give workloads an identity so the administrator can manage their reach.

Use the following command to generate a service account for Whisker:
```bash
kubectl create -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: whisker
  namespace: kube-system
EOF
```

Next, we need to generate a service to point to the Whisker UI port, this will be used by Whisker Users (Administrators) to view flowlogs and policies inside whisker. Keep in mind that we recommend using a `ClusterIP` service since flowlogs are sensitive information.

Use the following command to generate the service:
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
  type: ClusterIP
EOF
```

### Tunning Whisker Deployment

Similar to Goldmane binary, Whisker looks for certain indications to change its behavior, since certificates have finite life time we are using environment variables to tell Whisker which certificates it should use. On top of that Whisker needs a URL to know where Goldmane resides this is achieved by setting `GOLDMANE_HOST` environment variable in the deployment.

Examine the the following block, it is an example that illustrate the required changes in Whisker deployment that you will deploy later.

```bash
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
```

Similar to the previous parts we need to mount the VolumeMounts to the workload so they an be accessed inside the workload.
```bash
        volumeMounts:
        - mountPath: /whisker-backend-key-pair
          name: whisker-backend-key-pair
          readOnly: true
        - mountPath: /etc/pki/tls/certs
          name: goldmane-ca-bundle
          readOnly: true
```

Finally, we need to mount these VolumeMounts in their respective path inside the workload.
```bash
      volumes:
      - name: whisker-backend-key-pair
        secret:
          defaultMode: 420
          secretName: whisker-backend-key-pair
      - configMap:
          defaultMode: 420
          name: goldmane-ca-bundle
        name: goldmane-ca-bundle
```
**Note:** The following gist file has all the changes that we examined in the above.

Use the following command to deploy Whisker:
```bash
kubectl create -f https://gist.githubusercontent.com/frozenprocess/5555ec3266133e53510b4bda59f38e42/raw/003335e822557b8bd56b0180418ec1bf6f7232cb/whisker.yaml
```

You can verify Goldmane and Whisker deployment in a Manifest based install by using the following command:
```bash
kubectl rollout status -n kube-system deployment/goldmane
kubectl rollout status -n kube-system deployment/whisker
```

### Accessing Whisker

**Note:** The following step needs to be running if you wish to access Whisker UI using a ClusterIP approach.

Open a new terminal and run the following command:
```bash
kubectl port-forward -n kube-system deployment/whisker 8081:8081
```

Now open a browser and point it to `localhost:8081`, you should see the Whisker UI. However, no flows are going to appear and this is because flow generation happens in Felix (Brian of Calico) and we haven't told that to Calico.

It's time to fiddle with the environment Variables again!

## Instructing Felix to generate Flows Logs and ship them to Goldmane

Now that we have all the certificates in place, all we need to do is to set two more environment variables in the `calico-node` daemonset to instruct Felix to turn on its flow logs generation and where to send them to our Goldmane.
```bash
        - name: FELIX_FLOWLOGSGOLDMANESERVER
          value: goldmane.kube-system.svc:7443
        - name: FELIX_FLOWLOGSFLUSHINTERVAL
          value: "15"
```

Use the following command to enable Flow Log generation:
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

After this change, it will take a moment for all calico-node pods in your cluster to restart. However, if you check your Whisker UI you will not see any flowlogs to appear.

Ah, it's puzzle!

### It is always DNS

Until this point we have configured everything required for our Goldmane and Whisker to run and in a perfect world things should work as expected. However, if you check `calico-node` logs you will see a warning similar to the following:


> You can use the following command to check for these errors in your environment:
> ```bash
> kubectl logs -fn kube-system ds/calico-node
> ```
>

```bash
2025-07-17 14:21:38.502 [WARNING][75] felix/client.go 175: Failed to connect to flow server error=rpc error: code = Unavailable desc = name resolver error: produced zero addresses target="dns:///goldmane.kube-system.svc:7443"
``` 

Ah seems like there is an issue. So back to Kubernetes 101, Pods can be connected to the host network "This means they communicate by using the node IP address" or namespaced. This decision is indicated by a config in your `daemonset/deployment/pod` spec via the `hostNetwork: true` field. 

You might be wondering but what this has to do with our dns issue? Well, everything. Fromt he perspective of calico-node its trying to resolve the internal dns record of `goldmane.kube-system.svc` however, this query is sent to the host DNS server and forwarders which don't know anything about our Kubernetes records. To fix this issue all we need to do is to change the dnsPolicy of our daemonset to `ClusterFirstWithHostNet`. This will allow the first query to go through the cluster.


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
You should be able to see some flows in your Whisker UI shortly.
![alt text](image.png)

Congratulations, you are running Calico Whisker in a manifest based installation!
