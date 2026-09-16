FROM jenkins/jenkins:lts-jdk21

USER root

# 安装 docker CLI，供 Pipeline 的 agent { docker } 调用宿主机 Docker Engine
RUN apt-get update \
    && apt-get install -y --no-install-recommends docker-cli ca-certificates curl \
    && rm -rf /var/lib/apt/lists/*

# 插件清单保留在镜像中；已有 jenkins_home 时以 volume 内插件为准
# 首次空 volume 部署可取消下一行注释预装，或启动后在 UI / plugin-cli 安装
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
# RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

USER root
