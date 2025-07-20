<!-- filepath: /C:/Users/PBJ/Desktop/pypi.md -->
# Version 1: PyPI 서버 구축 가이드

이 문서는 Bandersnatch를 이용한 PyPI 미러 서버 구축 및 주기적인 미러링 업데이트 방법을 설명합니다.

---

## 1. 요구 사항

- Python 3.x 설치
- pip (Python 패키지 관리자)
- Bandersnatch (PyPI 미러링 도구)
- 충분한 디스크 공간 (미러 데이터 저장용)
- 관리자(root) 권한 (설정 파일 편집 및 서비스 등록 시 필요)

---

## 2. Bandersnatch 설치 및 기본 설정

### 2.0 특정 버전 특정하기 위해 확인할 점(2025-02-26 기준)
1. tensorflow 지원 버전 확인 [링크](https://www.tensorflow.org/install/source?hl=ko#gpu)
  - tensorflow 2.17.0 기준 파이썬 3.9 ~ 3.12, cuda12.3
  - tensorflow 2.0.0 기준 파이썬 3.3 ~ 3.7, cuda10.0
2. pytorch 버전 확인 [링크](https://pytorch.org/get-started/locally/)
  - 2025-02-26 기준 CUDA: 12.6 지원


### 2.1 Bandersnatch 설치

아래 명령어를 통해 Bandersnatch를 설치합니다.

```bash
pip install bandersnatch
```

### 2.2 Bandersnatch 수정

- `src\bandersnatch_filter_plugins\filename_name.py` 파일 수정
  - `/home/super/lib/python3.10/site-packages/bandersnatch_filter_plugins/filename_name.py
`

```python
    ...
    # 타입 추가
    _etcType = [
        "_i686",
        "_arm",
        "_ppc64",
        "_s390",
        "_aarch64"
    ]
    ...

    ...
    # etc 조건 추가: x86_64만 다운로드 하도록록
            elif lplatform in "etc":
                self._patterns.extend([lplatform])
                self._patterns.extend(["macosx_", "macosx-"])
                self._packagetypes.extend(["bdist_dmg"])
                self._patterns.extend(self._windowsPlatformTypes)
                self._packagetypes.extend(["bdist_msi", "bdist_wininst"])
                self._patterns.extend([".freebsd", "-freebsd"])
    ...
```

### 2.3 설정 파일 수정
Bandersnatch의 기본 설정 파일을 편집하여 미러링 경로와 기타 옵션을 구성합니다.


```bash
sudo vim /etc/bandersnatch.conf
```

`/etc/bandersnatch.conf` 파일
```ini
[mirror]
directory = /data/pypi
json = false
release-files = true
cleanup = false
master = https://pypi.org
timeout = 200
global-timeout = 3600
workers = 10
hash-index = false
simple-format = ALL
stop-on-error = false
storage-backend = filesystem
verifiers = 3
compare-method = hash
[plugins]
enabled =
    exclude_platform
    latest_release
    blacklist_project

[blocklist]
platforms =
    etc
    py2
    py3.1
    py3.2
    py3.3
    py3.4
    py3.5
    py3.6
    py3.7

[latest_release]
keep = 20
```
---

## 2.3 미러링 수행
설정이 완료되면, 아래 명령어로 PyPI 미러링을 시작합니다.

```bash
sudo mkdir /data/pypi
sudo chown -R ${USER}:${USER} /data/pypi
bandersnatch mirror
# nohup /usr/bin/python3 /home/${USER}/.local/bin/bandersnatch mirror &

```

## 3. Nginx 설정
미러링이 완료된 후, PyPI 미러 서버를 웹 서버(Nginx 또는 Apache)를 통해 서비스할 수 있습니다.

### 3.1 Nginx 설치
먼저 Nginx가 설치되어 있지 않다면, 아래 명령어로 설치합니다.
```bash
sudo apt-get update
sudo apt-get install nginx
```
DNS 포트 열기 
```bash
sudo ufw allow 53 comment 'DNS'
sudo ufw reload
```

### 3.2 Nginx 설정 파일 구성
Nginx 설정 파일을 편집하여 PyPI 미러 서버의 정적 파일이 위치한 `/data/pypi` 디렉터리를 서비스하도록 설정합니다.
<br><br>
보통 사이트 별 설정은 `/etc/nginx/sites-available/` 디렉터리에 생성하고, `/etc/nginx/sites-enabled/`에 심볼릭 링크를 생성하여 활성화합니다.
<br><br>
예를 들어, 새 설정 파일 `/etc/nginx/sites-available/pypi`을 생성합니다.

```bash
sudo vim /etc/nginx/sites-available/pypi
```

```conf
server {
    # 기본 포트를 80번으로 리스닝
    listen 80;
    # 본인의 도메인 또는 IP 주소를 사용합니다.
    server_name pypi.smc.com;

    # /simple 요청 시 자동으로 /simple/로 리디렉션
    location = /simple {
        return 301 /simple/;
    }

    # 요청이 들어올 경우 /data/pypi 디렉터리를 참조
    location /simple/ {
        # alias는 지정한 경로로 요청을 매핑합니다.
        alias /data/pypi/web/simple/;
        # 디렉터리 목록 출력 활성화 (필요시 적용하지 않을 수 있음)
        autoindex on;
        index index.html;
        # MIME 타입 설정 (선택 사항)
        #include mime.types;
        #autoindex_exact_size off;
        #default_type application/octet-stream;
    }

    location /packages/ {
        alias /data/pypi/web/packages/;
        autoindex on;
    }
    # 에러 로그 파일 위치 설정 (선택 사항)
    error_log /var/log/nginx/pypi_error.log;
    # 접근 로그 파일 위치 설정 (선택 사항)
    access_log /var/log/nginx/pypi_access.log;
}
```

### 3.3 설정 활성화

생성한 설정 파일을 활성화시키기 위해 심볼릭 링크 생성
```bash
sudo ln -s /etc/nginx/sites-available/pypi /etc/nginx/sites-enabled/
```
설정 파일에 오류가 없는지 확인합니다.
```bash
sudo nginx -t
```
오류가 없으면 아래 명령어로 Nginx를 다시 시작하여 적용합니다.
```bash
sudo systemctl restart nginx
```
---
## 4. Bind 설정

### 4.1 Bind 설치
```
sudo apt update
sudo apt install bind9 bind9utils bind9-doc
```
### 4.2. `/etc/bind/named.conf.local` 설정 추가
```conf
zone "smc.com" {
    type master;
    file "/etc/bind/db.smc.com";
};
```

### 4.3. Zone 파일 생성 `/etc/bind/db.smc.com`  
```dns
$TTL    604800
@       IN      SOA     ns.smc.com. admin.smc.com. (
                              2025031201 ; Serial (날짜형식: YYYYMMDDXX)
                              604800     ; Refresh
                              86400      ; Retry
                              2419200    ; Expire
                              604800 )   ; Negative Cache TTL
;
@       IN      NS      ns.smc.com.
ns      IN      A       192.168.1.100
pypi    IN      A       192.168.1.101

```
---
## 5. 각 [Client] 설정(도커이미지에서)

### 5.1 pip.conf 생성
1. 리눅스: `~/.pip/pip.conf`
2. Windows: `%APPDATA%\pip\pip.ini`

```bash
mkdir ~/.pip
vim ~/.pip/pip.conf
```

```bash
[global]
index-url = http://pypi.smc.com/simple
trusted-host = pypi.smc.com
```
### 5.2 hosts 설정
1. 리눅스: `/etc/hosts`파일에 `<PyPI서버 주소>    <DNS>` 추가
```bash
127.0.0.1 localhost
127.0.1.1 ubuntu
119.86.100.149    pypi.smc.com <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<추가

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
2. 윈도우: `C:\Windows\System32\drivers\etc\hosts`파일에 추가 
```bash
[사내 PyPI 서버 IP]   pypi.smc.com
```
## 6. 스토리지 서버 --> SSD 세팅
NFS 구성 후 마운트하여 작업하는 것이 안정적으로 보임.
### NFS 서버 (스토리지 서버, 10.201.225.2)
```bash
sudo apt update
sudo apt install -y nfs-kernel-server
sudo vim /etc/exports
```

```
/data/pypi/web/packages  10.201.225.0/24(rw,sync,no_subtree_check)
```

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
sudo ufw allow from 10.201.225.0/24 to any port nfs
```

### NFS 클라이언트 (SSD 연결한 서버, 10.201.225.166)
```bash
sudo apt update
sudo apt install -y nfs-common

sudo mkdir -p /mnt/nas
sudo mount -t nfs 172.30.1.118:/data/pypi/web/packages /mnt/nas
sudo mount -o remount,rw /ssd/pypi

# 고정 원하면
# sudo vim /etc/fstab
# 아래 추가
# 10.201.225.2:/data/pypi/web/packages  /mnt/nas  nfs  defaults,_netdev  0  0
```

### 대망의 실행
`nohup rsync -aW --info=progress2 /mnt/nas/ /ssd/pypi/web/packages/ > ~/rsync.log 2>&1 &`

## 7. SSD -->  내부망 서버
... 어떤 어려움이 기다리고 있을까??
winSCP
