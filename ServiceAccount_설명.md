# ServiceAccount

<aside>

🤔 **인증** vs **인가
인증 : 인증은 사용자의 신원을 검증하는 행위로서 보안 프로세스에서 첫 번째 단계
인가 : 사용자에게 특정 리소스나 기능에 액세스할 수 있는 권한을 부여하는 프로세스 (= 권한 부여)**

</aside>

<br>

### 인증

---

API 서버가 파드에서 실행되는 애플리케이션에게 요청을 받으면 API 서버는 인증 플러그인 목록을 거쳐 인가 단계를 진행한다.

**인증 플러그인 종류**

1. 클라이언트 인증서
2. HTTP 헤더로 전달된 인증 토큰
3. 기본 HTTP 인증
4. 기타

<br>

### ServiceAccount(sa)

---

- 파드 내부에서 실행되는 애플리케이션이 API 서버에 자신을 인증하는 방법

1. 애플리케이션은 요청에 serviceAccount의 토큰을 전달해서 이 과정을 수행한다.
2. 인증플러그인은 serviceAccount를 인증하고 serviceAccount의 사용자 이름을 API 서버 코어로 전달한다.

**serviceAccount과 Pod**

- sa는 namespace 별로 범위가 지정되며 하나의 네임스페이스를 여러 파드가 사용할 수 있다.
- 파드 매니페스트에 sa의 이름을 지정해 파드에 sa를 할당한다.

<br>

### sa 생성

---

- default sa가 있지만, 보안적인 문제로 직접 생성해보는 것이 좋다.
- sa를 만들고 이를 시크릿과 연결하고 파드에 할당해보자

1️⃣ serviceAccount 생성

```bash
kubectl create serviceaccount foo
```

Pod는 원하는 모든 시크릿을 마운트 할 수 있었는데, 파드가 serviceAccount의 마운트 가능한 시크릿 목록에 있는 시크릿만 마운트 하도록 파드의 serviceAccount를 설정할 수도 있다.

이를 위해선 `[kubernetes.io/enforce-mountable-secrets=”true](http://kubernetes.io/enforce-mountable-secrets=”true) ”.` 어노테이션을 포함하고 있어야 한다.

2️⃣ pod에 sa할당

/pod/ex-for-sa.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ex-for-sa
spec:
  serviceAccountName: foo
  automountServiceAccountToken: true
  containers:
    - name: main
      image: tutum/curl
      command: ["sleep", "9999999"]
    - name: ambassador
      image: luksa/kubectl-proxy:1.6.2
```

`automountServiceAccountToken` 은 해당 pod에 sa token을 자동으로 마운트할지의 여부를 나타냄

→ `true` : default 설정, Kubernetes는 Pod의 /var/run/secrets/kubernetes.io/serviceaccount 디렉토리에 서비스 계정 토큰을 마운트하고, Pod 내부에서 이를 사용할 수 있도록 한다. 이 디렉토리에는 서비스 계정 토큰과 함께 해당 서비스 계정의 CA 인증서와 키가 저장된다.

→ `false` : Kubernetes는 서비스 계정 토큰을 마운트하지 않는다. 이 경우, Pod가 API 서버와 통신하려면 수동으로 서비스 계정 토큰을 가져와야 한다. 보안상의 이유로 특정 Pod에서 서비스 계정 토큰을 사용할 필요가 없는 경우 false로 사용한다.

<br>

3️⃣ Pod의 컨테이너에 serviceAccount 토큰이 마운트 되었는지 확인

```bash
  kubectl --kubeconfig kubeconfig-lena-cluster.yaml exec -it ex-for-sa -c sleep-com -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
  eyJhbGciOiJSUzI1NiIsImtpZCI6IklhVWdMaDNmYXRxQXhJVFA1blBMd2F0OFJpMjRtVmhXNUV5bzFvWTlrY0EifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzEzMzEyNzU2LCJpYXQiOjE2ODE3NzY3NTYsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJleC1mb3Itc2EiLCJ1aWQiOiI0YjQ3NDhmOC1iMDMzLTQ5NzQtYTRmYS00OWE4ZWFkZmJhNTUifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImZvbyIsInVpZCI6IjFhOTlhNGExLTk2NWYtNDcwZC05NGJhLTA5OWViYTdiYTVjZCJ9LCJ3YXJuYWZ0ZXIiOjE2ODE3ODAzNjN9LCJuYmYiOjE2ODE3NzY3NTYsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmZvbyJ9.Nfc2CDwttsArDYoXSUqZnQg4Iv_Fb-8A4kAf-SkUQZl0ztvUYM501j7Rff941iOKeBAVrCMz2uV1AixlwIzROaTuz9k9xDvLfwEb5fU5Y2-ozRaylZPFOIRlnTjsYcv9nEno9GHDRvyNA6slRaJcnwe4veSJdk2IiNqsBpi1VeWNEM1CrD25T9tTePOa9zZCgAI3UpQFNgdL9uTPxLX8_09ODxzmr32m9PKXZ_aHxT7er7Ha_Rq-WSw0Zax3jGGt_fIyFUdcL6e_v9hpn6oDDtRzuLVHTNQXwMX5w2pttQcEodWMZBcSzcr55wNRth_wNVB32V_FEE41ZQDggsxsOQ%
```
