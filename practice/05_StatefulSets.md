# StatefulSets

Deployment와 다르게, **StatefulSet은 Storage Volume을 사용해서 워크로드에 지속성을 제공해야하는 경우** 사용될 수 있다.

**StatefulSet의 구성요소** (controllers/nginx-statefulset.yaml) k8s 공식문서

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # .spec.template.metadata.labels 와 일치해야 한다
  serviceName: "nginx"
  replicas: 3 # 기본값은 1
  minReadySeconds: 10 # 기본값은 0 (컨트롤 플레인이 이전 버전을 업데이트 하기 전에, 업데이트된 파드가 실행 및 준비될 때까지 기다리는 최소 준비시간 초)
  template:
    metadata:
      labels:
        app: nginx # .spec.selector.matchLabels 와 일치해야 한다
    spec:
			terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

→ `kind : Service` 는 외부로 포트를 열어두겠다… 서비스를 노출시키는 것!!

**PVC(**`pvc-test`**)를 이용하여 PV를 StatefulSet에 붙이는 예시** (controllers/pvc-nginx-statefulset.yaml)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: pvc-test
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates: # PV Provisioner에 의해 프로비전된 PV를 이용하는 안정적인 스토리지 제공
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
        storageClassName: csi-cinder-sc-delete
```

- `web`이라는 이름의 `StatefulSet`을 생성합니다.
- `nginx` 이미지를 사용하는 3개의 파드를 생성합니다.
- 각 파드는 `/usr/share/nginx/html` 디렉토리에 `pvc-test` PVC를 마운트합니다.
- `volumeClaimTemplates` 필드를 사용하여 각 파드에 대한 PVC를 정의합니다.

결과

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d1cddd58-d294-4e33-a31d-de6ce358c8eb/Untitled.png)

### `.spec.persistentVolumeClaimRetentionPolicy`

- StatefulSets의 생애주기동안 PVC를 삭제할 것인지, 삭제한다면 어떻게 삭제하는지를 관리
- 각 StatefulSets에 대해 두가지 정책을 설정
  - `whenDeleted` : statefulset이 삭제될 때 적용될 볼륨 유지 동작 설정
  - `whenScaled` : 레플리카 수가 줄어들 때 (스케일 다운) 볼륨 유지 동작 설정
- 각 설정 가능한 정책에 대해 값을 두가지로 설정
  - `Delete` : `volumeClaimTemplate` statefulset으로부터 생성된 PVC는 위의 정책에 영향을 받는 각 파드에 대해 삭제됨
  - `Retain` (기본값) : 파드가 삭제되어도 `volumeClaimTemplate` 으로부터 생성된 PVC는 영향을 받지 않는다.
