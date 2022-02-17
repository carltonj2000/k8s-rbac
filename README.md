# RBAC Introduction

The content in this repository is based on the
[Understanding Kubernetes RBAC | Access control basics explained](https://youtu.be/jvhKOAyD8S8)
video.

```bash
kind create cluster --name rbac --image kindest/node:v1.23.3
```

Get CA certificate via:

```bash
docker ps
docker exec -it rbac-control-plane bash
cd /etc/kubernetes/pki
exit
docker cp rbac-control-plane:/etc/kubernetes/pki/ca.crt ca.crt
docker cp rbac-control-plane:/etc/kubernetes/pki/ca.key ca.key
```

Create certs using an alpine container as noted below.

```bash
docker run -it -v ${PWD}:/work -w /work -v ${HOME}:/root/ --net host alpine sh
apk add openssl
# gen bob's private key
openssl genrsa -out bob.key 2048
# gen cert signing req (CSR)
openssl req -new -key bob.key -out bob.csr -subj "/CN=Bob Smith/O=Shopping"
# gen bob cert by signing CSR with CA
openssl x509 -req -in bob.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out bob.crt -days 1
```

Setup kubectl in the alpine container created above
or just use kubeclt on the host if installed and skip this step.

```
apk add curl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
```

After kubectl is install on the host or the alpine container noted above.

```bash
export KUBECONFIG=~/.kube/configs/rbac.yaml
# below cmd with create a skeleton file at the path noted above with warnings
kubectl config set-cluster dev-cluster --server=https://127.0.0.1:46249 \
--certificate-authority=ca.crt \
--embed-certs=true

kubectl config set-credentials bob --client-certificate=bob.crt --client-key=bob.key --embed-certs=true
kubectl config set-context dev --cluster=dev-cluster --namespace=shopping --user=bob
kubectl config use-context dev

kubectl get pods
```

With the original kind KUBECONFIG not the one `export`ed above.

```bash
kubectl create ns shopping
kubectl -n shopping apply -f ./role.yaml # create the role
kubectl -n shopping apply -f ./rolebinding.yaml # bind user to role
```

Create service accounts for app (processes in containers inside pods) to access the k8s apiserver.

```bash
kubectl -n shopping apply -f serviceaccount.yaml
```

With user bob's KUBECONFIG

```bash
kubectl -n shopping apply -f pod.yaml
kubectl -n shopping exec -it shopping-api -- bash
# Point to the internal API server hostname
APISERVER=https://kubernetes.default.svc

# Path to ServiceAccount token
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount

# Read this Pod's namespace
NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)

# Read the ServiceAccount bearer token
TOKEN=$(cat ${SERVICEACCOUNT}/token)

# Reference the internal certificate authority (CA)
CACERT=${SERVICEACCOUNT}/ca.crt

# List pods through the API
curl --cacert ${CACERT} --header "Authorization: Bearer $TOKEN" -s ${APISERVER}/api/v1/namespaces/shopping/pods/

# we should see an error not having access
```

With default KUBECONFIG

```bash
kubectl -n shopping apply -f serviceaccount-role.yaml
kubectl -n shopping apply -f serviceaccount-rolebinding.yaml
```

With user bob's KUBECONFIG

```bash
# List pods through the API
curl --cacert ${CACERT} --header "Authorization: Bearer $TOKEN" -s ${APISERVER}/api/v1/namespaces/shopping/pods/

# we should see NO error and have access
```
