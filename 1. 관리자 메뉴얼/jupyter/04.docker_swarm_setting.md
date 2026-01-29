# Docker in Server
1. *Docker Image/Container*
1. *Docker Compose*
1. **Docker Swarm**
---
## *Docker Compose*
- deeeeeeeeeeeeeeeeeelete
- docker-compose.yml
```yml
version: "3.8"
services:
  container1:
    image: test:0.1
    ports:
      - 8888:8888
    command: jupyter lab --config=/root/.jupyter/jupyter_lab_config.py
    volumes:
      - /workspace/8888:/home/jupyter
      - ./jupyter_lab_config.py:/root/.jupyter/jupyter_lab_config.py:ro
    deploy:
      resources:
        limits:
          cpus: "${AVAILABLE_CORES}"
          memory: "${AVAILABLE_MEM}m"
        reservations:
          devices:
          - driver: nvidia
            device_ids: ${GPU_CONTAINER1_FORMATTED}
            capabilities: [gpu]
      placement:
        constraints:
          - node.labels.gpu == true
    environment:
      - NVIDIA_VISIBLE_DEVICES=${GPU_CONTAINER1}
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
  container2:
    image: test:0.1
    ports:
      - 8889:8888
    command: jupyter lab --config=/root/.jupyter/jupyter_lab_config.py
    volumes:
      - /workspace/8889:/home/jupyter
      - ./jupyter_lab_config.py:/root/.jupyter/jupyter_lab_config.py:ro
    deploy:
      resources:
        limits:
          cpus: "${AVAILABLE_CORES}"
          memory: "${AVAILABLE_MEM}m"
        reservations:
          devices:
          - driver: nvidia
            device_ids: ${GPU_CONTAINER2_FORMATTED}
            capabilities: [gpu]
      placement:
        constraints:
          - node.labels.gpu == true
    environment:
      - NVIDIA_VISIBLE_DEVICES=${GPU_CONTAINER2}
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
```
---
## 3. Docker Swarm: Manager

### Docker-Compose version 확인
2.22 이상
```
docker compose version
```
### 초기: Manager 노드 초기화
Manager 노드에서 아래 명령어를 실행하여 Swarm을 초기화

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

```bash
sudo docker stack deploy -c docker-compose.yml server_manage
```
---
## 3. Docker Swarm: Worker
