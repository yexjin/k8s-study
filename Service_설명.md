# Service 정리

🔗 [참고링크](https://kubernetes.io/ko/docs/concepts/services-networking/service/)

### Service

---

- 클라이언트가 **파드를 검색하고 통신을 가능**하게 한다.
- 파드는 언제든지 클러스터 안에서 이동할 수 있어서 IP 주소가 변경될 수 있는데, 이런 경우 해당 파드에 대한 서비스를 만들고 클러스터 외부에서 액세스할 수 있도록 구성하면 외부 클라이언트가 파드에 연결할 수 있는 하나의 **고정 IP 주소**가 노출된다.
- 외부 뿐만 아니라 내부 클라이언트도 서비스로 파드 접근 가능

`Label Selector` : 어떤 파드가 서비스에 속하는지 정함

<br>

### Service 만들기

---

kubia-nginx-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-nginx-pod
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80
          name: http-web-svc
```

kubia-nginx-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nginx-svc
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
    - name: name-of-service-port
      protocol: TCP
      port: 80
      targetPort: http-web-svc
```

서비스에 내부 클러스터 IP가 할당됐는지 확인

```bash
kubectl --kubeconfig kubeconfig-lena-cluster.yaml get svc

# OUTPUT
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>           443/TCP        26d
kubia        ClusterIP      10.97.162.150   <none>           80/TCP         46s
```

ClusterIP 타입이므로 클러스터 내부에서만 액세스 할 수 있다

<br>

### Service Endpoint

---

```bash
kubectl --kubeconfig kubeconfig-lena-cluster.yaml describe service kubia-nginx-svc

# OUTPUT
Name:              kubia-nginx-svc
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app.kubernetes.io/name=proxy   # Service의 Pod selector는 엔드포인트 목록을 만드는데 사용된다.
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.108.126.164
IPs:               10.108.126.164
Port:              name-of-service-port  80/TCP
TargetPort:        http-web-svc/TCP
Endpoints:         192.168.163.200:80     # 이 서비스의 엔드포인트를 나타내는 파드 IP와 포트 목록
Session Affinity:  None
Events:            <none>
```

→ Pod Selector 없이 서비스를 만들면 쿠버네티스는 엔드포인트 리소스를 만들지 못한다. (서비스에 포함된 파드가 무엇인지 알 수 없어서)

<br>

### 클러스터 외부에 있는 서비스 연결 (외부 → 내부)

---

Selector가 없는 외부 서비스는 엔드포인트 리소스가 자동으로 생성되지 않으므로, 엔드포인트 서비스를 수동 생성해야한다.

→ 엔드포인트 오브젝트는 서비스와 이름이 같아야 하고 서비스를 제공하는 대상 IP 주소와 포트 목록을 가져야 한다.

<br>

### 외부 클라이언트에 서비스 노출 (내부 → 외부)

---

노출 방법

1. 노드포트로 서비스 유형 설정
2. 서비스 유형을 노드포트 유형의 확장인 로드밸런서로 설정
3. 단일 IP 주소로 여러 서비스를 노출하는 인그레스 리소스 만들기

<br>

### 요약

---

각 서비스를 제공하는 파드 인스턴스 수에 관계 없이 쿠버네티스 서비스 리소스를 생성해 애플리케이션에서 사용가능한 서비스를 노출하는 방법을 알아봤다.

1. 안정된 단일 IP 주소와 포트로 특정 레이블 셀렉터와 일치하는 여러 개의 파드를 노출한다.
2. 기본적으로 클러스터 내부에서 서비스에 액세스할 수 있지만 유형을 노드포트 또는 로드벨런서로 설정해 클러스터 외부에서 서비스에 액세스 할 수 있다.
3. 파드가 환경변수를 검색해 IP 주소와 포트로 서비스를 검색할 수 있다. `학습X`
4. 관련된 엔드포인트 리소스를 만드는 대신 셀렉터 설정 없이 서비스 리소스를 생성해 클러스터 외부에 있는 서비스를 검색하고 통신할 수 있다.
5. ExternalName 서비스 유형으로 외부 서비스에 대한 DNS CNAME 별칭을 제공한다. `학습X`
6. 단일 인그레스로 여러 HTTP 서비스를 노출한다 (단일 인그레스 컨트롤러 IP 사용) `학습X`
7. 파드 컨테이너의 레디니스 프로브는 파드를 서비스 엔드포인트에 포함해야하는지 여부를 결정한다. `학습X`
8. 헤드리스 서비스를 생성하면 DNS로 파드 IP를 검색할 수 있다. `학습X`
