BootStrap: docker
From: nvidia/cuda:8.0-cudnn6-runtime-ubuntu16.04

%runscript

    echo "examples at location /examples"

%post
 

    # TensorFlow version is tightly coupled to CUDA and cuDNN so it should be selected carefully
    export TENSORFLOW_VERSION=1.6.0
    export CUDNN_VERSION=7.0.5.15-1+cuda9.0
    export NCCL_VERSION=2.1.15-1+cuda9.0

    # Python 2.7 or 3.5 is supported by Ubuntu Xenial out of the box
    export PYTHON_VERSION=2.7

    echo "deb http://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/nvidia-ml.list

    apt-get update && apt-get install -y --no-install-recommends \
            build-essential \
            cmake \
            git \
            curl \
            vim \
            wget \
            ca-certificates \
            libcudnn7=$CUDNN_VERSION \
            libnccl2=$NCCL_VERSION \
            libnccl-dev=$NCCL_VERSION \
            libjpeg-dev \
            libpng-dev \
            python$PYTHON_VERSION \
            python$PYTHON_VERSION-dev

    ln -s /usr/bin/python$PYTHON_VERSION /usr/bin/python

    curl -O https://bootstrap.pypa.io/get-pip.py && \
        python get-pip.py && \
        rm get-pip.py

    # Install TensorFlow and Keras
    pip install --no-cache-dir tensorflow-gpu==$TENSORFLOW_VERSION keras h5py

    # Install Open MPI
    mkdir /tmp/openmpi && \
        cd /tmp/openmpi && \
        wget https://www.open-mpi.org/software/ompi/v3.0/downloads/openmpi-3.0.0.tar.gz && \
        tar zxf openmpi-3.0.0.tar.gz && \
        cd openmpi-3.0.0 && \
        ./configure --enable-orterun-prefix-by-default --enable-mpi-thread-multiple && \
        make -j $(nproc) all && \
        make install && \
        ldconfig && \
        rm -rf /tmp/openmpi

    # Install Horovod, temporarily using CUDA stubs
    ldconfig /usr/local/cuda-9.0/targets/x86_64-linux/lib/stubs && \
        HOROVOD_GPU_ALLREDUCE=NCCL pip install --no-cache-dir horovod && \
        ldconfig

    # Create a wrapper for OpenMPI to allow running as root by default
    mv /usr/local/bin/mpirun /usr/local/bin/mpirun.real && \
        echo '#!/bin/bash' > /usr/local/bin/mpirun && \
        echo 'mpirun.real --allow-run-as-root "$@"' >> /usr/local/bin/mpirun && \
        chmod a+x /usr/local/bin/mpirun

    # Configure OpenMPI to run good defaults:
    #   --bind-to none --map-by slot --mca btl_tcp_if_exclude lo,docker0
    echo "hwloc_base_binding_policy = none" >> /usr/local/etc/openmpi-mca-params.conf && \
        echo "rmaps_base_mapping_policy = slot" >> /usr/local/etc/openmpi-mca-params.conf && \
        echo "btl_tcp_if_exclude = lo,docker0" >> /usr/local/etc/openmpi-mca-params.conf

    # Set default NCCL parameters
    echo NCCL_DEBUG=INFO >> /etc/nccl.conf && \
        echo NCCL_SOCKET_IFNAME=^docker0 >> /etc/nccl.conf

    # Install OpenSSH for MPI to communicate between containers
    apt-get install -y --no-install-recommends openssh-client openssh-server && \
        mkdir -p /var/run/sshd

    # Allow OpenSSH to talk to containers without asking for confirmation
    cat /etc/ssh/ssh_config | grep -v StrictHostKeyChecking > /etc/ssh/ssh_config.new && \
        echo "    StrictHostKeyChecking no" >> /etc/ssh/ssh_config.new && \
        mv /etc/ssh/ssh_config.new /etc/ssh/ssh_config
