# Deployment

[https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/)

<br>
<br>

### Deployment

---

- k8s 오브젝트 중 하나로, 하나 이상의 Pod를 생성하고 업데이트하는 작업을 담당
- Deployment는 ReplicaSet 오브젝트를 사용하여 Pod를 관리하며, 함께 사용하게 되면 Pod 수가 원하는 수와 일치하도록 유지된다.
- 롤링 업데이트 지원 : 애플리케이션의 새로운 버전을 배포하면서 기존 버전에서 새로운 버전으로 천천히 전환 가능
- 즉, 쿠버네티스에서 애플리케이션 배포 및 관리를 쉽게 할 수 있도록 해주는 중요한 오브젝트

<br>

- **ReplicaSet?**
  - Pod를 복제하고 관리하는 리소스로, 지정된 수의 파드를 유지하고, 파드의 상태를 모니터링하며 필요한 경우에 파드를 생성하거나 삭제하여 지정된 수의 파드를 유지
  - 가용성과 확장성을 보장
  - Deployment (컨트롤러)와 함께 사용되어 ReplicaSet은 파드를 생성하고 관리하는데 초점을 두고, Deployment는 롤링 업데이트와 롤백 등과 같은 더 복잡한 배포 작업을 수행하기 위해 ReplicaSet을 내부적으로 사용한다.

<br>
<br>

### Deployment 생성

---

1. 3개의 nginx 파드를 불러오기 위한 `ReplicaSet`을 생성한다.

   ```yaml
   apiVersion: apps/v1 # 사용할 Kubernetes API 버전
   kind: Deployment # 리소스 유형 : Deployment
   metadata:
     name: nginx-deployment # Deployment 리소스 이름
     labels: # Deployment 리소스를 식별하는 데 사용될 레이블 지정
       app: nginx
   spec: # deployment 리소스 스펙
     replicas: 3 # 파드의 복제본 수를 지정
     selector: # Deployment 리소스를 관리하는 파드를 선택하는데 사용
       matchLabels: # 특정 레이블을 가진 파드를 선택하도록
         app: nginx # app: nginx 레이블을 가진 파드를 선택하도록 지정
     template: # pod-template 정의
       metadata:
         labels:
           app: nginx # 해당 레이블을 파드에 부여
       spec: # 파드 스펙 정의 : 파드 내부에서 실행될 컨테이너에 대한 정보
         containers: # 파드 내에서 실행될 컨테이너 정의
           - name: nginx # 컨테이너 이름
             image: nginx:1.14.2 # 컨테이너에서 사용할 Docker 이미지
             ports: # 컨테이너가 사용하는 포트 지정
               - containerPort: 80
   ```

   → 해당 파일을 쿠버네티스 클러스터에 적용하면 **Nginx 웹 서버 컨테이너를 실행하는 파드**를 배포하게 된다.

<br>

1. 다음 명령어로 `Deployment`를 생성한다.

   ```bash
   kubectl --kubeconfig kubeconfig-lena-cluster.yaml apply -f controllers/nginx-deployment.yaml
   ```

   → OUTPUT : `deployment.apps/nginx-deployment created`

- 생성된 파드 확인 및 파드 삭제
  ![image](https://user-images.githubusercontent.com/49095587/228191337-81578328-210f-4e1a-9c9d-a58dad5e80b4.png)

<br>

2. 생성된 Deployment 확인

   ```bash
   kubectl --kubeconfig kubeconfig-lena-cluster.yaml get deployments
   ```

   ![image](https://user-images.githubusercontent.com/49095587/228191394-b00aae3d-31e1-40f2-8a66-91b4124cbcd8.png)

   - `NAME` : 네임스페이스에 있는 디플로이먼트 이름의 목록
   - `READY` : 사용자가 사용할 수 있는 애플리케이션의 레플리카의 수를 표시 (`ready/desired`)
   - `UP-TO-DATE` : 의도한 상태를 얻기 위해 업데이트된 레플리카의 수를 표시
   - `AVAILABLE` : 사용자가 사용할 수 있는 애플리케이션 레플리카의 수를 표시
   - `AGE` : 애플리케이션이 실행된 시간 표시

<br>

3. Deployment의 롤아웃 상태 확인

   ```bash
   kubectl --kubeconfig kubeconfig-lena-cluster.yaml rollout status deployment/nginx-deployment
   ```

   → OUTPUT : `deployment “nginx-deployment” successfully rolled out`

   - `deployment/nginx-deployment` ⇒ `resource/name`
   - **롤아웃** : 쿠버네티스에서 애플리케이션 배포를 수행하는 작업으로, 롤아웃을 통해 새로운 버전의 애플리케이션을 배포하면서 기존 버전과 차이점을 확인하고, 문제가 발생하면 **롤백**을 수행할 수 있다.

<br>

4. Deployment로 생성된 ReplicaSet(rs)를 보려면, `kube get rs`을 실행

   ![image](https://user-images.githubusercontent.com/49095587/228191412-c800e4c1-8bb2-4851-93e3-1e898edb4342.png)

   - `NAME` : 네임스페이스에 있는 레플리카셋 이름의 목록
   - `DESIRED` : Deployment의 생성 시 정의된 의도한 애플리케이션 Replica 수를 표시, 의도한 상태!
   - `CURRENT` : 현재 실행 중인 레플리카의 수를 표시
   - `READY` : 사용자가 사용할 수 있는 애플리케이션의 레플리카의 수를 표시
   - `AGE` : 애플리케이션이 실행된 시간을 표시
   - ReplicaSet의 이름은 항상 `[DEPLOYMENT-NAME]-[HASH]` 형식인데, `HASH` 문자열은 레플리카셋의 `pod-template-hash` 레이블과 같다.
