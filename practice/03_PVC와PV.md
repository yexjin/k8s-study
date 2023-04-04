# PVC, PV

### 0️⃣ 사전 설정

---

- 클러스터에서 영구 볼륨을 사용하기 위해 일반적으로 스토리지와 PersistentVolume 객체를 직접 구성해야 한다.
- Kubernetes Engine에서는 **CSI(Container Storage Interface) Provisioner**를 설정하여 카카오 i 클라우드 Block storage를 영구 볼륨으로 사용할 수 있다.
- 클러스터에 CSI Provisioner을 설정하면, 간단하게 PersistentVolumeClaim을 생성하여 영구 볼륨을 생성할 수 있다.

- CSI Provisioner
  Container Storage Interface Provisioner의 약자로 이 프로비저너를 설정함으로써 간단하게 **`PersistentVolume`**을 생성하여 영구 볼륨을 생성할 수 있다.
- **PersistentVolumeClaim**

  - Pod에서 사용할 수 있는 **영구적인 스토리지 볼륨을 동적으로 할당**하기 위한 추상화된 개념
  - 일반적으로 Kubernetes 클러스터에서 스토리지는 PersistentVolume(PV) 객체로 나타내어지며, 이 객체는 클러스터 내의 스토리지 리소스를 나타내는데 사용된다. 하지만, 이러한 PV를 직접 사용하지 않고 PVC를 사용하여 PV를 요청하고 사용할 수 있다.
  - 즉, **PVC를 사용하면 Pod에서 스토리지를 사용하기 위해 일일이 PV를 만들 필요 없이 스토리지를 동적으로 할당하고 사용**

- 블록 스토리지 CSI Provisioner 설정하기
  [블록 스토리지 CSI(Container Storage Interface) Provisioner 설정하기](https://console.kakaoi.io/docs/posts/k8se/k8se_htg/2021-05-31-k8se_htg_dynamicPV/k8se_htg_dynamicPV#%EB%B8%94%EB%A1%9D-%EC%8A%A4%ED%86%A0%EB%A6%AC%EC%A7%80-csi-provisioner-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0)
  - YAML 파일로 CSI Provisioner 배포
  - Helm을 사용하여 CSI Provisioner 배포
    - 쿠버네티스 패키지 관리 도구인 Helm을 사용하여 Dynamic Volume Provisioning을 설정
  - ERROR
    ![image](https://user-images.githubusercontent.com/49095587/229854710-94a2ff26-df86-4a45-b4c5-973fa5fdeb56.png)
    ![image](https://user-images.githubusercontent.com/49095587/229854742-3845dfe7-42c4-4da6-a2d5-60f27be1b632.png)
    ![image](https://user-images.githubusercontent.com/49095587/229854975-f5516524-3b2a-4fcd-a6c0-5c4cce3b99be.png)
    **결론 : 최신버전의 Helm 설치하기**

<br>
<br>

### 1️⃣ Persistent Volume 동적 프로비저닝 테스트

---

PVC(PersistentVolumeClaim)을 적용해서 PV가 동적으로 생성되는 것을 확인하고, 파드에 연결하는 테스트 수행

1. PVC를 생성하기 위해 아래 YAML을 배포

   ```yaml
   # pv/pvc-test.yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: pvc-test
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
     storageClassName: csi-cinder-sc-delete
   ```

   → `pvc-test` 라는 이름의 PVC를 생성

   → `csi-cinder-sc-delete` 라는 이름의 스토리지 클래스에서 10Gi 의 저장소를 요청

   ```bash
   kubectl apply -f pv/pvc-test.yaml

   # OUTPUT : persistentvolumeclaim/pvc-test created

   ```

2. PVC에 맞게 PV가 동적으로 생성된 것을 확인할 수 있다.

   ```bash
   kubectl get pv,pvc
   ```

   ![image](https://user-images.githubusercontent.com/49095587/229855063-5dc68c4d-a182-4119-9ad7-808bdccaba22.png)

<br>

### 2️⃣ PVC를 사용하는 파드 생성

---

앞서 생성한 PV를 사용하는 파드를 배포해보자

1. 파드를 생성하기 위한 YAML을 배포

   ```yaml
   # pv/task-pv-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: task-pv-pod
   spec:
     volumes:
       - name: task-pv-storage
         persistentVolumeClaim:
           claimName: pvc-test
     containers:
       - name: task-pv-container
         image: nginx
         ports:
           - containerPort: 80
             name: "http-server"
         volumeMounts:
           - mountPath: "/usr/share/nginx/html"
             name: task-pv-storage
   ```

   → `spec.volumes` : PV를 참조하는 볼륨을 정의

   → `spec.containers.volumeMounts` : 볼륨을 컨테이너 파일 시스템 경로에 마운트

   - `task-pv-pod` 라는 이름의 Pod를 생성
   - `task-pv-storage`라는 이름의 볼륨을 사용
     - 이 볼륨은 `pvc-test` PVC와 연결되며
     - 이 볼륨은 `/usr/share/nginx/html` 경로에 마운트 됨
   - 컨테이너 파일 시스템에는`task-pv-storage`라는 볼륨이 마운트 됨

   ```bash
   kubectl apply -f pv/task-pv-pod.yaml

   # OUTPUT : pod/task-pv-pod created
   ```

2. PV가 파드에 잘 마운트 되었는지 확인

   ```bash
   # POD 확인
   kubectl get pods

   # OUTPUT
   NAME                                READY   STATUS    RESTARTS   AGE
   task-pv-pod                         1/1     Running   0          88s

   # 조회한 파드의 컨테이너를 조회 (현재 파일 시스템의 디스크 사용 현황 정보)
   kubectl exec -ti task-pv-pod -- df -h

   # OUTPUT
   Filesystem      Size  Used Avail Use% Mounted on
   /dev/vdb        9.8G   24K  9.8G   1% /usr/share/nginx/html
   ```

   - `kubectl exec`: Kubernetes에서 실행 중인 Pod 내부의 컨테이너에서 명령어를 실행하는 명령어입니다.
   - `ti`: 명령어를 실행할 컨테이너에 대한 TTY와 stdin을 활성화하는 옵션입니다. 이 옵션은 명령어 실행 중에 입력 및 출력을 처리하기 위해 터미널을 사용할 수 있게 합니다.
   - `task-pv-pod`: 명령어를 실행할 Pod의 이름입니다.
   - `-`: 명령어 실행을 위한 구분 기호입니다.
   - `df -h`: 실행할 명령어로, 현재 파일 시스템의 디스크 사용 현황을 보여주는 명령어입니다. **`h`** 옵션은 디스크 사용량을 사람이 읽기 쉬운 형식으로 표시합니다.
