# Build with: docker build -f docker/Dockerfile -t pgv001_gpu_para .

##########################
# Base image: Parabricks
##########################
FROM nvcr.io/nvidia/clara/clara-parabricks:4.6.0-1

LABEL maintainer="Julia Kodysh <julia326@gmail.com>"

##########################
# System dependencies
##########################
RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections && \
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
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

##########################
# Create non-root user and prepare environment
##########################

# 1. Create user 'user' with sudo privileges
RUN useradd --create-home --home-dir /home/user --shell /bin/bash -G sudo user && \
    echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# 2. Set environment variables and working directory
ENV HOME=/home/user
ENV SHELL=/bin/bash
WORKDIR /home/user

# 3. Create .snakemake directory as root
RUN mkdir -p /home/user/.snakemake && \
    echo "Created .snakemake directory:" && \
    ls -ld /home/user/.snakemake && \
    chown -R user:user /home/user/.snakemake && \
    echo "Ownership set to user:user"

# 4. Switch to non-root user
USER user

##########################
# Parabricks path
##########################
ENV PATH=/opt/parabricks/bin:$PATH

##########################
# Install vcftools
##########################
RUN cd /tmp && \
    wget https://github.com/openvax/neoantigen-vaccine-pipeline/releases/download/0.3.0/vcftools_0.1.13.tar.gz && \
    tar xvfz vcftools_0.1.13.tar.gz && cd vcftools_0.1.13/ && make && \
    sudo cp -r /tmp/vcftools_0.1.13/bin/* /usr/local/bin && \
    sudo cp -r /tmp/vcftools_0.1.13/lib/perl5/site_perl/* /usr/local/lib && \
    rm -rf /tmp/*
ENV PERL5LIB=/usr/local/lib

##########################
# Install Miniconda
##########################
RUN wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh && \
    bash miniconda.sh -b -p $HOME/miniconda
ENV PATH=$HOME/miniconda/bin:$PATH
ENV PGV_BIN=$HOME/miniconda/envs/pgv/bin
ENV JAVA7_BIN=$HOME/miniconda/envs/java7/bin

# Use libmamba solver
RUN conda config --set always_yes yes --set changeps1 no && \
    conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main && \
    conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r && \
    conda update -n base conda && \
    conda install -n base conda-libmamba-solver && \
    conda config --set solver libmamba

##########################
# Install PGV environment
##########################
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
        vcftools==0.1.15 && \
    conda clean --all -y

##########################
# MHCflurry environment
##########################
RUN conda create -n mhcflurry \
        python=3.10 \
        setuptools \
        cython \
        pip && \
    conda clean --all -y
ENV MHCFLURRY_BIN=$HOME/miniconda/envs/mhcflurry/bin

##########################
# Java 7 environment
##########################
RUN conda create -n java7 java-jdk==7.0.91 && conda clean --all -y

##########################
# MuTect & GATK
##########################
RUN wget -O $HOME/mutect-1.1.7.jar http://storage.googleapis.com/common-files/tools/mutect-1.1.7.jar
ENV MUTECT=$HOME/mutect-1.1.7.jar
RUN wget -O $HOME/GenomeAnalysisTK.jar http://storage.googleapis.com/common-files/tools/GenomeAnalysisTK.jar
RUN $PGV_BIN/gatk-register $HOME/GenomeAnalysisTK.jar

##########################
# Install Python requirements
##########################
COPY requirements.txt $HOME
RUN $PGV_BIN/pip install --upgrade pip setuptools wheel
RUN $PGV_BIN/pip install -r $HOME/requirements.txt
RUN $MHCFLURRY_BIN/python -m pip install --upgrade pip setuptools wheel
RUN $MHCFLURRY_BIN/python -m pip install "mhcflurry[tensorflow]"
RUN $MHCFLURRY_BIN/mhcflurry-downloads fetch
RUN $MHCFLURRY_BIN/mhcflurry-predict --alleles HLA-A0201 --peptides SIINFEKL


ENV PATH=$PGV_BIN:$MHCFLURRY_BIN:$PATH


##########################
# Vaxrank & Snakemake setup
##########################
ENV KERAS_BACKEND=tensorflow
ENV PYENSEMBL_CACHE_DIR=/reference-genome/pyensembl-cache
ENV VAXRANK_REF_PEPTIDES_DIR=/reference-genome/reference-peptides
RUN curl https://pastebin.com/raw/AmfYN3er > $HOME/.fonts.conf

COPY . $HOME

##########################
# Strelka
##########################
RUN wget https://github.com/Illumina/strelka/releases/download/v2.9.10/strelka-2.9.10.centos6_x86_64.tar.bz2 && \
    tar -xvf strelka-2.9.10.centos6_x86_64.tar.bz2 && \
    mv strelka-2.9.10.centos6_x86_64 $HOME/strelka && \
    rm strelka-2.9.10.centos6_x86_64.tar.bz2

ENV STRELKA_CONFIG=$HOME/pipeline/strelka_config.txt
ENV STRELKA_BIN=$HOME/strelka/bin
ENV SCRIPTS=$HOME/pipeline/scripts

ENV USER=user
RUN mkdir -p $HOME/.snakemake && chown -R $USER:$USER $HOME/.snakemake

# Patch Vaxrank report.py
RUN sed -i "/^\s*except requests\.exceptions\.ConnectionError as e:/s/except requests\.exceptions\.ConnectionError as e:/except Exception as e:/g" \
    $($PGV_BIN/python -c "import site; print(site.getsitepackages()[0])")/vaxrank/report.py

##########################
# Entrypoint
##########################
ENTRYPOINT ["python", "run_snakemake.py"]


