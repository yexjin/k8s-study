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
