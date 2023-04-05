# PV의 Dynamic Provisioning과 Storage Class

관리자는 **실제 스토리지를 미리 프로비저닝**해두어야 하는데, k8s의 PV의 동적 프로비저닝을 통해 자동으로 수행할 수 있다.

클러스터 관리자가 PV를 생성하는 대신 **PV 프로비저너를 배포하고 사용자가 선택 가능한 PV 타입을 하나 이상의 스토리지 클래스 오브젝트로 정의**할 수 있다.

→ 사용자가 **PVC에서 스토리지 클래스를 참조하면 프로비저너가 Persistent Storage를 프로비저닝할 때 이를 처리**한다.

PVC를 정의할 때, 특정 Storage Class를 요청할 수 있다.

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

<br>

**Storage Class 종류**

1. `standard` : 기본적인 스토리지 클래스, 임의의 PV를 생성
2. `fast` : SSD, NVMe 기반의 빠른 입출력 속도를 가진 스토리지를 사용
3. `gp2`, `io1`, `sc1`, `st1`: AWS EBS 용
4. `local` : 로컬 디스크에 바로 마운트 되는 스토리지 클래스로서, 로컬 스토리지를 사용할 수 있다.
5. 위의 코드에서 `csi-cinder-sc-delete` 는 오직 RWO volume에서 사용할 수 있다.
6. `ssd`: 고성능 SSD 스토리지를 사용하는 스토리지 클래스로서, 빠른 입출력 성능을 제공합니다.
7. `performance`: 최고 수준의 성능을 제공하는 스토리지 클래스로서, 고성능 SSD 스토리지와 최대 IOPS를 제공합니다.
8. `backup`: 백업용 스토리지 클래스로서, 데이터 보존 및 백업에 적합한 스토리지를 제공합니다.
9. `archive`: 아카이브용 스토리지 클래스로서, 저렴한 가격과 높은 내구성을 제공하는 스토리지를 제공합니다.
10. `snapshot`: 스냅샷용 스토리지 클래스로서, 스냅샷 생성 및 관리에 적합한 스토리지를 제공합니다.
11. `gpu`: GPU 가속 스토리지 클래스로서, AI/ML 작업에 적합한 고성능 스토리지를 제공합니다
