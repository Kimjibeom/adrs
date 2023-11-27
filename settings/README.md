# 쿠버네티스 1.28 설치 (containerd)

# hostname 설정
$ hostnamectl set-hostname cluster1-master // 예시

$ sudo vi /etc/hosts  //  프라이빗 IPv4 주소 추가
172.31.26.177 cluster1-master
172.31.21.132 cluster1-worker1
172.31.26.163 cluster1-worker2

---
# br_netfilter 모듈을 로드
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# bridge taffic 보게 커널 파라메터 수정
# 필요한 sysctl 파라미터를 /etc/sysctl.d/conf 파일에 설정하면, 재부팅 후에도 값이 유지된다.
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 재부팅하지 않고 sysctl 파라미터 적용하기
sysctl --system
---

# HTTPS를 활용해 패키지 저장소에 접근하기 위해 패키지를 설치
apt update
apt install  apt-transport-https ca-certificates curl gnupg lsb-release -y

# Docker의 공식 GPG키를 시스템에 추가.
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Docker를 repository URL 등록 
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 새로운 저장소가 추가되었으므로, repository update
apt update

# containerd.io를 설치.
apt install containerd.io -y


# containerd config file 구성
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# 구성파일의 systemdCgroup = true로 수정
vi /etc/containerd/config.toml
...
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true


# containerd 재시작 후 서비스 동작 상태 확인
systemctl restart containerd
systemctl status containerd
<Ctrl>+<c>

ls -l  /run/containerd/containerd.sock

// kubeadm
# Update the apt package index and install packages needed to use the Kubernetes apt repository:
apt update
apt install -y apt-transport-https ca-certificates curl

# Download the Google Cloud public signing key:
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes apt repository:
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

---------------------------------------------------------------------------------------------
# control-plaine 컴포넌트 구성
kubeadm init --pod-network-cidr=192.168.0.0/16 --cri-socket unix:///var/run/containerd/containerd.sock
...
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!


# Kubectl을 명령 실행 허용하려면 kubeadm init 명령의 실행결과 나온 내용을 동작해야 함
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config


# CNI(컨테이너 네트워크 인터페이스) 기반 Pod 네트워크 추가 기능을 배포해야 Pod가 서로 통신할 수 있습니다. 네트워크를 설치하기 전에 클러스터 DNS(CoreDNS)가 시작되지 않습니다.
kubectl get nodes
NAME                 STATUS     ROLES           AGE     VERSION
ck8s-control-plane    NotReady  control-plane   6m49s   v1.26.0


# Pod 네트워크가 호스트 네트워크와 겹치지 않도록 주의해야함. 
# Calico 설치
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/custom-resources.yaml

# calico POD가 모두 running 될때까지  기다림
watch kubectl get pods -n calico-system
kubectl get pods -n calico-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-85666c5b94-927wg   1/1     Running   0          82s
calico-node-hcnp9                          1/1     Running   0          82s
calico-typha-f8845f557-cwgmx               1/1     Running   0          83s
csi-node-driver-l94kh                      2/2     Running   0          40s

# 다시 확인
kubectl get nodes
NAME                 STATUS   ROLES           AGE     VERSION
ck8s-control-plane   Ready    control-plane   6m49s   v1.26.0

# Calico Mode 변경
# https://github.com/projectcalico/calico - release 버전확인후 조절.
curl -L https://github.com/projectcalico/calico/releases/download/v3.24.1/calicoctl-linux-amd64 -o calicoctl
chmod +x calicoctl
mv calicoctl /usr/bin

calicoctl get ippool -o wide
NAME                  CIDR             NAT    IPIPMODE   VXLANMODE     DISABLED   DISABLEBGPEXPORT   SELECTOR   
default-ipv4-ippool   192.168.0.0/16   true   Never      CrossSubnet   false      false              all()

cat << END > ipipmode.yaml 
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  blockSize: 26
  cidr: 192.168.0.0/16
  ipipMode: Always
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
END

calicoctl apply -f ipipmode.yaml 
calicoctl get ippool -o wide
NAME                  CIDR             NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
default-ipv4-ippool   192.168.0.0/16   true   Always     Never       false      false              all()

-----------------------------------------------------------------------------------------

워커 노드에서 join 명령어 실행
kubeadm join 172.31.19.94:6443 --token 2to970.ubvwpqgu397ih928 --discovery-token-ca-cert-hash sha256:5e88db8ff06872cc9937d0ecedab7ab06297021c042f0db1e54b28821bc798cf
