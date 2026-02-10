# 병원망 내 서버 개발환경 설치

작업 환경은 윈도우에서 WSL로 서버와 같은 OS 컨테이너 생성 후 작업
- apt install 하여 테스트
- apt download 하여 저장
해당 파일을 USB에 담아 병원망 컴퓨터로 이전

## 1. 우분투 버전 확인
   - Ubuntu 18.04의 경우 CUDA 11.8(최신 pytorch 기준, 우분투 지원 기준) 설치
   - Ubuntu 20.04의 경우 CUDA 12.8(최신 pytorch 기준, 우분투 지원 기준) 설치

## 2. WSL에서 서버 설치, 관련 패키지&드라이버 설치 및 다운로드

### WSL위 우분투(병원망 버전) 내 설치
#### 도커 설치 및 다운
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo mkdir /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install apt-rdepends sudo
```

```bash
# 일단 설치
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 의존성 모두 다운로드
apt-rdepends docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin \
| grep -v "^ " | sort -u > pkgs.txt
for pkg in $(cat pkgs.txt); do
    apt-get download "$pkg"
done
```

#### 컨테이너 툴킷 설치
```bash
sudo apt install -y gpg
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sudo sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update

sudo apt-get install -y nvidia-container-toolkit

apt-rdepends nvidia-container-toolkit \
| grep -v "^ " | sort -u > pkgs.txt
for pkg in $(cat pkgs.txt); do
    apt-get download "$pkg"
done
```

### WSL위 우분투(병원망 버전) 위 22.04 내 설치

---
# 3. CUDA 설치([우분투22.04 CUDA12.06](https://developer.nvidia.com/cuda-12-6-3-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_local))
- CUDA 패키지 준비: pin 파일 다운로드 및 이동, CUDA 데비안 패키지 다운로드 후 설치
- 키링 복사: 설치된 CUDA 키링 파일을 시스템 키링 디렉토리로 이동
- 패키지 설치: apt-get 업데이트 후 cuda-toolkit-12-6 설치
- 환경 변수 설정: PATH와 LD_LIBRARY_PATH가 bashrc에 추가되어 영구 반영
```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-ubuntu2204.pin
sudo mv cuda-ubuntu2204.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/12.6.3/local_installers/cuda-repo-ubuntu2204-12-6-local_12.6.3-560.35.05-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2204-12-6-local_12.6.3-560.35.05-1_amd64.deb
sudo cp /var/cuda-repo-ubuntu2204-12-6-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-6
echo "export PATH=/usr/local/cuda-12.6/bin${PATH:+:${PATH}}" >> "$HOME/.bashrc"
echo "export LD_LIBRARY_PATH=/usr/local/cuda-12.6/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}" >> "$HOME/.bashrc"
export PATH=/usr/local/cuda-12.6/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-12.6/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
```

---
# 4. NVIDIA Container Toolkit
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
### GB01: DNS 설정 초기화로 인터넷 연결 에러
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo vim /etc/systemd/resolved.conf.d/dns_servers.conf
```
`dns_servers.conf`
```conf
[Resolve]
DNS=8.8.8.8 8.8.4.4
FallbackDNS=1.1.1.1
```

```bash
sudo systemctl restart systemd-resolved
ls -l /etc/resolv.conf
#결과 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
```