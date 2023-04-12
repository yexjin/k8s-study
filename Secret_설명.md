# Secret

### Secret

---

- 설정 정보에서 자격증명, 개인 암호화 키, 보안을 유지해야하는 유사한 데이터 같은 종류의 정보는 특히 주의를 기울여야 하는데, 이를 위해 존재하는 쿠버네티스의 중요한 오브젝트
- 키-값 쌍 (Configmap과 매우 유사)
- 아래와 같은 상황에 사용 가능
  - **환경 변수**로 시크릿 항목을 컨테이너에 전달
  - 시크릿 항목을 **볼륨 파일**로 노출
- 노드 자체적으로 메모리에만 시크릿 데이터가 저장되게 하고 물리 저장소에는 기록되지 않게 한다.
- 마스터 노드에는 시크릿을 암호화하지 않은 형식으로 저장하므로, 마스터 노드를 보호하는 것도 필요하다.

  - etcd 저장소를 안전하게 하기 (시크릿을 암호화된 형태로 저장)
  - 권한 없는 사용자는 API 서버를 이용 못하게 하기

    <br>

### ConfigMap vs Secret

---

민감하지 않고, 일반 설정 데이터는 ConfigMap

**민감한 데이터와 민감하지 않는 데이터가 함께 가지고 있는 설정파일이 있다면, 해당 파일은 시크릿 안에 저장해야 한다.**

<br>

### Secret이 가지고 있는 3가지 항목

---

describe 명령어를 작성해보면, Data 아래에 세가지 항목이 뜨는데 이들은 파드 안에서 쿠버네티스 API 서버와 통신할 때 필요한 모든 것을 나타낸다.

- `ca.crt`
- `namespace`
- `token`

<br>

### Secret 생성

---

/pod/inject/secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  username: bXktYXBw
  password: Mzk1MjgkdmRnN0pi
```

시크릿 생성 및 정보 확인

```bash
kubectl --kubeconfig kubeconfig-lena-cluster.yaml apply -f pod/inject/secret.yaml
# OUTPUT
secret/test-secret created

kubectl --kubeconfig kubeconfig-lena-cluster.yaml get secret test-secret
# OUTPUT
NAME          TYPE     DATA   AGE
test-secret   Opaque   2      24s

kubectl --kubeconfig kubeconfig-lena-cluster.yaml describe secrets test-secret
# OUTPUT
Name:         test-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  12 bytes
username:  6 bytes
```

→ Configmap과 마찬가지로 from-literal, from-file을 통해 직접 시크릿을 생성할 수 있다.

`kubectl create secret generic test-secret --from-literal='username=my-app' --from-literal='password=39528$vdg7Jb’`

<br>

### 볼륨을 통해 시크릿 데이터에 접근할 수 있는 파드 생성

---

pod/secret-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
    - name: test-container
      image: nginx
      volumeMounts:
        # name must match the volume name below
        - name: secret-volume
          mountPath: /etc/secret-volume
  # The secret data is exposed to Containers in the Pod through a Volume.
  volumes:
    - name: secret-volume
      secret:
        secretName: test-secret
```

1. 파드에서 실행 중이 컨테이너 셸 가져오기

```bash
kubectl --kubeconfig kubeconfig-lena-cluster.yaml exec -i -t secret-test-pod -- /bin/bash
```

1. **시크릿 데이터는 /etc/secret-volume 에 마운트된 볼륨을 통해 컨테이너에 노출된다.**

   셸에서 /etc/secret-volume 디렉터리의 파일을 나열한다.

   ```bash
   # 컨테이너 내부의 셸에서 실행하자
   ls /etc/secret-volume
   ```

   ```bash
   # 두 개의 파일과 각 파일의 시크릿 데이터 조각을 확인할 수 있다.
   password username
   ```

   ```bash
   # username, password 파일의 내용을 출력
   echo "$( cat /etc/secret-volume/username )"
   echo "$( cat /etc/secret-volume/password )"
   ```

   ```bash
   # 이름과 비밀번호가 출력된다.
   my-app
   39528$vdg7Jb
   ```

   <br>

### 시크릿 데이터를 사용하여 컨테이너 환경 변수 정의하기

---

**1️⃣ 단일 시크릿 데이터**로 컨테이너 환경 변수 정의하기

- 환경 변수를 시크릿의 키-값 쌍으로 정의

```bash
kubectl --kubeconfig kubeconfig-lena-cluster.yaml create secret generic backend-user --from-literal=backend-username='backend-admin’
```

- 시크릿에 정의된 backend-username 값을 파드 명세의 SECRET_USERNAME 환경 변수에 할당

/pod/env-single-secret.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-single-secret
spec:
  containers:
    - name: envars-test-container
      image: nginx
      env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: backend-user
              key: backend-username
```

- 파드 생성
- 셸에서 SECRET_USERNAME 컨테이너 환경 변수의 내용 출력

```bash
kubectl exec -i -t env-single-secret -- /bin/sh -c 'echo $SECRET_USERNAME'
# OUTPUT
backend-admin
```

<br>

2️⃣ 여러 시크릿 데이터로 컨테이너 환경 변수 정의하기

- 추가 시크릿 생성

```bash
kubectl --kubeconfig kubeconfig-lena-cluster.yaml create secret generic db-user --from-literal=db-username='db-admin'
```

- 파드 명세에 환경 변수 정의

/pod/envvars-multiple-secrets.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envvars-multiple-secrets
spec:
  containers:
    - name: envars-test-container
      image: nginx
      env:
        - name: BACKEND_USERNAME
          valueFrom:
            secretKeyRef:
              name: backend-user
              key: backend-username
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-user
              key: db-username
```

- 셸에서 컨테이너 환경 변수를 출력

```bash
kubectl exec -i -t envvars-multiple-secrets -- /bin/sh -c 'env | grep _USERNAME'

# OUTPUT
DB_USERNAME=db-admin
BACKEND_USERNAME=backend-admin
```

<br>

3️⃣ 시크릿의 **모든 키-값 쌍을 컨테이너 환경 변수로 구성**하기

- 여러 키-값 쌍을 포함하는 시크릿 생성

```bash
kubectl create secret generic test-secret --from-literal=username='my-app' --from-literal=password='39528$vdg7Jb'
```

- envFrom 을 사용하여 시크릿의 모든 데이터를 컨테이너 환경 변수로 정의한다. 시크릿의 키는 파드에서 환경 변수의 이름이 된다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-secret
spec:
  containers:
    - name: envars-test-container
      image: nginx
      envFrom:
        - secretRef:
            name: test-secret
```

![image](https://user-images.githubusercontent.com/49095587/231386570-22f3a1e0-1ef0-4cd1-8df7-d9d6eb12dca1.png)
