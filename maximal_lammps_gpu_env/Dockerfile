# LAMMPS 2024 with CUDA 12.6 + GPU acceleration + OpenMPI + Mamba
# For Windows + RTX 3080

FROM nvidia/cuda:12.6.1-devel-ubuntu22.04 AS builder

LABEL maintainer="kangmg"

ENV DEBIAN_FRONTEND=noninteractive

# Install build dependencies
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        cmake \
        curl \
        git \
        gfortran \
        libfftw3-dev \
        libhdf5-dev \
        libjpeg-dev \
        liblapack-dev \
        libopenmpi-dev \
        libpng-dev \
        libreadline-dev \
        libzstd-dev \
        openmpi-bin \
        python3-dev \
        python3-pip \
        python3-venv \
        zlib1g-dev \
        wget && \
    rm -rf /var/lib/apt/lists/*

# Download LAMMPS 2024 stable release
ARG LAMMPS_VERSION=stable_29Aug2024_update3
RUN cd /tmp && \
    wget https://github.com/lammps/lammps/archive/refs/tags/${LAMMPS_VERSION}.tar.gz && \
    tar xzf ${LAMMPS_VERSION}.tar.gz && \
    mv lammps-${LAMMPS_VERSION} lammps

# Build MPI + GPU version
# Add CUDA stubs to library path for linking
RUN echo "/usr/local/cuda/lib64/stubs" > /etc/ld.so.conf.d/cuda-stubs.conf && ldconfig

RUN mkdir -p /tmp/lammps/build-gpu && \
    cd /tmp/lammps/build-gpu && \
    cmake -D CMAKE_BUILD_TYPE=Release \
          -D CMAKE_INSTALL_PREFIX=/usr \
          -D CMAKE_INSTALL_SYSCONFDIR=/etc \
          -D LAMMPS_MACHINE=gpu \
          -D BUILD_MPI=on \
          -D BUILD_OMP=on \
          -D PKG_GPU=on \
          -D GPU_API=cuda \
          -D GPU_ARCH=sm_86 \
          -D CUDA_CUDA_LIBRARY=/usr/local/cuda/lib64/stubs/libcuda.so \
          -D PKG_MOLECULE=on \
          -D PKG_MANYBODY=on \
          -D PKG_KSPACE=on \
          -D PKG_RIGID=on \
          -D PKG_REAXFF=on \
          -D PKG_MISC=on \
          -D PKG_EXTRA-MOLECULE=on \
          -D PKG_EXTRA-PAIR=on \
          -D LAMMPS_EXCEPTIONS=on \
          -D BUILD_TOOLS=on \
          -D BUILD_SHARED_LIBS=on \
          -D Python_EXECUTABLE=/usr/bin/python3 \
          /tmp/lammps/cmake && \
    make -j $(nproc) && \
    make install && \
    tar czvf /tmp/lammps-gpu.tgz -T install_manifest.txt

# Remove stubs from ldconfig (will use real driver at runtime)
RUN rm /etc/ld.so.conf.d/cuda-stubs.conf && ldconfig

# Runtime image - use devel for full CUDA libs
FROM nvidia/cuda:12.6.1-devel-ubuntu22.04

LABEL maintainer="kangmg"

ENV DEBIAN_FRONTEND=noninteractive
ENV LAMMPS_POTENTIALS=/usr/share/lammps/potentials
ENV MSI2LMP_LIBRARY=/usr/share/lammps/frc_files

# Install runtime dependencies
RUN apt-get update -y && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
        # MPI runtime
        openmpi-bin \
        libopenmpi3 \
        mpi-default-bin \
        # Libraries
        liblapack3 \
        libfftw3-double3 \
        libhdf5-103-1 \
        libjpeg-turbo8 \
        libpng16-16 \
        libreadline8 \
        libzstd1 \
        zlib1g \
        # System libraries
        libc6 \
        libgcc-s1 \
        libgomp1 \
        libstdc++6 \
        libevent-2.1-7 \
        libevent-pthreads-2.1-7 \
        libexpat1 \
        libhwloc15 \
        libltdl7 \
        libudev1 \
        # Tools
        hdf5-tools \
        ffmpeg \
        less \
        rsync \
        ssh \
        vim \
        git \
        wget \
        curl \
        zstd && \
    rm -rf /var/lib/apt/lists/*

# Install Miniforge (Mamba)
ENV CONDA_DIR=/opt/conda
ENV PATH=$CONDA_DIR/bin:$PATH

RUN wget -q https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh -O /tmp/miniforge.sh && \
    bash /tmp/miniforge.sh -b -p $CONDA_DIR && \
    rm /tmp/miniforge.sh && \
    conda clean -afy

# Create Python 3.11 environment
RUN mamba create -n sci python=3.11 -y && \
    mamba install -n sci -y \
        ase \
        numpy \
        scipy \
        matplotlib \
        pandas \
        jupyterlab && \
    mamba clean -afy

# Activate sci environment by default
RUN echo "source activate sci" >> /etc/bash.bashrc
ENV PATH=$CONDA_DIR/envs/sci/bin:$PATH

# Copy LAMMPS from builder
COPY --from=builder /tmp/lammps-gpu.tgz /tmp/
RUN tar -xvzf /tmp/lammps-gpu.tgz -C / && rm -f /tmp/lammps-gpu.tgz

# Copy and install LAMMPS Python package
RUN mkdir /tmp/lammps
COPY --from=builder /tmp/lammps/python /tmp/lammps/python
COPY --from=builder /tmp/lammps/src/version.h /tmp/lammps/src/version.h
RUN cd /tmp/lammps/python && \
    $CONDA_DIR/envs/sci/bin/pip install . && \
    rm -rf /tmp/lammps

# OpenMPI settings for Docker (run as root)
ENV OMPI_ALLOW_RUN_AS_ROOT=1
ENV OMPI_ALLOW_RUN_AS_ROOT_CONFIRM=1

# SSH Server setup (basic)
RUN apt-get update && apt-get install -y openssh-server sudo && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir /var/run/sshd

EXPOSE 10022

WORKDIR /workspace

CMD ["/usr/sbin/sshd", "-D", "-p", "10022"]