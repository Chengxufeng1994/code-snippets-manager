kubectl create ns cert-manager
kubectl create ns harbor

helm repo add jetstack <https://charts.jetstack.io>
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.16.1 \
  --set crds.enabled=true

## create root ca

echo "create root ca"
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=TW/ST=Taiwan/L=Taipei City/O=local/OU=Personal/CN=MyPersonal Root CA" \
 -key ca.key \
 -out ca.crt
openssl x509 -noout -text -in ca.crt

## 使用 cert-manager

### create k8s secret

kubectl create secret tls root-ca-secret \
   --cert=ca.crt \
   --key=ca.key \
   --namespace=cert-manager

### create k8s cluster issuer

cat <<EOF >my-ca-cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-cluster-issuer
spec:
  ca:
    secretName: root-ca-secret
EOF
kubectl apply -f my-ca-cluster-issuer.yaml -n cert-manager

## create certificate

cat <<EOF > harbor-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: harbor-localhost-domain
  namespace: harbor
spec:
  secretName: harbor-secret
  renewBefore: 360h
  subject:
    organizations:
    - local
  commonName: harbor.localhost.domain.com
  isCA: false
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
    rotationPolicy: Always
  usages:
    - server auth
    - client auth
  dnsNames:
    - harbor.localhost.domain.com
  issuerRef:
    name: my-ca-cluster-issuer
    kind: ClusterIssuer
    group: cert-manager.io
EOF

## 使用 openssl

### create domain csr

echo "create csr"
openssl genrsa -out harbor.localhost.domain.com.key 4096
openssl req -sha512 -new \
    -subj "/C=TW/ST=Taiwan/L=Taipei City/O=local/OU=Personal/CN=harbor.localhost.domain.com" \
    -key harbor.localhost.domain.com.key \
    -out harbor.localhost.domain.com.csr

cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=*.localhost.domain.com
DNS.2=localhost.domain.com
DNS.3=localhost
IP.1=192.168.49.2
EOF

### create cert

openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.localhost.domain.com.csr \
    -out harbor.localhost.domain.com.crt

openssl x509 -noout -text -in harbor.localhost.domain.com.crt
openssl x509 -inform PEM -in harbor.localhost.domain.com.crt -out harbor.localhost.domain.com.cert

### create k8s secret

kubectl create secret tls harbor-secret --cert harbor.localhost.domain.com.crt --key harbor.localhost.domain.com.key -n harbor

helm repo add harbor <https://helm.goharbor.io>
helm pull harbor/harbor --untar
helm install --namespace harbor harbor harbor/
helm upgrade -n harbor harbor .


## Test deploy image as k8s deployment
kubectl create secret docker-registry harbor \
  --docker-server=harbor.home.lab \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --docker-email=ZZZZ@gmail.com \
  -n default

cat <<EOF > my-cicd.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-cicd-deployment
  labels:
    app: cidc
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cicd
  template:
    metadata:
      labels:
        app: cicd
    spec:
      imagePullSecrets:
      - name: harbor
      containers:
      - name: cicd
        image: harbor.localhost.domain.com/library/cicd:v0.2.0
        ports:
        - containerPort: 8000
EOF

## Reference

* [openssl 自發簽證](https://hackmd.io/@yzai/rJXYxFpmq)
- [有個私人的港口還好吧：私有儲存庫 Harbor (一)(使用 nodePort](https://ithelp.ithome.com.tw/articles/10300036)
- [harbor and cert-manager self signed ca](https://yuweisung.medium.com/harbor-cert-manager-self-signed-ca-and-containerd-docker-troubleshooting-16b48b34503d)
