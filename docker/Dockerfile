# Build with: docker build -f docker/Dockerfile .

# ✅ Change base image to CUDA runtime for GPU
FROM --platform=linux/amd64 nvidia/cuda:12.2.2-cudnn8-devel-ubuntu22.04

MAINTAINER Julia Kodysh <julia326@gmail.com>

RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections && \
    apt-get clean && \
    apt-get update && \
    apt-get install --no-install-recommends --no-upgrade -y \
        ca-certificates \
        curl \
        gawk \
        git \
        graphviz \
        g++ \
        libfreetype6-dev \
        libx11-dev \
        libbz2-dev \
        libcurl4-openssl-dev \
        libgnutls28-dev \
        libssl-dev \
        make \
        perl \
        pkg-config \
        python-tk \
        python2.7-dev \
        rsync \
        sudo \
        tar \
        tcsh \
        unzip \
        wget \
        xvfb \
        zlib1g-dev && \
    apt-get clean -y && \
    rm -rf /var/lib/apt/lists/*

RUN useradd --create-home --home-dir /home/user --shell /bin/bash -G sudo user && \
    echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

USER user
ENV HOME=/home/user
ENV SHELL=/bin/bash
ENV USER=user
WORKDIR /home/user

# Install vcftools (same as before)
RUN cd /tmp && wget https://github.com/openvax/neoantigen-vaccine-pipeline/releases/download/0.3.0/vcftools_0.1.13.tar.gz && \
    tar xvfz vcftools_0.1.13.tar.gz && cd vcftools_0.1.13/ && make && \
    sudo cp -r /tmp/vcftools_0.1.13/bin/* /usr/local/bin && \
    sudo cp -r /tmp/vcftools_0.1.13/lib/perl5/site_perl/* /usr/local/lib && \
    rm -rf /tmp/*

ENV PERL5LIB /usr/local/lib

# Install Miniconda
RUN wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh && \
    bash miniconda.sh -b -p $HOME/miniconda
ENV PATH $HOME/miniconda/bin:$PATH

ENV PGV_BIN /home/user/miniconda/envs/pgv/bin
ENV JAVA7_BIN /home/user/miniconda/envs/java7/bin

# Switch to a faster solver for conda

RUN conda config --set always_yes yes --set changeps1 no && \
    conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main && \
    conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r && \
    conda update -n base conda && \
    conda install -n base conda-libmamba-solver && \
    conda config --set solver libmamba

# install tools in pgv environment (same as before)
RUN conda config --add channels r && \
    conda config --add channels defaults && \
    conda config --add channels conda-forge && \
    conda config --add channels bioconda && \
    conda create -n pgv \
    setuptools \
    python=3.6 \
    bedtools \
    bwa==0.7.17 \
    cython \
    fastqc \
    gatk==3.7 \
    java-jdk==8.0.112 \
    picard==2.18.5 \
    pip \
    sambamba \
    samtools \
    star==2.6.0c \
    vcftools==0.1.15 \
    && conda clean --all -y

# mhcflurry env (same)
RUN conda config --add channels r && \
    conda config --add channels defaults && \
    conda config --add channels conda-forge && \
    conda config --add channels bioconda && \
    conda create -n mhcflurry \
    python=3.10 \
    setuptools \
    cython \
    pip \
    && conda clean --all -y

# java7 env (same)
RUN conda create -n java7 \
    java-jdk==7.0.91 \
    && conda clean --all -y

# MuTect & GATK setup (same)
RUN wget -O $HOME/mutect-1.1.7.jar http://storage.googleapis.com/common-files/tools/mutect-1.1.7.jar
ENV MUTECT $HOME/mutect-1.1.7.jar
RUN wget -O $HOME/GenomeAnalysisTK.jar http://storage.googleapis.com/common-files/tools/GenomeAnalysisTK.jar
RUN $PGV_BIN/gatk-register $HOME/GenomeAnalysisTK.jar

# ✅ Install GPU TensorFlow instead of CPU
COPY requirements.txt $HOME
ENV MHCFLURRY_BIN /home/user/miniconda/envs/mhcflurry/bin

# Upgrade pip, setuptools, and wheel first
RUN $MHCFLURRY_BIN/pip install --upgrade pip setuptools wheel

# Install all requirements (remove --ignore-installed)
RUN $MHCFLURRY_BIN/pip install -r $HOME/requirements.txt

# mhcflurry downloads and test
RUN $MHCFLURRY_BIN/mhcflurry-downloads fetch
RUN $MHCFLURRY_BIN/mhcflurry-predict --alleles HLA-A0201 --peptides SIINFEKL

# Path setup
ENV PATH $MHCFLURRY_BIN:$PGV_BIN:$PATH

# Vaxrank & snakemake setup (same)
ENV KERAS_BACKEND tensorflow
ENV PYENSEMBL_CACHE_DIR /reference-genome/pyensembl-cache
ENV VAXRANK_REF_PEPTIDES_DIR /reference-genome/reference-peptides
RUN curl https://pastebin.com/raw/AmfYN3er > $HOME/.fonts.conf

COPY . $HOME

# Strelka setup (same)
RUN wget https://github.com/Illumina/strelka/releases/download/v2.9.10/strelka-2.9.10.centos6_x86_64.tar.bz2 \
    && tar -xvf strelka-2.9.10.centos6_x86_64.tar.bz2 \
    && mv strelka-2.9.10.centos6_x86_64 $HOME/strelka \
    && rm strelka-2.9.10.centos6_x86_64.tar.bz2

ENV STRELKA_CONFIG $HOME/pipeline/strelka_config.txt
ENV STRELKA_BIN $HOME/strelka/bin
ENV SCRIPTS $HOME/pipeline/scripts

RUN mkdir -p $HOME/.snakemake && sudo chown -R $USER $HOME/.snakemake
RUN sed -i "/^\s*except requests\.exceptions\.ConnectionError as e:/s/except requests\.exceptions\.ConnectionError as e:/except Exception as e:/g" $($MHCFLURRY_BIN/python -c "import site; print(site.getsitepackages()[0])")/vaxrank/report.py


ENTRYPOINT ["python", "run_snakemake.py"]

