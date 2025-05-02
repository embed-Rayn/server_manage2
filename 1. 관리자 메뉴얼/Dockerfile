FROM nvcr.io/nvidia/cuda:12.6.3-cudnn-devel-ubuntu22.04

# 필요한 시스템 패키지 설치
RUN apt update
RUN apt install -y \
        apt-utils \
        build-essential \
        ca-certificates \
        curl \
        gpg \
        kmod \
        file \
        libelf-dev \
        libglvnd-dev \
        pkg-config \
        wget \
        vim \
        zlib1g-dev \
        libncurses5-dev \
        libgdbm-dev \
        libnss3-dev \
        libssl-dev \
        libreadline-dev \
        libffi-dev\
        software-properties-common\
  && rm -rf /var/lib/apt/lists/*

ARG DEBIAN_FRONTEND=noninteractive
ENV DEBIAN_FRONTEND=noninteractive
RUN ln -fs /usr/share/zoneinfo/Asia/Seoul /etc/localtime && \
    apt-get update && \
    apt-get install -y tzdata && \
    dpkg-reconfigure -f noninteractive tzdata

RUN apt-add-repository ppa:deadsnakes/ppa
RUN apt install -y python3.11 python3-pip
RUN ln -sf /usr/bin/python3.11 /usr/bin/python\
  && ln -sf /usr/bin/python3.11 /usr/bin/python3

# 최신 pip 업그레이드 후, Jupyter Lab 설치
RUN pip install --upgrade pip && \
    pip install jupyterlab

# 작업 디렉토리 설정
WORKDIR /home/jupyter

# 컨테이너 시작 시 Jupyter Lab을 실행하도록 설정
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--allow-root", "--no-browser"]
