서버 설치, 관련 패키지&드라이버 설치

# 1. 서버: DGX OS 6 설치

1. 사전 확인: nvidia-container-runtime이 [가능한 OS 여부 확인](https://nvidia.github.io/nvidia-container-runtime/)
    - 현재 DGX OS 6가 Ubuntu 22.04 기반
1. 필수 작업: IP 확인, 포맷, 우분투 설치, SSH 설정, DGX OS 6 부팅 디스크 제작작
1. 네트워크 문제 시: openvswitch-switch 설치, Netplan 수정 후 적용
1. 방화벽 설정: SSH(22), HTTP(80), HTTPS(443), Jupyter(8888:8889/tcp) 관련 포트 허용
```bash
sudo mkdir /home/ubuntu
sudo cp /etc/skel/.* /home/ubuntu/
sudo chown ubuntu:ubuntu /home/ubuntu/.* 
sudo passwd ubuntu 

#예비 로그인
sudo adduser 새사용자이름
sudo usermod -aG sudo 새사용자이름

# 네트워크 안 되면
sudo apt install -y openvswitch-switch
vim /etc/netplan/01-network-manager-all.yaml

### 아래 참조
sudo netplan apply

# 방화벽
sudo ufw allow 22 comment 'ssh'
sudo ufw allow 80 comment 'http'
sudo ufw allow 443 comment 'https'
sudo ufw allow 8888:8889/tcp comment 'jupyter'
```

```yml
network:
  ethernets:
    enp2s0f0:
      dhcp4: no
      addresses:
        - 172.30.1.62/24
      routes:
        - to: default
          via: 172.30.1.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
  version: 2
```
---
# 2. 도커 설치
- 기존 패키지 제거: docker.io, docker-doc, docker-compose, podman-docker, containerd, runc 제거
- Docker repo 설정: 공식 GPG 키 및 repository 추가 후 apt-update 실행
- 설치: docker-ce, docker-ce-cli, containerd.io, docker-buildx-plugin, docker-compose-plugin 설치
## uninstall
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```
## repo 설정
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

## 설치
```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

# 3. NVIDIA Container Toolkit
- repo 설정: NVIDIA Container Toolkit GPG 키 및 repository 추가, experimental 항목 주석 해제 후 apt-get 업데이트
- 설치: nvidia-driver-570 및 nvidia-container-toolkit 설치
- 도커 설정: nvidia-ctk로 도커 런타임 구성, 도커 재시작, Jupyter 작업공간 디렉토리 및 권한 설정
## repo 설정
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sudo sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
```
## 설치
```bash
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt install -y nvidia-driver-570
# sudo ubuntu-drivers autoinstall
sudo apt-get install -y nvidia-container-toolkit
```

## 도커 설정
```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
sudo mkdir -p /workspace/8888 /workspace/8889
sudo chown -R $USER:$USER /workspace
```

## NVML 에러 수정
#nvidia-smi<br>
Failed to initialize NVML: Unknown Error
```bash
sudo vim /etc/nvidia-container-runtime/config.toml
```
```toml
...
no-cgroups = false # 주석 해제
...
```

or/and
```bash
sudo vim /etc/docker/daemon.json 
```
```json
{  
   "runtimes": {  
       "nvidia": {  
           "args": [],  
           "path": "nvidia-container-runtime"  
       }  
   },  
   "exec-opts": ["native.cgroupdriver=cgroupfs"] # 추가
} 
```

리눅스 커널 자동 업데이트 & 종료 끄기
`sudo systemctl disable unattended-upgrades`
`sudo systemctl stop unattended-upgrades`
`sudo nano /etc/apt/apt.conf.d/50unattended-upgrades`
```bash
Unattended-Upgrade::Automatic-Reboot "false";
```

재부팅
```
sudo shutdown -r now
```



---

# 계정 사라지는 문제
**설정 후 테스트 아이디 생성!!**

아래는 **이번 DGX Station (A100×8) 로그인 장애에 대한 원인·조치·해결 명령어를 정리한 Markdown 문서**입니다.
보고/공유/위키 업로드용으로 바로 사용하셔도 됩니다.

---

# DGX Station A100 (DGX OS 6)

## 로그인 실패 이슈 원인 및 해결 가이드

---

## 1. 장애 증상 요약

* DGX OS 6 설치 직후 **첫 로그인은 정상**
* 재부팅 이후 **동일 계정으로 로그인 실패 (password incorrect)**
* Recovery mode 진입 시 **iSCSI 로딩 단계에서 무한 대기**
* GRUB까지는 진입 가능
* `/home` 디렉터리 비어 있음
* `reboot` 시 `failed to talk to daemon` 메시지 발생

---

## 2. 근본 원인 분석

### 2.1 직접 원인

1. **iSCSI 서비스가 부팅(initramfs) 단계에서 활성화**

   * 실제 iSCSI 타겟 없음
   * systemd / initramfs 단계에서 타임아웃 없이 대기
   * recovery mode 포함 부팅 경로 차단

2. **RAID 초기화 타이밍 문제**

   * mdadm / LVM 활성화 지연
   * `/home` 파티션 미마운트
   * PAM 인증 이후 홈 디렉터리 접근 실패 → 로그인 실패로 인식

3. **응급 쉘(init=/bin/bash) 진입 상태**

   * systemd(PID 1) 미기동
   * `reboot`, `shutdown` 명령 정상 동작 불가

---

### 2.2 왜 “패스워드 오류”로 보였는가

Linux 로그인 흐름:

```
비밀번호 인증 성공
 → 홈 디렉터리 접근 시도
   → /home 미존재 또는 접근 불가
     → Login incorrect 반환
```

즉, **실제 원인은 인증이 아니라 스토리지 마운트 실패**.

---

## 3. 해결 절차 (단계별)

### 3.1 GRUB에서 우회 부팅

GRUB 메뉴 → 부팅 항목 선택 → `e`

`linux`로 시작하는 줄 끝에 추가:

```text
rd.iscsi=0 systemd.mask=iscsi.service systemd.mask=iscsid.service init=/bin/bash
```

부팅 실행:

* `Ctrl + X` 또는 `F10`

---

### 3.2 응급 쉘 진입 후 초기 설정

```bash
# 루트 파일시스템 RW 마운트
mount -o remount,rw /
```

---

### 3.3 iSCSI 영구 비활성화 (핵심)

```bash
systemctl stop iscsi iscsid
systemctl disable iscsi iscsid
systemctl mask iscsi iscsid
systemctl mask open-iscsi
```

확인:

```bash
systemctl status iscsi
```

정상 상태:

```
Loaded: masked
Active: inactive (dead)
```

---

### 3.4 RAID / LVM 활성화

```bash
mdadm --assemble --scan
vgchange -ay
```

상태 확인:

```bash
lsblk
cat /proc/mdstat
```

---

### 3.5 mdadm 설정 고정

```bash
mdadm --detail --scan > /etc/mdadm/mdadm.conf
```

---

### 3.6 initramfs 재생성 (재발 방지 필수)

```bash
update-initramfs -u -k all
```

---

### 3.7 임시 사용자 생성 (운영 복구 목적)

> `/home` 파티션이 존재하지 않거나 분리되지 않은 상태이므로
> **임시 사용자로 로그인 복구 진행**

```bash
adduser test
usermod -aG sudo test
```

---

### 3.8 재부팅 (응급 쉘 상태에서)

```bash
sync
reboot -f
```

또는:

```bash
sync
echo b > /proc/sysrq-trigger
```

---

## 4. 부팅 후 검증 체크리스트

```bash
# systemd 정상 여부
systemctl is-system-running
```

정상:

```
running
```

```bash
# iSCSI 비활성화 확인
systemctl status iscsi
```

```bash
# RAID 상태 확인
lsblk
cat /proc/mdstat
```

```bash
# 로그인 테스트
login test
```

---

## 5. /home 관련 정리 방침

* 현재 상태:

  * `/home`은 루트 파티션 위
  * RAID에 별도 홈 파티션 없음

* 단기:

  * 임시 사용자로 운영 가능

* 중·장기:

  * RAID 파티션 신규 생성
  * `/home` 분리 마운트
  * 사용자 홈 이전 및 fstab 고정

---

## 6. 결론 요약

* 로그인 실패의 **본질은 패스워드가 아닌 스토리지/부팅 서비스 문제**
* iSCSI 자동 로딩이 **DGX OS 6 + RAID 환경에서 부팅을 차단**
* iSCSI 영구 비활성화 + initramfs 재생성으로 재발 방지 가능
* 임시 사용자 생성으로 즉시 운영 복구 가능
