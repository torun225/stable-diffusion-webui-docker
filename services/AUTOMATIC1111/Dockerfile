# syntax=docker/dockerfile:1
FROM alpine/git:2.36.2 as download

COPY clone.sh /clone.sh


RUN . /clone.sh stable-diffusion-stability-ai https://github.com/Stability-AI/stablediffusion.git cf1d67a6fd5ea1aa600c4df58e5b47da45f6bdbf \
  && rm -rf assets data/**/*.png data/**/*.jpg data/**/*.gif

RUN . /clone.sh CodeFormer https://github.com/sczhou/CodeFormer.git c5b4593074ba6214284d6acd5f1719b6c5d739af \
  && rm -rf assets inputs

RUN . /clone.sh BLIP https://github.com/salesforce/BLIP.git 48211a1594f1321b00f14c9f7a5b4813144b2fb9
RUN . /clone.sh k-diffusion https://github.com/crowsonkb/k-diffusion.git ab527a9a6d347f364e3d185ba6d714e22d80cb3c
RUN . /clone.sh clip-interrogator https://github.com/pharmapsychotic/clip-interrogator 2cf03aaf6e704197fd0dae7c7f96aa59cf1b11c9
RUN . /clone.sh generative-models https://github.com/Stability-AI/generative-models 45c443b316737a4ab6e40413d7794a7f5657c19f


FROM pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime

ENV DEBIAN_FRONTEND=noninteractive PIP_PREFER_BINARY=1

# RUN echo "deb http://ftp.jp.debian.org/debian/ bullseye main contrib non-free" > /etc/apt/sources.list
# RUN echo "deb http://deb.debian.org/debian-security bullseye-security main" >> /etc/apt/sources.list
# RUN echo "deb http://ftp.jp.debian.org/debian bullseye-updates main contrib" >> /etc/apt/sources.list

RUN --mount=type=cache,target=/var/cache/apt   apt-get update && apt-get install -y libsepol-dev
RUN --mount=type=cache,target=/var/cache/apt \
  apt-get update && \
  apt-get upgrade -y
# we need those
RUN --mount=type=cache,target=/var/cache/apt apt-get install -y --fix-missing \
  fonts-dejavu-core rsync git jq moreutils aria2
# extensions needs those
RUN --mount=type=cache,target=/var/cache/apt apt-get install -y --fix-missing \
  ffmpeg libglfw3-dev libgles2-mesa-dev pkg-config
RUN --mount=type=cache,target=/var/cache/apt apt-get install -y --fix-missing \
  libcairo2
RUN --mount=type=cache,target=/var/cache/apt apt-get install -y --fix-missing \
  libcairo2-dev
RUN --mount=type=cache,target=/var/cache/apt apt-get install -y --fix-missing \
  build-essential

RUN --mount=type=cache,target=/cache --mount=type=cache,target=/root/.cache/pip \
  aria2c -x 5 --dir /cache --out torch-2.0.1-cp310-cp310-linux_x86_64.whl -c \
  https://download.pytorch.org/whl/cu118/torch-2.0.1%2Bcu118-cp310-cp310-linux_x86_64.whl && \
  pip install /cache/torch-2.0.1-cp310-cp310-linux_x86_64.whl torchvision --index-url https://download.pytorch.org/whl/cu118


WORKDIR /
RUN --mount=type=cache,target=/root/.cache/pip \
  git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git && \
  cd stable-diffusion-webui && \
  git reset --hard 4afaaf8a020c1df457bcf7250cb1c7f609699fa7 && \
  pip install -r requirements_versions.txt


ENV ROOT=/stable-diffusion-webui

COPY --from=download /repositories/ ${ROOT}/repositories/
RUN mkdir ${ROOT}/interrogate && cp ${ROOT}/repositories/clip-interrogator/clip_interrogator/data/* ${ROOT}/interrogate
RUN --mount=type=cache,target=/root/.cache/pip \
  pip install -r ${ROOT}/repositories/CodeFormer/requirements.txt

RUN --mount=type=cache,target=/root/.cache/pip \
  pip install pyngrok xformers \
  git+https://github.com/TencentARC/GFPGAN.git \
  git+https://github.com/openai/CLIP.git \
  git+https://github.com/mlfoundations/open_clip.git


COPY . /docker

RUN \
  # mv ${ROOT}/style.css ${ROOT}/user.css && \
  # one of the ugliest hacks I ever wrote \
  sed -i 's/in_app_dir = .*/in_app_dir = True/g' /opt/conda/lib/python3.10/site-packages/gradio/routes.py && \
  git config --global --add safe.directory '*'

# HTTPS認証/ControleNet
RUN pip install certipie opencv-python prettytable

WORKDIR ${ROOT}
ENV NVIDIA_VISIBLE_DEVICES=all
ENV CLI_ARGS=""
EXPOSE 7860
ENTRYPOINT ["/docker/entrypoint.sh"]
CMD python -u webui.py --listen --port 7860 ${CLI_ARGS}
