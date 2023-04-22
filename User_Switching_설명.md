# User Switching

> User 만들고 Switching

1. OpenSSL로 Secret 생성하기

```bash
openssl genrsa -out lena.key 2048
```

1. Private Key를 이용한 CSR(Certificate Signing Request) 생성

```bash
openssl req -new -key lena.key -out lena.csr

# 생성옵션에서 Common Name과 A Challenge password만 작성해도 괜찮다.
```

1. CRT 인증서 생성

```bash
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: lena
spec:
  request: ${Private key}
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```

1. 현재 kubeconfig 파일에 정의된 모든 컨텍스트의 목록을 출력

```bash
kubectl --kubeconfig kubeconfig-lena-cluster.yaml config get-contexts

# OUTPUT
CURRENT   NAME                              CLUSTER        AUTHINFO             NAMESPACE
*         lean                              kubernetes     lena
          lena-cluster-admin@lena-cluster   lena-cluster   lena-cluster-admin
```

5. 현재 활성화된 Kubernetes 클러스터와 사용자를 변경

```bash
kubectl --kubeconfig kubeconfig-lena-cluster.yaml config use-context lena-cluster-admin@lena-cluster

# OUTPUT
Switched to context "lena-cluster-admin@lena-cluster".
```
