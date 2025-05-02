# Version 1: Docker Swarm 배포 가이드

본 문서는 Docker Swarm 클러스터 구성 및 docker-compose.yml 파일을 통한 컨테이너 자원 예약 설정 방법을 설명합니다.

---

## 1. 준비 사항

1. 모든 서버에 동일한 CUDA 버전 설치  
2. Docker 및 nvidia-docker 등의 적절한 버전 설치  
3. 기타 필요한 라이브러리 및 패키지 설치

### 1.1 docker compose 설치 확인
```
docker compose version
```
---

## 2. Docker Swarm 구성

### 2.1 Manager 노드 초기화

Manager 노드에서 아래 명령어를 실행하여 Swarm을 초기화합니다.

```bash
sudo docker swarm init
sudo docker swarm join-token worker
```

아래 결과 나오면 아래 토큰 포함하여 각 worker 서버에 입력

```
docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxxxxxxxx <MANAGER-IP>:2377
```
출력된 토큰을 확인한 후, Worker 노드에 아래 명령어를 이용해 조인합니다.<br>
`sudo docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxxxxxxxx `

### 2.2 Worker 노드 및 Docker Stack 추가

Worker 노드에서는 아래 스크립트를 사용해 cpu, 메모리, GPU 수를 고려한 docker-compose.yml 파일 생성 및 Swarm Stack에 배포할 수 있습니다.

1. 스크립트 파일 생성 및 수정
2. 스크립트에 실행 권한 부여 후 실행

```bash
vim make_compose_yml.sh
# 아래 파일 추가
chmod +x make_compose_yml.sh
bash make_compose_yml.sh
```
스크립트 예제
  - 기반 docker img: nvcr.io/nvidia/cuda:12.6.3-cudnn-devel-ubuntu22.04
```bash
#!/bin/bash

# 도커 스웜 워커 설정. 아래 내용 수정할 것
docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxxxxxxxx <MANAGER-IP>:2377
#####################
# 자원 예약 설정
#####################
TOTAL_CORES=$(nproc)
# 서버를 구동을 위한 cpu 코어 4개 남겨둠
RESERVED_CORES=4
AVAILABLE_CORES=$(($TOTAL_CORES - $RESERVED_CORES))
HALF_CORES=$(($AVAILABLE_CORES / 2))

TOTAL_MEM=$(free -m | awk '/Mem:/ {print $2}')
# 전체 메모리의 20% 예약 (정수 나눗셈이므로 소수점은 버림)
RESERVED_MEM=$(($TOTAL_MEM / 5))
AVAILABLE_MEM=$(($TOTAL_MEM - $RESERVED_MEM))
HALF_MEM=$(($AVAILABLE_MEM / 2))

####################
# GPU 설정
####################
TOTAL_GPUS=$(nvidia-smi -L | wc -l)
HALF_GPUS=$(($TOTAL_GPUS / 2))


echo "Total CPU: $TOTAL_CORES, AVAILABLE_CORES: $AVAILABLE_CORES, Half CPU: $HALF_CORES"
echo "Total Memory: ${TOTAL_MEM}MB, AVAILABLE_CORES: {$AVAILABLE_CORES}MB, Half Memory: ${HALF_MEM}MB"
echo "Total GPUs: $TOTAL_GPUS, Half GPUs: $HALF_GPUS"

# GPU 할당 디바이스 설정
if [ "$TOTAL_GPUS" -eq 0 ]; then
    echo "GPU가 0개 있습니다. 스크립트를 종료합니다."
    exit 1
elif [ "$TOTAL_GPUS" -eq 1 ]; then
    echo "GPU가 1개만 있습니다. 하나의 컨테이너만 생성합니다."
    echo "docker-compose.yml 파일(1개 GPU 전용) 생성 중..."
    cat <<EOF > docker-compose.yml
version: "3.8"
services:
  container1:
    image: test:0.1
    command: sleep infinity
    # runtime: nvidia # swarm에서 runtime 작동X
    deploy:
      resources:
        limits:
          cpus: "${AVAILABLE_CORES}"    
          memory: "${AVAILABLE_MEM}m"
      placement:
        constraints:
          - node.labels.gpu == true
    environment:
      - NVIDIA_VISIBLE_DEVICES=0
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    gpus: "device=0"
EOF
    echo "docker-compose.yml 파일(1개 GPU용)이 생성되었습니다."
else
    GPU_CONTAINER1=$(seq -s, 0 $(($HALF_GPUS - 1)))
    GPU_CONTAINER2=$(seq -s, $HALF_GPUS $(($TOTAL_GPUS - 1)))
    echo "docker-compose.yml 파일 생성 중..."
    cat <<EOF > docker-compose.yml
version: "3.8"
services:
  container1:
    image: test:0.1
    ports:
      -8888:8888
    command: sleep infinity
    # runtime: nvidia # swarm에서 runtime 작동X
    deploy:
      resources:
        limits:
          cpus: "${HALF_CORES}"
          memory: "${HALF_MEM}m"
      placement:
        constraints:
          - node.labels.gpu == true
    environment:
      - NVIDIA_VISIBLE_DEVICES=${GPU_CONTAINER1}
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    gpus: "device=${GPU_CONTAINER1}"
  container2:
    image: test:0.1
    ports:
      -8889:8888
    command: sleep infinity
    # runtime: nvidia # swarm에서 runtime 작동X
    deploy:
      resources:
        limits:
          cpus: "${HALF_CORES}"
          memory: "${HALF_MEM}m"
      placement:
        constraints:
          - node.labels.gpu == true
    environment:
      - NVIDIA_VISIBLE_DEVICES=${GPU_CONTAINER2}
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    gpus: "device=${GPU_CONTAINER2}"
EOF
    echo "docker-compose.yml 파일이 생성되었습니다."
fi

# docker-compose up -d 대신 swarm stack에 디플로이
sudo docker stack deploy -c docker-compose.yml server_manage
```
---

## 3.요약
- Manager 노드에서 Swarm 초기화 및 토큰 조회
- Worker 노드에서 스크립트를 통해 docker-compose.yml 파일 생성
- Swarm Stack 배포로 컨테이너 실행