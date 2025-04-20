# 목표

## 쿠버네티스를 실습하고, 서비스에 적용시키기 위한 환경 세팅하기

쿠버네티스 이론만 조금 공부해봤지, 실습하면서 공부한 적이 없어서 이번 기회에 공부 환경을 세팅해보기로 결정!

기존에 클라우드 환경에서 호스팅하던 서비스에 쿠버네티스 적용시킬 수 있도록 연습하는 것이 목표

일단 해당 글을 보고 쿠버테티스 설치 시도

[Setup Homelab Kubernetes Cluster](https://cavecafe.medium.com/setup-homelab-kubernetes-cluster-cfc3acd4dca5)

우선 도커 설치 필요 → 내 서버에는 이미 설치되어 있음

kubernetes repository 버전 문제가 발생하여 수정
최근 패키지 저장소의 경로가 바뀌었으니 유의!

```bash
# 기존 버전 삭제
sudo rm /etc/apt/sources.list.d/kubernetes.list
sudo rm /etc/apt/sources.list.d/archive_uri-http_apt_kubernetes_io_-noble.list

# 키링 디렉토리 생성 (필요 시)
sudo mkdir -p -m 755 /etc/apt/keyrings

# Kubernetes 패키지 저장소의 공개 키 다운로드
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 새로운 저장소 추가
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubeadm kubelet kubectl
```

- <설치한 패키지>
    1. kubeadm: 클러스터를 초기화하고 관리하는 도구.
        - 클러스터를 "만드는" 도구로, 초기 설정과 노드 관리를 담당
        - 워커 노드를 추가할 때도 사용
        - kubeadm init은 마스터 노드의 kubelet을 시작하고 컨트롤 플레인 pod를 실행함
    2. kubectl: Kubernetes 클러스터와 상호작용하는 CLI.
        - 클러스터를 "사용하는" 도구로, 관리자와 개발자가 리소스를 제어
        - 리소스 관리 : pod, deployment, service 등 쿠버네티스 오브젝트를 생성, 수정, 삭제
        - 상태 확인 : 클러스터의 노드, pod, 이벤트 등을 조회
        - 디버깅 : 로그 확인, pod 내 명령 실행 등 문제를 진단
        - 배포: yaml/json 파일을 적용해 애플리케이션을 배포
        - API 클라이언트로 동작하며, kubernetes api 서버와 통신함
    3. kubelet: 각 노드에서 컨테이너를 실행하는 에이전트.
        - 노드의 핵심!
        - 클러스터를 "운영하는" 도구로, 노드에서 실제 작업을 수행
        - pod 실행 : api 서버로부터 pod 사양을 받아 컨테이너 런타임을 통해 pod를 생성하고 **실행**
        - 상태 보고 : 노드와 pod의 상태를 api 서버에 주기적으 보고함
        - 헬스 체크 : pod의 liveness/readiness 프로브를 실행해 컨테이너 상태를 모니터링 함
        - 노드 관리 : 노드의 리소스 할당 및 네트워크 설정을 조정함
        - kubeadm으로 설치되지만, 별도로 실행되며 systemd 서비스로 관리됨
        - api 서버와 직접 통신하며, 클러스터의 일부로 동작함
    4. apt-mark hold: 패키지가 자동으로 업데이트되지 않도록 버전을 고정

## control plane 노드 초기화

`--control-plane-endpoint` 옵션을 사용하여 모든 컨트롤 플레인 노드에 대한 공통 엔드포인트를 설정해주면 HA로 사용 가능

- 단일 control plane 클러스터로 설정하면 HA로 업그레이드는 불가능
- 외부 접근, 모드 간 통신을 위해 kubeadm에서 네트워크 인터페이스에 사용 가능한 ip를 지정 →

```bash
jiwon@jiwon:~$ ip route show
default via 172.30.1.254 dev wlp2s0 proto dhcp src 172.30.1.20 metric 600 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
172.30.1.0/24 dev wlp2s0 proto kernel scope link src 172.30.1.20 metric 600 
```

이론상 저 172.17.0.1 docker0 인터페이스도 사용할 수 있지만 linkdown이므로 172.30.1.20의 wlp2s0 무선 인터페이스를 사용

```bash
# 단일 마스터 노드
sudo kubeadm init --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=172.30.1.20

# 다중 마스터 노드
sudo kubeadm init --control-plane-endpoint="k8s-lb.example.com" --apiserver-advertise-address=172.30.1.20 --upload-certs --pod-network-cidr=10.10.0.0/16

sudo kubeadm join k8s-lb.example.com:6443 --token <token> --discovery-token-ca-cert-hash <hash> --control-plane --certificate-key <key> --apiserver-advertise-address=172.30.1.21
```

⚠️ *다중 마스터 노드 사용을 위해 임의로 지정한 “k8x-lb.example.com” 의 dns 오류 발생*

```bash
sudo nano /etc/hosts

# 추가
172.30.1.20 k8s-lb.example.com
```

- join으로 워커 노드 추가할때는 본인의 인증 값이 포함된 명령어는 아래의 명령어로 확인 하여 실행(해당 ip 접속하여)

```bash
sudo kubeadm token create --print-join-command
```

단일, 다중 마스터 노드 설정을 둘 다 시도해보고,
지금은 단일 마스터 노드 환경에서 init으로 메인 서버에 마스터 노드만 생성된 상태
워커 노드를 추가하기 위해 가상화 플랫폼인 Proxmox로 서버 더 띄우고 CNI 설정 해줘야 함