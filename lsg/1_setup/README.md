# k8s 셋업

MobaXterm과 VitualBox를 통해 쿠버네티스 세팅을 진행해 보았다.
인프런 강좌인 [대세는 쿠버네티스](https://www.inflearn.com/course/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EA%B8%B0%EC%B4%88/dashboard)를 참고해 진행하였고 마스터 파드 1개, 2개의 슬레이브로 구성했다.

### 작업 진행순서
- 필요한 파일 다운로드
- vm 세팅
- 쿠버네티스 설정


### 1. 필요한 파일 다운로드
 - [VitualBox](https://www.virtualbox.org/wiki/Downloads)
 - [VitualBox - Mac](https://www.virtualbox.org/wiki/Mac%20OS%20X%20build%20instructions)
 - [centos 이미지](http://isoredirect.centos.org/centos/7/isos/x86_64/)
 - [원격 접속 MobaXterm](https://mobaxterm.mobatek.net/)

 ### 2. vm 세팅
 |순서| 설명|
 |--|--|
 |1| 머신 > 새로만들기 클릭
 |2| 이름 : k8s-master, 종류: Linux, 버전: Other Linux(64-bit)
 |3| 메모리 : 4096 MB
 |4| 하드디스크 : 지금 새 가상 하드 디스크 만들기 (VDI:VirtualBox 디크스 이미지, 동적할당, 150GB)
 |5| 프로세서 개수 : CPU 2개
 |6| 저장소 설정 -> 컨트롤러:IDE 하위에 있는 광학드라이브 클릭 > CentOS 이미지 선택 후 확인
 |7| 네트 워크 설정 -> VM 선택 후 설정 버튼 클릭 > 네트워크 > 어댑터 1 탭 > 다음에 연결됨 [어댑터에 브리지] 선택

#### 2-1. CentOS 설치 
1. Test this media & install CentOS 7
2. Language : 한국어 
3. Disk 설정 [시스템 > 설치 대상]
   - [기타 저장소 옵션 > 파티션 설정] 파티션을 설정합니다. [체크] 후 [완료]
   - 새로운 CentOS 설치 > 여기를 클릭하여 자동으로 생성합니다. [클릭]
   - /home [클릭] 후 용량 5.12 GiB로 변경 [설정 업데이트 클릭]
   - / [클릭] 후 140 GiB 변경 후 [설정 업데이트 클릭]
   - [완료], [변경 사항 적용]
4. 네트워크 설정 [시스템 > 네트워크 및 호스트명 설정]
   - 호스트 이름: k8s-master [적용]
   - 이더넷 [설정]
      [일반] 탭
      - 사용 가능하면 자동으로 이 네트워크에 연결 [체크]
      [IPv4 설정] 탭
      - 방식: 수동으로 선택, 
      - [Add] -> 주소: 192.168.0.30, 넷마스크 : 255.255.255.0, 게이트웨이: 192.168.0.1, DNS 서버 : 8.8.8.8
      - [저장][완료]   
 5. 설치시작
6. [설정 > 사용자 설정] ROOT 암호 설정 
7. 설치 완료 후 [재부팅]


#### 2-2 SELinux설정

|순서|명령어 | 설명|
|--|--|--|
|1|setenforce 0| 쿠버네티스가 Pod Network에 필요한 호스트 파일 시스템에 액세스가 가능하도록 하기 위해서 필요한 설정
|2|sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config|리부팅시 다시 원복되기 때문에 아래 명령을 통해서 영구적으로 변경
|3|systemctl stop firewalld && systemctl disable firewalld | firewalld 비활성화
|4|systemctl stop NetworkManager && systemctl disable NetworkManager|NetworkManager 비활성화
|5|swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab |Swap 비활성화
|6|cat <<EOF >  /etc/sysctl.d/k8s.conf<br/>net.bridge.bridge-nf-call-ip6tables = 1<br/>net.bridge.bridge-nf-call-iptables = 1<br/>EOF<br/>sysctl --system<br/>| RHEL이나 CentOS7 사용시 iptables가 무시되서 트래픽이 잘못 라우팅되는 문제가 발생때문에 추가
|7| cat <<EOF > /etc/yum.repos.d/kubernetes.repo<br/>[kubernetes]<br/>name=Kubernetes<br/>baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64<br/>enabled=1<br/>gpgcheck=1<br/>repo_gpgcheck=1<br/>gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpghttps://packages.cloud.google.com/yum/doc/rpm-package-key.gpg<br/>exclude=kubelet kubeadm kubectl<br/>EOF | 쿠버네티스 YUM Repository 설정
|8| yum -y update | Centos Update
|9| cat << EOF >> /etc/hosts<br/> 192.168.0.30 k8s-master<br/> 192.168.0.31<br/> k8s-node1 192.168.0.32 k8s-node2 <br/>EOF | Hosts 등록
|10|yum install -y yum-utils device-mapper-persistent-data lvm2 |도커 설치 전에 필요한 패키지 설치
|11|yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo|도커 설치를 위한 저장소 를 설정
|12|yum update -y && yum install -y containerd.io-1.2.13 docker-ce-19.03.11 docker-ce-cli-19.03.11| 도커 패키지 설치
|13|mkdir /etc/docker<br/>cat > /etc/docker/daemon.json <<EOF<br/>{<br/>  "exec-opts": ["native.cgroupdriver=systemd"],<br/>  "log-driver": "json-file",<br/>  "log-opts": {<br/>    "max-size": "100m"<br/>  },<br/>  "storage-driver": "overlay2",<br/>  "storage-opts": [<br/>    "overlay2.override_kernel_check=true"<br/>  ]<br/>}<br/>EOF<br/><br/>mkdir -p /etc/systemd/system/docker.service.d| 도커 설치 후 설정
|14|yum install -y --disableexcludes=kubernetes kubeadm-1.19.4-0.x86_64 kubectl-1.19.4-0.x86_64 kubelet-1.19.4-0.x86_64|Kubernetes 설치

### 3. 쿠버네티스 설치 및 설정
|순서|명령어|설명|비고
|--|--|--|--|
|1|systemctl daemon-reload|도커실행
|2|systemctl enable --now docker| 도커 실행
|3|systemctl enable --now kubelet|쿠버네티스 실행
|4|kubeadm init --pod-network-cidr=20.96.0.0/12 --apiserver-advertise-address=192.168.0.30| 쿠버네티스 초기화 | 실행후 kubeadm join 이후 내용 복사
|5|mkdir -p $HOME/.kube<br/>sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config<br/>sudo chown $(id -u):$(id -g) $HOME/.kube/config<br/> | kubectl실행하기 위한 환경 변수
|6|yum install bash-completion -y<br/>source <(kubectl completion bash)<br/>echo "source <(kubectl completion bash)" >> ~/.bashrc<br/>| Kubectl 자동완성 기능 설치

#### 3-1. 하위 노드에서 마스터와 연결
- systemctl daemon-reload
- systemctl enable --now docker
- systemctl enable --now kubelet
- 위에서 복사한 kubeadm join 이후 내용 붙여넣기 & 실행
