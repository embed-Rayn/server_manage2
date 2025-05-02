# 터널링
## Localtunnel
- 오픈 소스 기반으로 매우 단순하고 설치 및 사용이 간편
- 별도의 복잡한 설정 없이 명령 한 줄로 로컬 서버를 외부에 공개 가능

```bash
sudo apt install npm
npx localtunnel --port 8889 --subdomain smc_ai_center --print-requests
lt --port 80 --subdomain kibua20 --print-requests
```

## ngrok
### ngrok 설치 
```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc \
  | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \
  && echo "deb https://ngrok-agent.s3.amazonaws.com buster main" \
  | sudo tee /etc/apt/sources.list.d/ngrok.list \
  && sudo apt update \
  && sudo apt install ngrok
```
### ngrok 설정
- 토큰 보는 주소: https://dashboard.ngrok.com/get-started/your-authtoken

```bash
ngrok config add-authtoken <token>
```
ngrok config add-authtoken 2vADwmHjqLaxnF3LUjEefihetiN_TTPh8BKzSvJNL1VZquyp
### ngrok 실행 및및 주소 확인
```bash
nohup ngrok http 8888 &
curl http://127.0.0.1:4040/api/tunnels
```