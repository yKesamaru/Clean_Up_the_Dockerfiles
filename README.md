## はじめに
2種類のDockerイメージ（`face01_gpu`と`face01_no_gpu`）を作成します。
1. 2種類のイメージを作成し、Docker Hubへのプッシュを自動化するスクリプト
2. 2種類のdockerfile
それぞれをブラッシュアップします。
具体的には、開発段階の、見通しが悪くて非効率なコードをリファクタリングします。

![](assets/eye-catch.png)

## 2種類のイメージを作成し、Docker Hubへのプッシュを自動化するスクリプト

自分用に用意してあるbash用のテンプレートを使った形ですが、変数の設定などひと塊にしたコードにします。

### Before
```bash
#!/usr/bin/env bash
set -Ceux -o pipefail
IFS=$'\n\t'

# -----------------------------------------------------------------
# サマリー:
# このスクリプトは、DockerイメージのビルドおよびDocker Hubへのプッシュを自動化します。
# face01_gpuとface01_no_gpuの2種類のイメージを作成し、それぞれをDocker Hubにプッシュします。
# -----------------------------------------------------------------

function my_command() {
    # cd: FACE01_DEV/
    cd ~/bin/FACE01_DEV

    # ////////////////////////////////////////
    # face01_gpu
    # ////////////////////////////////////////

    # docker build: CPU100%になるので他の作業との兼ね合いに注意すること
    docker build -t tokaikaoninsho/face01_gpu:3.03.04 -f docker/Dockerfile_gpu . --network host
    # login
    docker login
    # docker push
    docker push tokaikaoninsho/face01_gpu:3.03.04

    # ////////////////////////////////////////
    # face01_no_gpu
    # ////////////////////////////////////////

    # docker build: CPU100%になるので他の作業との兼ね合いに注意すること
    docker build -t tokaikaoninsho/face01_no_gpu:3.03.04 -f docker/Dockerfile_no_gpu . --network host
    # login
    docker login
    # docker push
    docker push tokaikaoninsho/face01_no_gpu:3.03.04

    return 0
}

function my_error() {
    zenity --error --text="\
    失敗しました。
    "
    exit 1
}

my_command || my_error
```

### After
```bash
#!/usr/bin/env bash

: <<'DOCSTRING'
このスクリプトは、2種類のDockerイメージ（GPU対応版と非対応版）をビルドし、
Docker Hubにプッシュするプロセスを自動化します。

- ビルド対象:
    1. face01_gpu
    2. face01_no_gpu
- 主な操作:
    - Dockerイメージのビルド
    - Docker Hubへのログイン
    - Dockerイメージのプッシュ
- 注意:
    - ビルド中にCPU使用率が高くなるため、他の作業への影響を考慮してください。
DOCSTRING

set -euo pipefail
IFS=$'\n\t'

# 定数設定
WORKDIR=~/bin/FACE01_DEV  # 作業ディレクトリ
DOCKER_REPO=tokaikaoninsho
TAG=3.03.04

# Dockerイメージをビルドしてプッシュする関数
build_and_push_image() {
    local image_name=$1   # イメージ名
    local dockerfile=$2   # 使用するDockerfile

    echo "Docker imageをビルドします: ${image_name}:${TAG}"
    docker build -t "${DOCKER_REPO}/${image_name}:${TAG}" -f "${dockerfile}" . --network host

    echo "Docker imageを発行します: ${DOCKER_REPO}/${image_name}:${TAG}"
    # ログイン
    docker login
    docker push "${DOCKER_REPO}/${image_name}:${TAG}"
}

# メイン処理
main() {
    # 作業ディレクトリに移動
    cd "${WORKDIR}"

    # GPU対応版のイメージ
    build_and_push_image "face01_gpu" "docker/Dockerfile_gpu"

    # 非GPU対応版のイメージ
    build_and_push_image "face01_no_gpu" "docker/Dockerfile_no_gpu"
}

# エラー時の処理
error_handler() {
    echo "エラーが発生しました。" >&2
    exit 1
}

# 実行
trap error_handler ERR
main
```

## 2種類のdockerfile
### Before: Dockerfile_gpu
```bash
# -----------------------------------------------------------------
# サマリー:
# このDockerfileは、CUDA 11.0およびUbuntu 20.04をベースにしたface01_gpuイメージをビルドします。
# ユーザーの追加、必要なパッケージのインストール、環境設定を行い、Docker_INSTALL_FACE01.shスクリプトを実行してセットアップを行います。
# -----------------------------------------------------------------

# ベースイメージ: https://hub.docker.com/r/nvidia/cuda/tags
## ubuntu20.04: https://hub.docker.com/r/nvidia/cuda/tags?page=1&page_size=&name=ubuntu20.04&ordering=
## cuda11: https://hub.docker.com/r/nvidia/cuda/tags?page=&page_size=&ordering=&name=11
# FROM nvidia/cuda:11.0.3-base-ubuntu20.04
# 開発版。全てのパッケージを含む。: https://hub.docker.com/layers/nvidia/cuda/11.6.1-devel-ubuntu20.04/images/sha256-2732cd07ffb34d73621a12de8ecea08a1caa7b87d43ccbca34112de189824c20?context=explore
# 軽量なベースイメージを選択
FROM nvidia/cuda:11.6.1-runtime-ubuntu20.04

LABEL maintainer="y.kesamaru@tokai-kaoninsho.com"

ENV DEBIAN_FRONTEND=noninteractive
ENV LANG=ja_JP.UTF-8
ENV TZ=Asia/Tokyo

# deb http://archive.canonical.com/ubuntu focal partnerを有効化
RUN sed -i "s@# deb http://archive.canonical.com/ubuntu focal partner@deb http://archive.canonical.com/ubuntu focal partner@" /etc/apt/sources.list

# CUDA環境作成
RUN apt-get update
# すでに鍵はインストールされているので下記は不要
# RUN apt-get install -y wget
# RUN wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-keyring_1.0-1_all.deb
# RUN dpkg -i cuda-keyring_1.0-1_all.deb
# RUN apt-get update
# RUN apt-get install -y nvidia-cuda-toolkit
RUN apt-get install -y libcudnn8
RUN apt-get install -y libcudnn8-dev

# その他をインストール
RUN apt-get install -y wget
RUN apt-get install -y software-properties-common
RUN apt-get install -y apt-utils
RUN apt-get install -y sudo
RUN apt-get install -y supervisor
RUN apt-get install -y build-essential
RUN apt-get install -y cmake
RUN apt-get install -y ffmpeg
RUN apt-get install -y fonts-mplus
RUN apt-get install -y git
RUN apt-get install -y libavcodec-dev
RUN apt-get install -y libavformat-dev
RUN apt-get install -y libswscale-dev
RUN apt-get install -y libx11-dev
RUN apt-get install -y python3-dev
RUN apt-get install -y python3-tk
RUN apt-get install -y python3-pkg-resources
RUN apt-get install -y python3-venv
RUN apt-get install -y liblapack-dev
RUN apt-get install -y libopenblas-dev
RUN apt-get install -y language-pack-ja
RUN apt-get install -y fonts-noto-cjk
RUN apt-get install -y vim
RUN apt-get install -y libsm6
RUN apt-get install -y libxext6
RUN apt-get install -y libxrender-dev
RUN apt-get install -y gedit
RUN apt-get install -y firefox
RUN rm -rf /var/lib/apt/lists/*


# add user
ARG DOCKER_UID=1000
ARG DOCKER_USER=docker
ARG DOCKER_PASSWORD=docker

RUN useradd -m \
  --uid ${DOCKER_UID} --groups sudo,video --shell /bin/bash ${DOCKER_USER} \
  && echo ${DOCKER_USER}:${DOCKER_PASSWORD} | chpasswd

WORKDIR /home/${DOCKER_USER}
RUN chown -R ${DOCKER_USER} ./
USER ${DOCKER_USER}

# Install FACE01
RUN mkdir /home/${DOCKER_USER}/FACE01_DEV
WORKDIR /home/${DOCKER_USER}/FACE01_DEV/

COPY face01lib /home/${DOCKER_USER}/FACE01_DEV/face01lib
COPY output /home/${DOCKER_USER}/FACE01_DEV/output
COPY noFace /home/${DOCKER_USER}/FACE01_DEV/noFace
# COPY images /home/${DOCKER_USER}/FACE01_DEV/images
COPY pyproject.toml /home/${DOCKER_USER}/FACE01_DEV/
COPY requirements_dev.txt /home/${DOCKER_USER}/FACE01_DEV/
COPY config.ini /home/${DOCKER_USER}/FACE01_DEV/

## Folder newly prepared from v1.4.09
COPY docs /home/${DOCKER_USER}/FACE01_DEV/docs
COPY example /home/${DOCKER_USER}/FACE01_DEV/example
# COPY tests /home/${DOCKER_USER}/FACE01_DEV/tests
COPY assets /home/${DOCKER_USER}/FACE01_DEV/assets

## Folders and files obsolete from v1.4.09
# COPY FACE01.py /home/${DOCKER_USER}/FACE01_DEV/
# COPY CALL_FACE01.py /home/${DOCKER_USER}/FACE01_DEV/
# COPY *.mp4 /home/${DOCKER_USER}/FACE01_DEV/

# COPY dlib-19.24.tar.bz2 /home/${DOCKER_USER}/FACE01_DEV/
COPY SystemCheckLock /home/${DOCKER_USER}/FACE01_DEV/
RUN echo ${DOCKER_PASSWORD} | sudo -S chown -R ${DOCKER_USER} /home/${DOCKER_USER}/FACE01_DEV
COPY preset_face_images /home/${DOCKER_USER}/FACE01_DEV/preset_face_images

COPY docker/Docker_INSTALL_FACE01.sh /home/${DOCKER_USER}/FACE01_DEV/
WORKDIR /home/${DOCKER_USER}/FACE01_DEV
RUN echo ${DOCKER_PASSWORD} | sudo -S chown ${DOCKER_USER} /home/${DOCKER_USER}/FACE01_DEV/Docker_INSTALL_FACE01.sh \
    && chmod +x /home/${DOCKER_USER}/FACE01_DEV/Docker_INSTALL_FACE01.sh \
    && /bin/bash -c /home/${DOCKER_USER}/FACE01_DEV/Docker_INSTALL_FACE01.sh \

# `--clean` see bellow
# [Have you done sudo python3 setup.py install --clean yet?](https://github.com/davisking/dlib/issues/1686#issuecomment-471509357)
```

RUNコマンドをまとめることでレイヤーの数を減らし、イメージサイズを削減しました。# Dockerfileでは、各`RUN`コマンドや他のコマンドが実行されるたびに、新しいレイヤーが作成されます。レイヤーが増えると、ビルド時のディスク使用量が増え、Dockerイメージのサイズが大きくなります。そのため、複数の`RUN`コマンドを一つにまとめることでレイヤーの数を減らし、結果としてイメージサイズを小さく保つことができます。また、これにより、イメージの作成と配布が効率化されます。

# ベストプラクティス改善点:
# 1. `RUN`コマンドをまとめて一度に実行することでレイヤーを減らし、イメージサイズを削減。
# 2. 不要なキャッシュファイルを削除してイメージの軽量化を行った。
# 3. `--no-install-recommends`オプションを使って余分な依存関係のインストールを防止。
# 4. ユーザー権限設定の整理と不要な`sudo`の使用を削減。

# パッケージ変更リスト:
# - libavcodec-dev -> libavcodec58
# - libavformat-dev -> libavformat58
# - libswscale-dev -> libswscale5
# - libx11-dev -> libx11-6
# - python3-dev -> python3
# - liblapack-dev -> liblapack3
# - libopenblas-dev -> libopenblas-base
# - libxrender-dev -> libxrender1

```bash: gpu
# ベースイメージの選択
FROM nvidia/cuda:11.6.1-runtime-ubuntu20.04
LABEL maintainer="y.kesamaru@tokai-kaoninsho.com"

# 環境変数の設定
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=ja_JP.UTF-8 \
    TZ=Asia/Tokyo \
    USER_HOME=/home/docker

# 必要なパッケージのインストールとロケール設定
RUN apt-get update && apt-get install -y \
    # CUDA関連
    libcudnn8 \
    # 線形代数関連
    liblapack3 \
    libopenblas-base \
    # dlib build
    build-essential \
    cmake \
    # 必要なツール
    wget \
    sudo \
    supervisor \
    git \
    vim \
    gedit \
    # Python関連
    python3 \
    python3-tk \
    python3-pkg-resources \
    python3-venv \
    # メディア関連
    ffmpeg \
    libavcodec58 \
    libavformat58 \
    libswscale5 \
    # X11ライブラリ関連
    libx11-6 \
    libsm6 \
    libxext6 \
    libxrender1 \
    # 言語サポート・フォント関連
    language-pack-ja \
    fonts-mplus \
    fonts-noto-cjk \
    locales tzdata && \
    locale-gen ja_JP.UTF-8 && \
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# ユーザー追加
ARG DOCKER_UID=1000
ARG DOCKER_USER=docker
ARG DOCKER_PASSWORD=docker
RUN useradd -m --uid ${DOCKER_UID} --home-dir ${USER_HOME} --groups sudo,video --shell /bin/bash ${DOCKER_USER} \
    && echo ${DOCKER_USER}:${DOCKER_PASSWORD} | chpasswd \
    && chown -R ${DOCKER_USER}:${DOCKER_USER} ${USER_HOME}

# ユーザーに切り替え
USER ${DOCKER_USER}
WORKDIR ${USER_HOME}/FACE01_DEV

# 必要なフォルダとファイルをコピー
COPY face01lib output noFace pyproject.toml requirements_dev.txt config.ini \
     example assets preset_face_images \
     ${USER_HOME}/FACE01_DEV/

# スクリプトのコピーと実行
COPY --chmod=+x docker/Docker_INSTALL_FACE01.sh ${USER_HOME}/FACE01_DEV/
RUN /bin/bash -c ${USER_HOME}/FACE01_DEV/Docker_INSTALL_FACE01.sh
```

### before
```bash
FROM ubuntu:20.04
LABEL maintainer="y.kesamaru@tokai-kaoninsho.com"

# ########################################################
# GPU **CANNOT** be used for the docker image
# created by this docker file.
# Therefore, the execution speed is slow and the GUI cannot
# be displayed.
# If you want native processing speed, build using
# 'Dockerfile_gpu'.
# ########################################################

ENV DEBIAN_FRONTEND noninteractive

RUN sed -i "s@# deb http://archive.canonical.com/ubuntu focal partner@deb http://archive.canonical.com/ubuntu focal partner@" /etc/apt/sources.list
RUN apt-get update \
    && apt-get install -y \
	software-properties-common \
	apt-utils \
	sudo \
	supervisor \
    build-essential \
    cmake \
    ffmpeg \
    fonts-mplus \
    git \
    libavcodec-dev \
    libavformat-dev \
    libswscale-dev \
    python3-dev \
    python3-tk \
    python3-pkg-resources \
    python3-venv \
    wget \
    language-pack-ja \
    fonts-noto-cjk \
    vim \
	&& rm -rf /var/lib/apt/lists/*

# configure
RUN locale-gen ja_JP.UTF-8
ENV LANG=ja_JP.UTF-8
ENV TZ=Asia/Tokyo
# add user
ARG DOCKER_UID=1000
ARG DOCKER_USER=docker
ARG DOCKER_PASSWORD=docker
RUN useradd -m \
  --uid ${DOCKER_UID} --groups sudo,video --shell /bin/bash ${DOCKER_USER} \
  && echo ${DOCKER_USER}:${DOCKER_PASSWORD} | chpasswd

WORKDIR /home/${DOCKER_USER}
RUN chown -R ${DOCKER_USER} ./
USER ${DOCKER_USER}

# Install FACE01
RUN mkdir /home/${DOCKER_USER}/FACE01_DEV
WORKDIR /home/${DOCKER_USER}/FACE01_DEV/

COPY face01lib /home/${DOCKER_USER}/FACE01_DEV/face01lib
COPY output /home/${DOCKER_USER}/FACE01_DEV/output
COPY noFace /home/${DOCKER_USER}/FACE01_DEV/noFace
# COPY images /home/${DOCKER_USER}/FACE01_DEV/images
COPY pyproject.toml /home/${DOCKER_USER}/FACE01_DEV/
COPY requirements_dev.txt /home/${DOCKER_USER}/FACE01_DEV/
COPY config.ini /home/${DOCKER_USER}/FACE01_DEV/

## Folder newly prepared from v1.4.09
COPY docs /home/${DOCKER_USER}/FACE01_DEV/docs
COPY example /home/${DOCKER_USER}/FACE01_DEV/example
# COPY tests /home/${DOCKER_USER}/FACE01_DEV/tests
COPY assets /home/${DOCKER_USER}/FACE01_DEV/assets

## Folders and files obsolete from v1.4.09
# COPY FACE01.py /home/${DOCKER_USER}/FACE01_DEV/
# COPY CALL_FACE01.py /home/${DOCKER_USER}/FACE01_DEV/
# COPY *.mp4 /home/${DOCKER_USER}/FACE01_DEV/

# COPY dlib-19.24.tar.bz2 /home/${DOCKER_USER}/FACE01_DEV/
COPY SystemCheckLock /home/${DOCKER_USER}/FACE01_DEV/
RUN echo ${DOCKER_PASSWORD} | sudo -S chown -R ${DOCKER_USER} /home/${DOCKER_USER}/FACE01_DEV
COPY preset_face_images /home/${DOCKER_USER}/FACE01_DEV/preset_face_images

COPY docker/Docker_INSTALL_FACE01_CPU.sh /home/${DOCKER_USER}/FACE01_DEV/
WORKDIR /home/${DOCKER_USER}/FACE01_DEV
RUN echo ${DOCKER_PASSWORD} | sudo -S chown ${DOCKER_USER} /home/${DOCKER_USER}/FACE01_DEV/Docker_INSTALL_FACE01_CPU.sh \
    && chmod +x /home/${DOCKER_USER}/FACE01_DEV/Docker_INSTALL_FACE01_CPU.sh \
    && /bin/bash -c /home/${DOCKER_USER}/FACE01_DEV/Docker_INSTALL_FACE01_CPU.sh

# `--clean` see bellow
# [Have you done sudo python3 setup.py install --clean yet?](https://github.com/davisking/dlib/issues/1686#issuecomment-471509357)
```

### after
```bash: cpu
FROM ubuntu:20.04
LABEL maintainer="y.kesamaru@tokai-kaoninsho.com"

# 環境変数の設定
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=ja_JP.UTF-8 \
    TZ=Asia/Tokyo \
    USER_HOME=/home/docker

# 必要なパッケージのインストールとロケール設定
RUN apt-get update && apt-get install -y \
    # 線形代数関連
    liblapack3 \
    libopenblas-base \
    # dlib build
    build-essential \
    cmake \
    # 必要なツール
    wget \
    sudo \
    supervisor \
    git \
    vim \
    gedit \
    # Python関連
    python3 \
    python3-tk \
    python3-pkg-resources \
    python3-venv \
    # メディア関連
    ffmpeg \
    libavcodec58 \
    libavformat58 \
    libswscale5 \
    # X11ライブラリ関連
    libx11-6 \
    libsm6 \
    libxext6 \
    libxrender1 \
    # 言語サポート・フォント関連
    language-pack-ja \
    fonts-mplus \
    fonts-noto-cjk \
    locales tzdata && \
    locale-gen ja_JP.UTF-8 && \
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# ユーザー追加
ARG DOCKER_UID=1000
ARG DOCKER_USER=docker
ARG DOCKER_PASSWORD=docker
RUN useradd -m --uid ${DOCKER_UID} --home-dir ${USER_HOME} --groups sudo,video --shell /bin/bash ${DOCKER_USER} \
    && echo ${DOCKER_USER}:${DOCKER_PASSWORD} | chpasswd \
    && chown -R ${DOCKER_USER}:${DOCKER_USER} ${USER_HOME}

# ユーザーに切り替え
USER ${DOCKER_USER}
WORKDIR ${USER_HOME}/FACE01_DEV

# 必要なファイルをコピー
COPY face01lib output noFace pyproject.toml requirements_dev.txt config.ini \
     example assets preset_face_images \
     ${USER_HOME}/FACE01_DEV/

# スクリプトのコピーと実行
COPY --chmod=+x docker/Docker_INSTALL_FACE01_CPU.sh ${USER_HOME}/FACE01_DEV/
RUN /bin/bash -c ${USER_HOME}/FACE01_DEV/Docker_INSTALL_FACE01_CPU.sh

```