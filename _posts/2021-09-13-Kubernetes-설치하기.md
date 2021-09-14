---
layout: post
title: Kubernetes - 쿠버네티스 설치하기
categories: [TIL-Kubernetes]
comments: true
---

새로운 프로젝트를 진행하며 다량의 소셜 데이터를 분석하고, 저장, 변환하는 등 일련의 과정에 필요한 데이터 서버가 필요했다. 기존에 Elasticsearch 클러스터를 도커 컨테이너로 구성해서 운영중이였는데, 데이터 양이 점점 쌓일 경우 확장이 필요한데 컨테이너 환경에서 무중단 스케일 아웃이 어려운 편이고 물리서버 자체도 임시로 bare-metal 서버를 사용중이였기 때문에 스케일 업도 쉽지않은 상황이었다.

때마침, 새로운 IDC센터의 서버 구축이 완료되어 추가 서버자원 사용이 가능해졌고 냉큼 6대의 VM을 할당받고 쿠버네티스 클러스터를 구성하게 되었다.

## 설치 환경 

6개의 VM을 이용
- OS  : Ubuntu 18.04
- CPU : 8Core
- RAM : 16GB

3개의 마스터(컨트롤 플레인) 노드와 3개의 워커노드로 구성할 예정

## Install with Kubeadm

### 준비물

Linux(Debian, RedHat, 패키지매니저 없는 버전) 서버만 대상으로 함.

- 2GB 이상의 RAM
- 2코어 이상 CPU
- 클러스터를 구성할 전체 노드의 네트워크 연결 가능성
- 모든 노드에 대해 고유한 호스트 네임, MAC 주소, product_uuid
- 특정 포트 개방(일단 방화벽을 비활성화 후 설치하는 것을 추천)
- swap 비활성화 (`swapoff -a`)

### 설치 과정

#### 1. MAC 주소, product_uuid 고유성 확인

- MAC 주소 확인 : `ip link` 또는 `ifconfig -a`

  ```bash
  ip link show

  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000 link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
  2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000 link/ether 52:54:00:87:9e:e6 brd ff:ff:ff:ff:ff:ff # 52:54: 로 시작하는 부분이 Network 인터페이스의 MAC 주소
  ```

- product_uuid 확인 : `sudo cat /sys/class/dmi/id/product_uuid`
  > (원문 참조) 일부 가상 머신은 동일한 값을 가질 수 있지만 하드웨어 장치는 고유한 주소를 가질 가능성이 높다. 쿠버네티스는 이러한 값을 사용하여 클러스터의 노드를 고유하게 식별한다. 이러한 값이 각 노드에 고유하지 않으면 설치 프로세스가 실패할 수 있다.

#### 2. iptables가 브리지된 트래픽 확인할 수 있도록 하기

- `lsmod | grep br_netfilter` 명령어를 통해 `br_netfilter` 모듈이 로드되었는지 확인
- 명시적으로 로드하려면 `sudo modprobe br_netfilter` 를 실행

리눅스 노드의 iptables가 브리지된 트래픽을 올바르게 보기 위한 요구 사항으로, sysctl 구성에서 net.bridge.bridge-nf-call-iptables 가 1로 설정되어 있는지 확인해야 한다. 

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

#### 3. 필수 포트 확인

**컨트롤 플레인 노드**

|프로토콜|포트범위|목적|사용자|
|---|---|---|---|
|TCP|6443|쿠버네티스 API 서버|모두|
|TCP|2379-2380|etcd 서버 클라이언트 API|kube-apiserver, etcd|
|TCP|10250|kubelet API|자체, 컨트롤 플레인|
|TCP|10251|kube-scheduler|자체|
|TCP|10252|kube-controller-manager|자체|

**워커 노드**

|프로토콜|포트범위|목적|사용자|
|---|---|---|---|
|TCP|10250|kubelet API|자체, 컨트롤 플레인|
|TCP|30000-32767|NodePort 서비스|모두|

#### 4. 컨테이너 런타임 설치

쿠버네티스는 컨테이너 런타임 인터페이스(CRI)를 사용하여 사용자가 선택한 컨테이너와 소통함.

컨테이너 런타임 종류
- docker
- containerd
- CRI-O

각 컨테이너 런타임은 공식 홈페이지를 통해 설치 (특별한 일이 없다면 도커를 추천)
> **CRI** : Kubelet과 컨테이너 런타임을 통합시키는 API

#### 5. cgroup 드라이버 구성 설정

cgroup(control groups의 약자)는 프로세스들의 자원 사용을 제한하고 격리시키는 리눅스 커널 기능.
보안과 자원효율을 위해 컨테이너 기술의 기반이 되는 요소.

cgroup에 대한 자세한 내용은 [cgroup 소개](https://access.redhat.com/documentation/ko-kr/red_hat_enterprise_linux/6/html/resource_management_guide/ch01) 글을 읽어보는 것으로 대체하도록 함.

cgroup 별로 관리자가 하나씩 할당되며 같은 그룹 내에 할당된 리소스를 단순화하고 사용가능 / 사용중 인 리소스를 일관성있게 확인이 가능하지만, 관리자가 두 개인 경우 리소스도 두 개의 관점에서 보게됨.

대부분의 서비스가 kubernetes 클러스터 위에서 동작하는 것을 감안했을때 리눅스 init 시스템이 사용하는 드라이버와 docker, kubelet 의 cgroup 드라이버를 맞춰주는 것이 리소스 관리적인 면에서 효율적임.

kubernetes의 권장 cgroup 설정은 `systemd`이기 때문에 docker를 컨테이너 런타임으로 설정한 경우 cgroup을 동일하게 맞춰주어야 한다.

```bash
# docker daemon.json 설정 변경 (exec-opts 속성에 주목 / 나머지 설정은 사용자 환경에 맞게..)
cat <<EOF | sudo tee /etc/docker/daemon.json {
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# 데몬 리로드 및 도커 재시작
sudo systemctl daemon-reload
sudo systemctl restart docker

# docker cgroup 확인
docker info
```

kubernetes도 마찬가지로 `cgroupDriver` 속성의 값을 systemd로 바꿔주어야 한다.
kubeadm 설정을 커스터마이징 할 수 있는 kubead-config 파일을 작성하고 설정한 내용대로 클러스터를 구성하도록 명령하면 된다. 아래는 cgroupDriver를 변경하는 가장 기본적인 예시.

```yaml
# kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.21.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

설정파일을 저장한 후 해당 설정을 기준으로 `kubeadm init` 명령어를 아래와 같이 실행

```bash
kubeadm init --config kubeadm-config.yaml
```

> kubeadm v1.22 버전부터는 kubeadm 설정에 `KubeletConfiguration` 아래의 `cgroupDriver` 필드를 지정하지 않을 경우 기본으로 `systemd`가 되기 때문에 cgroup을 `systemd`를 사용할 예정이라면 따로 지정할 필요가 없음

cgroupfs를 사용할 경우 [공식문서](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/) 확인

#### 6. kubead, kubelet, kubectl 설치

모든 노드(머신, VM)에 아래 패키지들을 설치

- kubeadm: 클러스터를 부트스트래핑하기 위한 패키지
- kubelet: 클러스터의 모든 머신에서 컨트롤 플레인과 통신하는 어플리케이션. 노드에 작업을 요청할 경우 kubelet을 통해 작업을 실행함.
- kubectl: 클러스터와 통신하기 위한 커맨드라인 유틸

kubeadm vs kubelet + kubectl 버전과 관련된 이슈가 있기 때문에 업그레이드 또는 기존 kubelet, kubectl 위에 kubeadm 설치할 경우 [버전 및 버전차이](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#version-skew-policy) 정책을 꼭 확인해야함. (새로 설치하는 경우는 무방할듯)

추천 설치방법은 OS 패키지 관리자를 통해 설치하는 방법. 그 외의 방법은 공식문서 참고.

1. 패키지 관리자 업데이트 및 기본 패키지 설치

    ```bash
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    ```

2. 구글 클라우드의 공개 키 다운로드

    ```bash
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    ```

3. 쿠버네티스 레포지토리 추가

    ```bash
    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```
    이 과정에서 `Conflicting values set for option Signed-By regarding source....` 로 시작하는 에러가 발생하면 도커나 쿠버네티스와 관련해서 기존에 추가되어있던 레포지토리가 존재하고 이 버전이 충돌나서 그럴 확률이 높음.

    Ubuntu 기준 /etc/apt/sources.list.d/ 아래에 있는 등록된 레포들을 확인하면서 주석처리 하거나 파일 제거 후 재등록 해보면서 해결

4. 패키지 관리자를 다시 업데이트하고 kubelet, kubead, kubectl 설치 및 버전 고정

    ```bash
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

## 클러스터 구성하기

설치된 kubeadm을 이용하여 가장 처음에 언급했던 3대의 마스터 노드와 3대의 워커노드로 클러스터를 구성한다.

![](/img/kube/kubernetes-cluster2.png)

### HA 구성 관련 참고사항

고가용성을 위해서는 여러 대의 노드(마스터, 워커 둘 다)가 존재해야하고, 로드밸런서를 통해 부하분산을 해줘야한다. 
OnPremise 환경에서는 로드밸런서를 따로 준비해야하므로, [Metal LB](https://metallb.universe.tf/installation/)같은 OnPremise용 쿠버네티스 로드밸런서를 준비해야 한다.

> 여기서 말하는 로드밸런서는 일반 사용자(고객)에게 제공되는 웹 서비스를 위한 로드밸런서가 아니라 마스터 노드로 몰리는 워커 노드의 API 요청을 분산하기 위한 로드 밸런서이다. 
> (아래 이미지 참고)

![](/img/kube/kubeadm-ha-topology-stacked-etcd.svg)


Kubernetes 마스터 노드의 HA 구성도 etcd 를 같은 서버에 두냐 분리하냐로 구분할 수 있음. 위의 이미지는 etcd를 같은 서버에 둔 경우의 아키텍쳐(stacked etcd / pulse lake에 구성된 클러스터도 해당 방식으로 구성).

자세한 사항은 [공식문서](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/ha-topology/) 참고.


### 클러스터에 조인하기

```bash
# Control-plane(master)
You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

kubeadm join <load balancer IP>:<load balancer port> --token <token> \
      --discovery-token-ca-cert-hash 
      <ca-cert-hash> \
      --control-plane

# Worker nodes
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <load balancer IP>:<load balancer port> --token <token> \
      --discovery-token-ca-cert-hash 
      <ca-cert-hash> 
```

실제로 할때는 처음 `kubeadm init` 을 한 마스터 노드에서 certificate key를 발급하고 해당 키를 이용하여 추가 옵션을 넣어주어야 제대로 동작했음.

```bash
# 첫 번째 마스터 노드(kubeadm init)
$ sudo kubeadm init phase upload-certs --upload-certs 
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
<certificate key>

# 노드 Join
kubeadm join <load balancer IP>:<load balancer port> --token <token> \
      --discovery-token-ca-cert-hash 
      <ca-cert-hash> \
      --control-plane # control node 대상에만 옵션 추가
      --certificate-key <위에 나온 certificate key>
```


### Kubeadm 클러스터 완전 초기화

```bash
# Docker 초기화 (citypulse 서버 경우에는 /var/lib/docker 기본 마운트 폴더를 커스텀하여 조금 다름)
docker rm -f `docker ps -aq`
docker volume rm `docker volume ls -q`
sudo umount /var/lib/docker/volumes
sudo rm -rf /var/lib/docker/
sudo systemctl restart docker

# kubeadm 초기화
sudo kubeadm reset
sudo systemctl restart kubelet
```

## Install with Kubespray

Ansible에 의존성을 두고있기 때문에 IaC(Infrastructure as Code)에 대한 이해도(적어도 Ansible과 플레이북) 가 필요하고 결정적으로 설치 가이드라인을 따라하다 실패했기 때문에, 나중에 다시 정리하는 것으로 함. 

**Kubespray 참고**
- https://kubernetes.io/ko/docs/setup/production-environment/tools/kubespray/
- https://kubespray.io/
- https://lapee79.github.io/article/setup-production-ready-kubernetes-on-baremetal-with-kubespray/
- https://schoolofdevops.github.io/ultimate-kubernetes-bootcamp/cluster_setup_kubespray/


## 참고
- [kubeadm 설치 공식문서](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [kubeadm ha 클러스터 구성 블로그](https://fliedcat.tistory.com/170)
- [kubedex](https://kubedex.com/kubernetes-network-plugins/)
- [kubernetes cni 관련 블로그](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=alice_k106&logNo=221574467441)
- [kubeadm 클러스터 완전 초기화](https://wookiist.dev/143)

