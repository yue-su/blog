---
layout: post
title: Setting Up vGPU in Docker Container (with Specific Python Version)
category: DevOps
---

Our team has recently been building a backend service using the Django framework. Alongside this, we've also developed an in-house machine learning command line utility tool. To run this tool efficiently, we utilize an instance with an enabled nVidia GPU. The challenge lies in setting up an environment where both Django and the utility are containerized with Docker, with the ability to harness the GPU power inside the container. In addition, the tool requires a specific Python version (3.9 in our case) to run.

This task was not as straightforward as we initially imagined. It seemed we could not just pull off a Python 3.9 office image, install the CUDA driver, and expect it to run smoothly. After a few attempts, we found this to be a considerable challenge. We then took a different approach, opting to use an official nvidia/cuda image and installing the needed Python version. After several tweaks, it finally worked. 

Here are the main steps involved:

1. **Ensure that the GPU driver is installed properly on the instance.** This is relatively easy. We followed this [guide](https://docs.alliancecan.ca/wiki/Using_cloud_vGPUs#Preparation_of_a_VM_running_Debian11). To check if everything is working as expected, use the `nvidia-smi` command line. If all is well, you will see the CUDA version and other related information. For additional guidance, refer to the official [CUDA installation guide](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html).

    ```
    +-----------------------------------------------------------------------------+
    | NVIDIA-SMI 470.199.02   Driver Version: 470.199.02   CUDA Version: 11.4     |
    |-------------------------------+----------------------+----------------------+
    | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
    |                               |                      |               MIG M. |
    |===============================+======================+======================|
    |   0  GRID V100D-8C       On   | 00000000:00:05.0 Off |                    0 |
    | N/A   N/A    P0    N/A /  N/A |    560MiB /  8192MiB |      0%      Default |
    |                               |                      |                  N/A |
    +-------------------------------+----------------------+----------------------+
                                                                                
    +-----------------------------------------------------------------------------+
    | Processes:                                                                  |
    |  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
    |        ID   ID                                                   Usage      |
    |=============================================================================|
    |  No running processes found                                                 |
    +-----------------------------------------------------------------------------+
    ```

2. **Use the base Dockerfile to build the backend.** It's crucial to mention that the chosen image should be based on the CUDA driver version used in your environment. If they do not match, the container might throw errors. Ubuntu20.04 comes with Python 3.8 by default. If you do not set up symbolic links correctly, the packages will be installed with Python 3.8, resulting in errors. We used the official [NVIDIA CUDA Docker images](https://hub.docker.com/r/nvidia/cuda) as a base.

    ```Dockerfile
    # Use an official NVIDIA CUDA runtime as a parent image
    FROM nvidia/cuda:11.4.3-cudnn8-devel-ubuntu20.04

    # Set environment variables
    ENV PYTHONUNBUFFERED 1
    ENV DEBIAN_FRONTEND=noninteractive
    ENV TZ=America/Toronto

    # Install Python and pip
    RUN apt-get update && apt-get install -y \
        python3.9 \
        python3-pip \
        && rm -rf /var/lib/apt/lists/*

    # Create symbolic links for Python and pip
    RUN ln -sf /usr/bin/python3.9 /usr/bin/python \
        && ln -sf /usr/bin/python3.9 /usr/bin/python3 \
        && ln -sf /usr/bin/pip3 /usr/bin/pip

    # Set the working directory to /backend
    WORKDIR /backend
    COPY requirements.txt requirements.txt

    RUN pip install --upgrade pip
    RUN pip install -r requirements.txt

    COPY . .
    ```

3. **Use Docker Compose to manage our services.** Here is a base yml file you can use if you're utilizing Django as the backend. Make sure to configure the `runtime` and `environment` settings correctly for the container to properly utilize the GPU.

    ```yaml
    version: '3'

    services:
    backend:
        build: 
        context: .
        dockerfile: Dockerfile
        ports:
        - 8000:8000
        container_name: backend
        runtime: nvidia
        environment:
        - NVIDIA_VISIBLE_DEVICES=all
        restart: unless-stopped
        volumes:
        - ./:/backend
        env_file:
        - .env
        command: >
        bash -c "python manage.py makemigrations
        && python manage.py migrate
        && python manage.py runserver 0.0.0.0:8000"
    ```

4. **Test your configuration.** If all the previous steps were successful, you should be able to run the `nvidia-smi` command within the container.