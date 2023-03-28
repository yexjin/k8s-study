# DaemonSet

<br>
<br>

### DaemonSet

---

- 데몬셋은 모든(또는 일부) 노드가 파드의 사본을 실행하도록 함.
- **클러스터의 모든 노드에 특정 Pod 인스턴스를 배포하고 유지하는 리소스**
- 새로운 노드가 클러스터에 추가되면 데몬셋 컨트롤러는 해당 노드에도 자동으로 파드를 배포하여 클러스터 확장을 자동으로 처리
- 데몬셋을 삭제하면 데몬셋이 생성한 파드들이 정리됨.
- 데몬셋의 용도
  - 모든 노드에서 클러스터 스토리지 데몬 실행
  - 모든 노드에서 로그 수집 데몬 실행
  - 모든 노드에서 노드 모니터링 데몬 실행
  ⇒ 백그라운드 작업과 같은 작업을 실행하는데 사용됨.
- 즉, **데몬셋은 클러스터의 모든 노드에서 특정 작업을 수행하는 파드를 배치하는데 사용되며, 이를 통해 클러스터 전체에서 동일한 서비스 또는 작업을 수행할 수 있다.**

<br>
<br>

### DaemonSet 생성

---

`deamonset.yaml`로 `fluentd-elasticsearch` 도커 이미지를 실행하는 데몬셋을 설명해보자.

해당 데몬셋을 생성하면 **Kubernetes 클러스터의 모드 노드에서 실행되며, 모든 노드의 로그를 수집하여 Elasticsearch에 전달한다.**

- **fluentd-elasticsearch?**
  **로그 수집 및 분석 도구**로서 Elastic Stack에서 사용되는 오픈 소스 소프트웨어

```yaml
**apiVersion**: apps/v1
**kind**: DaemonSet
**metadata**:
  name: fluentd-elasticsearch
  namespace: kube-system            # kube-system 네임스페이스는 일반적으로 k8s 관리에 사용되는 시스템 네임스페이스
  labels:
    k8s-app: fluentd-logging
**spec**:                               # 데몬셋의 세부정보가 표시
  selector:                         # pod 셀렉터 : 데몬셋을 실행할 파드를 정의
    matchLabels:
      name: fluentd-elasticsearch
  template:                         # 데몬셋을 실행할 파드를 정의
    metadata:
      labels:
        name: fluentd-elasticsearch # 해당 레이블을 가진 파드를 실행 (위의 .spec.selector.matchLabels의 name과 같은 값을 가져야한다. 일치하지 않을 경우 API에 의해 데몬셋 구성이 거부된다.)
    spec:
      tolerations:
      # 이 **톨러레이션(toleration)**은 데몬셋이 **컨트롤 플레인 노드**에서 실행될 수 있도록 만든다.
      # 컨트롤 플레인 노드가 이 파드를 실행해서는 안 되는 경우, 이 톨러레이션을 제거한다.
      - key: node-role.kubernetes.io/control-plane   # 톨러레이션의 키
        operator: Exists                             # 톨러레이션의 연산자
        effect: NoSchedule                           # 톨러레이션의 효과
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch            # 사용하는 컨테이너
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2    # 해당 이미지를 사용하여 컨테이너를 생성
        resources:
          limits:
            memory: 200Mi    # 메모리 리소스 제한
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:        # 컨테이너와 호스트 머신 간 볼륨 마운트를 설정
				# 두가지 볼륨을 마운트(/var/log, /var/lib/docker/containers)
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30       # 파드가 종료될 때 grace period를 30초로 설정
      # grace-period : k8s가 pod를 삭제하기 전에, 해당 pod가 종료되거나 작업을 완료할 수 있도록 설정한 시간 간격 (기본값은 30초)
      volumes:     # 각 볼륨이 연결되는 경로 설정
      - name: varlog
        hostPath:  # 호스트 머신에서 볼륨을 찾을 경로 설정
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

<br>

해당 파일을 기반으로 데몬셋을 생성한다.

```bash
kubectl --kubeconfig kubeconfig-lena-cluster.yaml apply -f controllers/daemonset.yaml
```

→ OUTPUT : `daemonset.apps/fluentd-elasticsearch created`

<br>

- ⚠️ Problem
  daemonset이 만든 파드를 확인할 수 없음
  ![image](https://user-images.githubusercontent.com/49095587/228191756-33428cd9-a2cc-4179-808c-e2b292ac009b.png)

<br>
<br>

### 데몬셋 생성 파일 파헤쳐보기

---

- Pod 템플릿의 `RestartPolicy`는 `Always`를 가져야 하며, 명시되지 않은 경우 기본으로 `Always`가 된다.
- `.spec.template.spec.nodeSelector`를 명시하면 데몬셋 컨트롤러는 nodeSelector와 일치하는 노드에 파드를 생성한다.
- `.spec.template.spec.affinity`를 명시하면 데몬셋 컨트롤러는 노드 어피니티와 일치하는 노드에 파드를 생성한다.
  - `node affinity`: 특정 파드가 스케줄링 되기 위한 node의 제약 사항을 설정하는 기능
- 만약 `.spec.template.spec.nodeSelector`와 `.spec.template.spec.affinity`중 하나를 명시하지 않으면 데몬셋 컨트롤러는 모든 노드에서 파드를 생성한다.
