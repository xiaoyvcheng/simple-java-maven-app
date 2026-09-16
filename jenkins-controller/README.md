# Jenkins 控制器（Docker）

本目录用于在本机启动 Jenkins 控制器，**不是**构建 Java 应用的流水线定义。

- 流水线：仓库根目录 [`Jenkinsfile`](../Jenkinsfile)
- 完整部署文档：[`docs/DEPLOYMENT.md`](../docs/DEPLOYMENT.md)

## 启动

```bash
cd jenkins-controller
docker compose up -d --build
```

初始密码：

```bash
# Git Bash
MSYS_NO_PATHCONV=1 docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

浏览器：http://localhost:8081

## 常用命令

```bash
docker compose ps
docker compose logs -f jenkins
docker compose down          # 停容器，保留 volume
docker compose down -v       # 停容器并删除 volume（慎用）
```

## 注意

- 请只从**一处**启动（本目录或旧的 `jenkins-demo`），避免抢 `container_name: jenkins` / 端口 8081。
- 官方示例里的 `jenkins/scripts/` 是 Deliver 脚本，与本目录无关。
- Volume 策略见部署文档（V1：本目录首次 up 多为新 volume）。
