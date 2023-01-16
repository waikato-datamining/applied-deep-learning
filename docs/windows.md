---
title: Docker on Windows
---

# Installing Docker

One way of installing Docker on Windows is using [Docker Desktop](https://www.docker.com/products/docker-desktop/).
However, the terms for using Docker Desktop changed on August 31st, 2022 and it may no
longer be free for you (e.g., when working at a larger research organization or company).

However, if you are not afraid of running a few commands in the terminal and
having a terminal open while working with Docker, then you can follow these
instructions of getting Docker working under [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install).

* Get **Ubuntu 20.04.5** from the Windows store
* Configure the default user and password when asked for during the installation
* Get your system ready

```bash
sudo apt update && sudo apt upgrade`
sudo apt install --no-install-recommends apt-transport-https ca-certificates curl gnupg2
```
  
* Change *iptables* to `legacy`:
  
```bash
sudo update-alternatives --config iptables
```

* Install Docker

```bash
. /etc/os-release
curl -fsSL https://download.docker.com/linux/${ID}/gpg | sudo tee /etc/apt/trusted.gpg.d/docker.asc
echo "deb [arch=amd64] https://download.docker.com/linux/${ID} ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
```
  
* Close the WSL2 window and open a new one for the changes to take effect
* Test your Docker installation by running:
  
```bash
sudo dockerd
```
  
* After a lot of output of Docker starting up, you should see the following:

```
API listen on /var/run/docker.sock
```

(The above instructions were taken from [this post](https://dev.to/bowmanjd/install-docker-on-windows-wsl-without-docker-desktop-34m9) by Jonathan Bowman)


# Installing NVIDIA Docker

Since our CUDA dependencies are packaged within the Docker images, we only need
to worry about installing [NVIDIA Docker](https://github.com/NVIDIA/libnvidia-container)
to get access to the GPU:

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
```

Due to installing a new runtime, we need to restart our `dockerd` daemon, of course.

(The above instructions were taken from [this post](https://medium.com/htc-research-engineering-blog/nvidia-docker-on-wsl2-f891dfe34ab) by Frank Chung)


# Testing the GPU

The following commands test the inference of a Yolov7 model on the GPU: 

* Create a test directory:

```bash
mkdir gpu-test
cd gpu-test
```

* Download the pre-trained model and a test image:

```bash
wget https://github.com/WongKinYiu/yolov7/releases/download/v0.1/yolov7_training.pt
wget "https://raw.githubusercontent.com/waikato-datamining/adams-addons/master/adams-docker/src/main/flows/data/2021_Toyota_GR_Yaris_Circuit_4WD_1.6_(1).jpg"
```

* Perform inference:

```bash
docker run -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -w /workspace \
  --gpus=all \
  -t waikatodatamining/pytorch-yolov7:2022-10-08_cuda11.1 \
  yolov7_detect \
  --weights ./yolov7_training.pt \
  --source ./'2021_Toyota_GR_Yaris_Circuit_4WD_1.6_(1).jpg' \
  --no-trace \
  --conf-thres 0.8 \
  --device 0
```

* In directory `gpu-test/runs/detect/exp` you will find the image with the
  detected objects overlaid.
  