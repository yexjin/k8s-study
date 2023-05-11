# kubeadm으로 클러스터 생성해보기

- `kubeadm` : 클러스터를 부트스트랩하는 명령
  - 마스터 노드에서 실행되며, k8s 클러스터의 초기화 및 마스터 노드 설정을 담당
  - 새로운 노드를 클러스터에 추가하고, 마스터 노드의 인증서를 생성하고, 네트워크를 구성하는 등의 작업 수행

![image](https://github.com/yexjin/k8s-study/assets/49095587/d115a3f5-dbb3-47b2-b043-705b13537de8)

- `kubelet` : 클러스터의 모든 머신에서 실행되는 파드와 컨테이너 시작과 같은 작업을 수행하는 컴포넌트
  - 클러스터 각 노드에서 실행되는 에이전트
  - 컨테이너 상태를 모니터링하고 이를 Kubernetes API 서버에 보고하여 상태를 유지
- `kubectl` : 클러스터와 통신하기 위한 커맨드 라인 유틸리티

<br>

## 실습 과정

---

1. Kakao i Cloud에 VM 3개 생성 (master-node, worker-node 2개)

   → 요 VM들은 k8s Node 들로 사용하기 위해 최소 **CPU:2, RAM:2** 이상이어야 함

2. 포트 OPEN

   > **Master**

   | 프로토콜 | 방향     | 포트 범위 | 용도                     | 사용 주체            |
   | -------- | -------- | --------- | ------------------------ | -------------------- |
   | TCP      | 인바운드 | 6443      | k8s API Server           | 전부                 |
   | TCP      | 인바운드 | 2379-2380 | etcd 서버 클라이언트 API | kube-apiserver, etcd |
   | TCP      | 인바운드 | 10250     | Kubelet API              | Self, 컨트롤 플레인  |
   | TCP      | 인바운드 | 10259     | kube-scheduler           | Self                 |
   | TCP      | 인바운드 | 10257     | kube-controller-manager  | Self                 |

   > **Worker**

   | 프로토콜 | 방향     | 포트 범위   | 용도            | 사용 주체           |
   | -------- | -------- | ----------- | --------------- | ------------------- |
   | TCP      | 인바운드 | 10250       | Kubelet API     | Self, 컨트롤 플레인 |
   | TCP      | 인바운드 | 30000-32767 | NodePort 서비스 | 전부                |

3. master-node에 공인 IP 연결 (실제로 모든 클러스터 노드들은 꽁꽁 숨겨두고, VPN으로 연결해야 함)
4. master-node에 SSH 접속

<br>

## Master Node

---

1.  IP 환경 변수 설정 (`vim /tmp/env.sh`)

    ```bash
    #!/bin/bash

    export PUBLIC_IP="${PUBLIC_IP}"
    export PRIVATE_IP="${PRIVATE_IP}"
    ```

2.  master node 스크립트 (`vim master.sh`)
    <details>
        <summary>Shell Script</summary>
    <pre>

    ```bash
    #!/bin/bash

    # Configure ENV
    . /tmp/env.sh

    # Install Https Packages
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl

    ##### Docker + CRI 세팅 #####
    # Add Docker GPG key
    sudo mkdir -m 0755 -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    # Add Docker repository
    echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update

    # Install Docker Engine
    sudo apt-get install -y containerd.io

    # Configure containerd (Container Runtime Tool)
    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml

    # Restart containerd
    sudo systemctl restart containerd

    ##### Kubernetes 세팅 #####
    # Add Kubernetes GPG key
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

    # Add Kubernetes repository
    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    # Update package
    sudo apt-get update

    # Install Kubernetes components
    version="1.26.0-00"
    sudo apt-get install -y kubelet=${version} kubeadm=${version} kubectl=${version}

    ##### Kernel 세팅 #####
    # Load kernel modules
    sudo modprobe overlay # 쿠버네티스 노드의 컨테이너 런타임과 함께 사용
    sudo modprobe br_netfilter # 쿠버네티스 CNI에서 브리지 네트워크와 함께 사용

    # Load kernel modules for Kubernetes : 컨테이너 통신을 가능하게 하며, 여러 kubernetes(서비스, 로드밸런싱, 파드간 통신) 기능을 구현
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    br_netfilter
    EOF

    # Set kernel parameters for Kubernetes : iptables을 사용하여 컨테이너 간 통신 처리
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward                 = 1
    EOF

    # Allow traffic for Kubernetes : 방화벽 규칙을 제거하여, k8s 클러스터의 노드 간에 통신할 수 있도록 허용
    sudo iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited
    sudo iptables -D FORWARD -j REJECT --reject-with icmp-host-prohibited

    # Apply kernel settings
    sudo sysctl --system

    ##### Kubernetes master node를 초기화하고 클러스터 생성 및 네트워크 설정 #####
    sudo kubeadm init --pod-network-cidr=192.168.0.0/16 \
        --apiserver-advertise-address=${PRIVATE_IP}\
        --control-plane-endpoint=${PRIVATE_IP}\
        --apiserver-cert-extra-sans=${PRIVATE_IP},${PUBLIC_IP}\

    ##### Kubernetes Cluster에 대한 접근 권한을 갖기 위한 설정 작업 #####
    mkdir -p ~/.kube
    sudo rm -f ~/.kube/config
    sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config
    sudo chown $(id -u):$(id -g) ~/.kube/config
    sudo cp -i /etc/kubernetes/admin.conf ./
    sudo cp -i /etc/kubernetes/admin.conf /tmp/admin.conf
    # 이후, kubectl 명령어를 사용할 때 ~/.kube/config 파일을 참조하여 쿠버네티스 클러스터에 대한 인증 및 인가를 수행

    # 이전에 정의된 iptable 규칙과 충돌하는 경우를 방지
    sudo iptables --flush
    sudo iptables -tnat --flush
    ```

    </pre>
    </details>
    <details>
        <summary>Contaiered?</summary>
    - Docker 프로젝트의 일부로 개발된 오픈 소스 컨테이너 런타임 (CRI : k8s에서 컨테이너 런타임과 통신하기 위한 표준 인터페이스를 제공) <br>
    - 컨테이너 이미지를 다운로드 하고, 파일 시스템을 생성하고, 컨테이너를 시작하고 중지하는 등의 기능을 수행
    </details>

3.  master node 생성

    ```bash
    bash master.sh
    ```

4.  생성의 결과로 마지막에

    ```bash
    kubeadm join 172.16.130.121:6443 --token 1bni8j.7yl71u35v8x0vngt \
    	--discovery-token-ca-cert-hash sha256:c93dd28747bd9f24a898379747d02f593e77e65ee10a6c7c5f5d7e40d1025b0c
    ```

    이라는 명령어가 화면에 출력되는데, 이는 **kubeadm을 사용해서 새로운 워커 노드를 Kubernetes 클러스터에 추가하는 명령어**이다. (어딘가 복사해두기 or `kubeadm token create --print-join-command` 명령어로 조회 가능)

    - `kubeadm join` 명령어 다음에 마스터 노드의 IP 주소와 포트번호 6443이 지정 → 마스터 노드의 Kubernetes API 서버가 리스닝하는 포트
    - `—token` 플래그는 새로 추가될 워커 노드를 인증하는데 사용되는 토큰 : 이 token 값은 마스터 노드에서 생성되며, 일반적으로 24시간 간 유효
    - `—discovery-token-ca-cert-hash` 플래그는 새로 추가될 워커노드가 마스터 노드와 신뢰관계를 맺을 수 있도록 하는 인증서 해시 값을 지정 : 이 값은 마스터 노드에서 생성되며, 새로운 워커 노드가 이 값을 사용하여 마스터 노드와의 TLS 인증서 교환을 수행

5.  모든 노드 확인

    ![image](https://user-images.githubusercontent.com/49095587/236661040-b38e341d-b361-4504-8247-a76412e1ab75.png)

    → 여기서 coredns Pod가 Pending 상태인 이유는, 노드에 적절한 리소스가 부족하여 스케줄링 되지 않았거나 파드가 필요로하는 스토리지 클래스나 네트워크 등의 리소스가 부족하기 때문이다. (CNI가 없어서 Coredns가 안뜨는 것임)

    → **Calico**를 설정하여 해결하면 된다.

    - **Calico**
      > - **Linux 및 Windows 에 대해 인터넷과 동일한 IP 네트워킹 원칙을 기반으로 Kubernetes 포드를 연결하기위한 네트워킹 및 네트워크 정책 솔루션**을 제공하는 오픈소스 프로젝트
      - k8s에서 **각 Node에 설치되어 각 Pod 간 네트워크 통신이 가능**하도록 도와주는 역할
        >

6.  Calico CNI 플러그인의 매니페스트 파일 다운로드

    ```bash
    curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml -O
    ```

    클러스터에 배포

    ```bash
    kubectl apply -f calico.yaml
    ```

    ![image](https://user-images.githubusercontent.com/49095587/236661052-c5583d9a-5034-4605-9875-d182022c6c1b.png)

    → 동작!

<br>

## Worker node

---

1. Bastion 호스트로 nginx-proxy를 통해 워커 노드에 SSH 접근
2. nginx-proxy-manager 관리 페이지로 접속하여, 각 워커노드의 22번 포트를 10001, 10002 포트로 등록

   ![image](https://user-images.githubusercontent.com/49095587/236661061-69bbd494-bc76-40f7-b355-68f12b70dcdf.png)

   ![image](https://user-images.githubusercontent.com/49095587/236661068-904c07dc-936c-4dd9-b8cb-778a3e168544.png)

3. 워커노드 접속

   ```bash
   ssh -i lena-key-27.pem ubuntu@210.109.63.173 -p 10001
   ```

4. worker node 스크립트 (`vim worker.sh`)
   <details>
       <summary>Shell Script</summary>
       
   ```bash
   #!/bin/bash

   sudo apt-get update
   sudo apt-get -y upgrade

   # Install Https Packages

   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl

   ##### Docker + CRI 세팅

   # Add Docker GPG key

   sudo mkdir -m 0755 -p /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

   # Add Docker repository

   echo \
   "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
   "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

   sudo apt-get update

   # Install Docker Engine

   sudo apt-get install -y containerd.io

   # Configure containerd (Container Runtime Tool)

   sudo mkdir -p /etc/containerd
   containerd config default | sudo tee /etc/containerd/config.toml

   # Restart containerd

   sudo systemctl restart containerd

   ##### Kubernetes 세팅

   # Add Kubernetes GPG key

   sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

   # Add Kubernetes repository

   echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

   # Update package

   sudo apt-get update

   # Install Kubernetes components

   version="1.26.0-00"
   sudo apt-get install -y kubelet=${version} kubeadm=${version} kubectl=${version}

   ##### Kernel 세팅

   # Load kernel modules

   sudo modprobe overlay
   sudo modprobe br_netfilter

   # Load kernel modules for Kubernetes : 컨테이너 통신을 가능하게 하며, 여러 kubernetes(서비스, 로드밸런싱, 파드간 통신) 기능을 구현

   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   br_netfilter
   EOF

   # Set kernel parameters for Kubernetes : iptables을 사용하여 컨테이너 간 통신 처리

   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   EOF

   # Allow traffic for Kubernetes : 방화벽 규칙을 제거하여, k8s 클러스터의 노드 간에 통신할 수 있도록 허용

   sudo iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited
   sudo iptables -D FORWARD -j REJECT --reject-with icmp-host-prohibited

   # Apply kernel settings

   sudo sysctl --system

   #이전에 정의된 iptable 규칙과 충돌하는 경우를 방지
   sudo iptables --flush
   sudo iptables -tnat --flush

   ```
   </details>

   ```

5. worker node 생성

   ```bash
   bash worker.sh
   ```

6. kubeadm으로 새로운 워커노드를 kubernetes 클러스터에 연결한다. (복사해뒀던 명령어!)

   ```bash
   sudo kubeadm join 172.16.130.121:6443 --token 1bni8j.7yl71u35v8x0vngt \
   	--discovery-token-ca-cert-hash sha256:c93dd28747bd9f24a898379747d02f593e77e65ee10a6c7c5f5d7e40d1025b0c
   ```

   `sudo kubeadm join` 명령어는 새로운 노드가 클러스터에 합류하도록 하는 명령어이다.

   이 명령어를 실행하면 쿠버네티스 컨트롤 플레인이 해당 노드를 인식하고 노드의 상태를 확인하고 최종적으로 노드를 승인한다. 그 후, 쿠버네티스 컨트롤 플레인은 새로운 노드에 필요한 구성요소를 다운로드하고 설치하고 구성 → 시간 소요

7. worker node 생성 끝! (나머지 하나도 똑같이 진행하면 된다.)

<br><br>
kubectl은 기본적으로 마스터 노드에만 설치되어 있으며, 일반적으로 kubernetes 워커 노드에는 설치되어 있지 않다. 워커 노드가 애플리케이션을 실행하는 역할을 수행하고, 마스터 노드에서 클러스터를 제어하고 관리하는데 사용되기 때문!

그치만, 워커 노드에서 kubectl을 사용해야하는 경우, 로컬 머신에 kubectl을 설치하고 마스터 노드에서 발급하는 kubeconfig을 사용하여 원격으로 마스터 노드에서 kubernetes 클러스터를 관리할 수 있다.
