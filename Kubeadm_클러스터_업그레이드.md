# Kubeadm Cluster Upgrade

[참고 문서](https://kubernetes.io/ko/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

<br>

```bash
sudo apt-get update
```

<br>

### 컨트롤 플레인 노드 업그레이드

---

1. kubeadm 업그레이드

```bash
# 1.27.x-00에서 x를 최신 패치 버전으로 바꾼다.
# 나는 1.27.1로 변경

apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.27.x-00 && \
apt-mark hold kubeadm
```

2. 업그레이드 계획 확인

```bash
kubeadm upgrade plan
```

`kubeadm upgrade` 는 이 노드에서 관리하는 인증서를 자동으로 갱신한다. 인증서 갱신을 하지 않으려면 `--certificate-renewal=false` 플래그를 사용할 수 있다.

3. 업그레이드할 버전 선택 후, 명령 실행

```bash
# 이 업그레이드를 위해 선택한 패치 버전으로 x를 바꾼다.
# 나는 1.27.1로 변경

sudo kubeadm upgrade apply v1.27.x
```

4. 컨트롤 플레인 노드 업그레이드 완료

![image](https://github.com/yexjin/k8s-study/assets/49095587/740a6635-172f-45a4-8baa-d16043cb435e)

<br>

### 워커노드 업그레이드

---

워커 노드의 업그레이드 절차는 워크로드를 실행하는 데 필요한 최소 용량을 보장하면서, 한 번에 하나의 노드 또는 한 번에 몇 개의 노드로 실행해야 한다.

```bash
kubectl get nodes

# NAME                  STATUS   ROLES           AGE     VERSION
# host-172-16-130-10    Ready    <none>          7d14h   v1.26.0
# host-172-16-130-121   Ready    control-plane   10d     v1.27.1
```

→ `host-172-16-130-10`을 `v1.27.1`로 업그레이드 해보자

```bash
sudo apt-upgrade
```

했더니 자동으로 v1.27.1로 최신 업뎃이 됨

```bash
kubectl get nodes

# NAME                  STATUS   ROLES           AGE     VERSION
# host-172-16-130-10    Ready    <none>          7d14h   v1.27.1
# host-172-16-130-121   Ready    control-plane   10d     v1.27.1
```

<br>

### `kubeadm upgrade apply` 작동 원리

---

다음을 수행.

- 클러스터가 업그레이드 가능한 상태인지 확인
  - API 서버에 접근 가능
  - 모든 노드가 `Ready` 상태
  - 컨트롤 플레인이 정상 동작
- 버전 차이 정책 적용
- 컨트롤 플레인 이미지가 사용가능한지 또는 머신으로 가져올 수 있는지 확인
- 컨트롤 플레인 컴포넌트 또는 롤백 중 하나라도 나타나지 않으면 업그레이드
- 새로운 CoreDNS와 kube-proxy 매니페스트를 적용하고 필요한 모든 RBAC 규칙이 생성되도록 한다.
- API 서버의 새 인증서와 키 파일을 작성하고 180일 후에 만료될 경우 이전 파일을 백업

**컨트롤 플레인 노드**에선?

- 클러스터에서 kubeadm `ClusterConfiguration` 을 가져온다.
- 선택적으로 kube-apiserver 인증서를 백업한다.
- 컨트롤 플레인 컴포넌트에 대한 정적 파드 매니페스트를 업그레이드한다.
- 이 노드의 kubelet 구성을 업그레이드한다.

**워커 노드**에선?

- 클러스터에서 kubeadm `ClusterConfiguration` 을 가져온다.
- 이 노드의 kubelet 구성을 업그레이드한다.
