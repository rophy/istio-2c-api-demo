# istio-2c-api-demo

Demo of exposing kubectl across two kubernetes clusters with istio.


## Setup istio multicluster

Reference: https://istio.io/v1.13/docs/setup/install/multicluster/


```bash
kind create cluster --name=cluster1 --image=kindest/node:v1.23.17
kind create cluster --name=cluster2 --image=kindest/node:v1.23.17


export CTX_CLUSTER1=kind-cluster1
export CTX_CLUSTER2=kind-cluster2

curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.13.4 TARGET_ARCH=x86_64 sh -
cd istio-1.13.4
export PATH=$PWD/bin:$PATH
mkdir -p certs
pushd certs
make -f ../tools/certs/Makefile.selfsigned.mk root-ca
make -f ../tools/certs/Makefile.selfsigned.mk cluster1-cacerts
make -f ../tools/certs/Makefile.selfsigned.mk cluster2-cacerts

kubectl create namespace istio-system --context $CTX_CLUSTER1
kubectl create secret generic cacerts --context $CTX_CLUSTER1 -n istio-system \
      --from-file=cluster1/ca-cert.pem \
      --from-file=cluster1/ca-key.pem \
      --from-file=cluster1/root-cert.pem \
      --from-file=cluster1/cert-chain.pem

kubectl create namespace istio-system --context $CTX_CLUSTER2
kubectl create secret generic cacerts --context $CTX_CLUSTER2 -n istio-system \
      --from-file=cluster2/ca-cert.pem \
      --from-file=cluster2/ca-key.pem \
      --from-file=cluster2/root-cert.pem \
      --from-file=cluster2/cert-chain.pem

popd

cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF
istioctl install --context="${CTX_CLUSTER1}" -y -f cluster1.yaml

cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network1
EOF
istioctl install --context="${CTX_CLUSTER2}" -y -f cluster2.yaml


istioctl create-remote-secret \
    --context="${CTX_CLUSTER1}" \
    --name=cluster1 | \
    kubectl apply -f - --context="${CTX_CLUSTER2}"




```
