## はじめに

拙作の[`FACE01`](https://github.com/yKesamaru/FACE01_DEV)は日本人専用顔学習モデルを備えたオープンソースの顔認証フレームワークです。

https://github.com/yKesamaru/FACE01_DEV

![](https://raw.githubusercontent.com/yKesamaru/Clean_Up_the_Dockerfiles/refs/heads/master/assets/screenshot.png)

環境構築をしないでも使えるようにDockerイメージを[DockerHub](https://hub.docker.com/u/tokaikaoninsho)に用意しています。

さてバージョンが上がるたびにDockerイメージを作り直すのですが、自動化しているスクリプトやDockerfileの運用実績はあるものの非効率なコードなため、タイミング的にちょうどよいと思いリファクタリングすることになりました。

自動化スクリプト: `FACE01_DEV/make_DockerImages.sh`
Dockerfile: `FACE01_DEV/docker/face01_gpu`と`FACE01_DEV/docker/face01_no_gpu`

計3つのコードのリファクタリングを行います。

`FACE01_DEV/docker/face01_gpu`はNVIDIA GPUを備えたPC用のイメージ、

`FACE01_DEV/docker/face01_no_gpu`はCPUのみで動作するPC用のイメージです。

さきほどブランチをマージしたので、簡単な説明をつけて共有をします。

対象となる読者はDockerfileのリファクタリングを考えてる方です。対象範囲がかなり狭いですが、よろしくおねがいします。

![](https://raw.githubusercontent.com/yKesamaru/Clean_Up_the_Dockerfiles/refs/heads/master/assets/eye-catch.png)

## 環境
```bash
$ inxi -SG --filter
System:
  Kernel: 6.8.0-49-generic x86_64 bits: 64 Desktop: GNOME 42.9
    Distro: Ubuntu 22.04.5 LTS (Jammy Jellyfish)
Graphics:
  Device-1: NVIDIA TU116 [GeForce GTX 1660 Ti] driver: nvidia v: 555.42.06
  Display: x11 server: X.Org v: 1.21.1.4 driver: X: loaded: nvidia
    unloaded: fbdev,modesetting,nouveau,vesa gpu: nvidia
    resolution: 2560x1440~60Hz
  OpenGL: renderer: NVIDIA GeForce GTX 1660 Ti/PCIe/SSE2
    v: 4.6.0 NVIDIA 555.42.06
```

## dlibのビルドをするかどうか
`dlib`は`pip`でインストールした場合、うまくインストールできない場合があります。（原因は不明です）

ビルドに必要なヘッダファイルやライブラリ（`Python.h` ヘッダーファイルなど）が余計に必要になるため、Dockerイメージを小さくシンプルにしたいならマルチステージビルドを検討したほうが良いです。

ソースからビルドする場合、線形代数の計算に使用するBLASライブラリが不足すると`dlib`はビルトインのBLASライブラリを使用しようとします。組み込み版のBLASは最適化されておらず、計算速度が遅くなる可能性があるとのことです。

## 2種類のイメージを作成し、Docker Hubへのプッシュを自動化するスクリプト
`FACE01_DEV/make_DockerImages.sh`

自分用に用意してあるbash用のテンプレートを使った形ですが、変数の設定などひと塊にしたコードにします。

結果的に`TAG=3.03.04`のように、修正箇所が一箇所になるので人為的ミスが減りそうです。

### Before
```bash: FACE01_DEV/make_DockerImages.sh
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
```bash: FACE01_DEV/make_DockerImages.sh
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
`FACE01_DEV/make_DockerImages.sh`から呼び出される`dockerfile`は2種類存在します。
1. `FACE01_DEV/docker/Dockerfile_gpu`: NVIDIA GPUを積んだPC用のDocker Imageを作るdockerfile
2. `FACE01_DEV/docker/Dockerfile_no_gpu`: CPUのみで動作するPC用のDocker Imageを作るdockerfile

それぞれ`dockerfile`の修正前・修正後を転載します。

### Before: Dockerfile_gpu
```bash: FACE01_DEV/docker/Dockerfile_gpu
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

### After: Dockerfile_gpu
RUNコマンドをまとめることでレイヤーの数を減らしました。

RUNは減らせば減らすほどよい、というものではありません。

Dockerfileの各命令（RUN, COPY, ADDなど）は、実行されるたびに新しいレイヤーを作成します。1つのレイヤー内でインストールとキャッシュの削除を行えば削除されたキャッシュサイズ分だけ容量を削減できます。

基本的にインストールやコピーをしたらその容量分がレイヤーの容量となりますが、レイヤーは変更差分を記録する仕組みなので、あるレイヤーで生成されたファイルを次のレイヤーで削除しても、その削除は新しいレイヤーの差分として記録され、元のレイヤーのデータは残ります。

つまり同じレイヤーで削除をしない限りレイヤーごとの容量は変わりません。

Dockerはレイヤー単位でキャッシュを管理します。レイヤーが多いとキャッシュチェックや再利用に時間がかかりますが、レイヤーを減らせばその負担が軽減され、ビルドにかかる時間の短縮に繋がります。

逆に過度にレイヤーを削減すると可読性が落ちます。それどころかレイヤーを減らしすぎると、特定の箇所だけを修正した場合でも全体を再ビルドする必要が出てくることがあります（キャッシュが有効活用できない）。またエラーが起きた場合、特定が難しくなります。

Dockerfile作成の最初期はなるべくレイヤーを分割し、様子を見ます。Dockerfileに問題がないようならレイヤー数を減らす方向で修正をし、ビルド時間の短縮やイメージ容量の削減を図ります。

最終的にイメージ容量を気にするならマルチステージビルドを検討すべきです。

この段階では各命令を整理しレイヤー数を削減する修正を行いました。

`dlib`のビルドに必要なファイルのため、`Dockerfile_gpu`では`devパッケージ`を採用し、反対に`dlib`を`pip`からインストールする`Dockerfile_no_gpu`では`devパッケージ`を削除しました。

またユーザー権限設定の整理と不要な`sudo`の使用を削除しました。

```bash: FACE01_DEV/docker/Dockerfile_gpu
# ベースイメージの選択
FROM nvidia/cuda:11.6.1-runtime-ubuntu20.04
LABEL maintainer="y.kesamaru@tokai-kaoninsho.com"

# 環境変数の設定
## USE_GPU: Docker_INSTALL_FACE01.shで使用
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=ja_JP.UTF-8 \
    TZ=Asia/Tokyo \
    USE_GPU=1 \
    USER_HOME=/home/docker

# 必要なパッケージのインストールとロケール設定
RUN apt-get update && apt-get install -y \
    # dlib build
    build-essential \
    cmake \
    python3-dev \
    libopenblas-dev \
    liblapack-dev \
    libboost-all-dev \
    libeigen3-dev \
    libx11-dev \
    pkg-config \
    libcudnn8 \
    libcudnn8-dev \
    # 必要なツール
    wget \
    sudo \
    supervisor \
    git \
    vim \
    gedit \
    # Python関連
    python3-tk \
    python3-pip \
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
COPY --chown=docker:docker face01lib /home/docker/FACE01_DEV/face01lib
COPY --chown=docker:docker output /home/docker/FACE01_DEV/output
COPY --chown=docker:docker noFace /home/docker/FACE01_DEV/noFace
COPY --chown=docker:docker multipleFaces /home/docker/FACE01_DEV/multipleFaces
COPY --chown=docker:docker example /home/docker/FACE01_DEV/example
COPY --chown=docker:docker assets /home/docker/FACE01_DEV/assets
COPY --chown=docker:docker preset_face_images /home/docker/FACE01_DEV/preset_face_images
COPY --chown=docker:docker --chmod=0755 docker /home/docker/FACE01_DEV/docker
COPY --chown=docker:docker pyproject.toml requirements_dev.txt config.ini dlib-19.24.tar.bz2 /home/docker/FACE01_DEV/

# スクリプトのコピーと実行
# COPY --chmod=0755 docker/Docker_INSTALL_FACE01.sh ${USER_HOME}/FACE01_DEV/
RUN /bin/bash -c "${USER_HOME}/FACE01_DEV/docker/Docker_INSTALL_FACE01.sh"
```

### Before: Dockerfile_no_gpu
```bash: FACE01_DEV/docker/Dockerfile_no_gpu
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

### After: Dockerfile_no_gpu

変更点は`Dockerfile_gpu`と同様です。

```bash: FACE01_DEV/docker/Dockerfile_no_gpu
FROM ubuntu:20.04
LABEL maintainer="y.kesamaru@tokai-kaoninsho.com"

# 環境変数の設定
## USE_GPU: Docker_INSTALL_FACE01.shで使用
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=ja_JP.UTF-8 \
    TZ=Asia/Tokyo \
    USE_GPU=0 \
    USER_HOME=/home/docker

# 必要なパッケージのインストールとロケール設定
RUN apt-get update && apt-get install -y \
    pkg-config \
    cmake \
    build-essential \
    python3-dev \
    libboost-all-dev \
    libopenblas-dev \
    liblapack-dev \
    libx11-dev \
    libgtk-3-dev \
    libjpeg-dev \
    libpng-dev \
    libtiff-dev \
    libavcodec-dev \
    libavformat-dev \
    libswscale-dev \
    # 必要なツール
    wget \
    sudo \
    supervisor \
    git \
    vim \
    gedit \
    # Python関連
    python3-tk \
    python3-pip \
    python3-venv \
    # メディア関連
    ffmpeg \
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
COPY --chown=docker:docker face01lib /home/docker/FACE01_DEV/face01lib
COPY --chown=docker:docker output /home/docker/FACE01_DEV/output
COPY --chown=docker:docker noFace /home/docker/FACE01_DEV/noFace
COPY --chown=docker:docker multipleFaces /home/docker/FACE01_DEV/multipleFaces
COPY --chown=docker:docker example /home/docker/FACE01_DEV/example
COPY --chown=docker:docker assets /home/docker/FACE01_DEV/assets
COPY --chown=docker:docker preset_face_images /home/docker/FACE01_DEV/preset_face_images
COPY --chown=docker:docker --chmod=0755 docker /home/docker/FACE01_DEV/docker
COPY --chown=docker:docker pyproject.toml requirements_dev.txt config.ini dlib-19.24.tar.bz2 /home/docker/FACE01_DEV/

# スクリプトのコピーと実行
# COPY --chmod=0755 docker/Docker_INSTALL_FACE01_CPU.sh ${USER_HOME}/FACE01_DEV/
RUN /bin/bash -c "${USER_HOME}/FACE01_DEV/docker/Docker_INSTALL_FACE01_CPU.sh"
RUN pip uninstall -y onnxruntime-gpu && pip install onnxruntime
```

## さいごに
結果的に作成されたコードは`v3.03.04`としてマージしました。成果物はDockerHubの[こちらのページ](https://hub.docker.com/u/tokaikaoninsho)にアップしてあります。

今回のリファクタリングでバグを潰しイメージサイズも小さくなりました。

以上です。ありがとうございました。