# Kubernetes Architecture 및 Component 설명

![image](https://user-images.githubusercontent.com/49095587/230916161-1cabd453-76ef-4aee-b2bb-3c148ec47c80.png)

### `etcd`

- 모든 상태와 데이터를 저장
- 분산 시스템으로 구성하여 안전성을 높임 (고가용성)
- 가볍고 빠르면서 정확하게 설계 (일관성)
- Key(directory)-Value 형태로 데이터 저장

### `API-server`

- 상태를 바꾸거나 조회
- etcd와 유일하게 통신하는 모듈
- REST API 형태로 제공
- 권한을 체크하여 적절한 권한이 없을 경우 요청을 차단, 관리자 요청 뿐 아니라 다양한 내부 모듈과 통신
- 수평으로 확장되도록 디자인(요청이 많기 때문에)

### `Scheduler`

- 새로 생성된 Pod을 감지하고 실행할 노드를 선택
- 노드의 현재 상태와 Pod의 요구사항을 체크

### `Controller`

- 끊임 없이 상태를 체크하고 원하는 상태를 유지

### `kubelet`

- 노드에 배포되는 Agent로, Pod 내의 container들이 실행되는 것을 직접적으로 관리

<br>

### Flow

1. user(yaml) 요청은 api-server로
2. api-server 요청은 etcd에 저장
3. etcd의 결과가 다시 api-server에 리턴됨
4. kube-scheduler는 새로 생성된 Pod를 감지하고 실행할 노드를 선택
5. pod가 어디에 뜰건지를 api-server가 인지하고 etcd와 통신
6. api-server -> kubelet(client) -> kubelet(server)
7. kubelet(worker node) -> container runtime
8. 컨테이너를 띄우고 kubelet -> api server(상태값)
9. 또 etcd와 통신 (상태값 업뎃)
10. kubelet이 주기적으로 Pod 상태 체크
