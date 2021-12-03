---
title: "kubernetes 설치"
excerpt: "kubeflow를 사용하기 위한 kubeflow 설치"
categories: kubeflow
tags: kubernetes kubeflow MLOps

---

개발 환경
===
```
운영체제 : ubuntu 20 lts
GPU : RTX3080
```
현재 설치한 k8s와 kubeflow 는 컴퓨터 한대에서 설치를 하였습니다.  
<br/>

도커 설치 및 설정
===
```bash
# 우분투에서 도커를 설치
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update

# 특정 버전을 설치하시고 싶을 때
sudo apt-get install -y \
docker-ce=5:18.09.9~3-0~ubuntu-bionic \
docker-ce-cli=5:18.09.9~3-0~ubuntu-bionic \
containerd.io

# 현재 맞는 최신 버전을 설치하시고 싶을 때
sudo apt-get update && sudo apt-get install -y \
docker-ce \
docker-ce-cli \
containerd.io
```

```bash
# 쿠버네티스를 위한 도커 설정

sudo su
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload
systemctl restart docker
exit
```
```bash
# 컨테이너에서 GPU 사용을 위한 nvidia-plugin 설치
release="ubuntu"$(lsb_release -sr | sed -e "s/\.//g")
sudo apt install sudo gnupg
sudo apt-key adv --fetch-keys "http://developer.download.nvidia.com/compute/cuda/repos/"$release"/x86_64/7fa2af80.pub"
sudo sh -c 'echo "deb http://developer.download.nvidia.com/compute/cuda/repos/'$release'/x86_64 /" > /etc/apt/sources.list.d/nvidia-cuda.list'
sudo sh -c 'echo "deb http://developer.download.nvidia.com/compute/machine-learning/repos/'$release'/x86_64 /" > /etc/apt/sources.list.d/nvidia-machine-learning.list'
sudo apt update
apt-cache search nvidia

# 택 1
-----------------------------------
# 알맞는 버전 찾아서 설치
sudo apt-get install -y nvidia-XXX

# 자동 설치
sudo ubuntu-drivers autoinstall
-----------------------------------
sudo apt-get install -y dkms nvidia-modprobe

sudo reboot
sudo cat /proc/driver/nvidia/version | nvidia-smi
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
sudo apt-get install -y nvidia-docker2

# daemon.json에 추가
$ sudo vi /etc/docker/daemon.json
   "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }

sudo systemctl restart docker
sudo docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
```
<br/>

쿠버네티스 설치
===
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

# 특정 버전을 설치하고 싶을 때 (이 실습에서는 1.19.16-00)
sudo apt-get install -y kubelet=1.18.5-00 kubeadm=1.18.5-00 kubectl=1.18.5-00

# 현재 맞는 최신 버전을 설치하시고 싶을 때
sudo apt-get install -y kubelet kubeadm kubectl

# 업데이트 방지
sudo apt-mark hold kubelet kubeadm kubectl
```
```bash
# 쿠버네티스가 작동하기 위한 리눅스 설정
sudo sysctl net.bridge.bridge-nf-call-iptables=1
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```