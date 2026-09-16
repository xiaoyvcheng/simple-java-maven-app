# Jenkins + Docker 完整部署方案

> 本文档属于业务仓库 [xiaoyvcheng/simple-java-maven-app](https://github.com/xiaoyvcheng/simple-java-maven-app)（Fork 自 [jenkins-docs/simple-java-maven-app](https://github.com/jenkins-docs/simple-java-maven-app)）。  
> 目标：在 Docker 中部署 Jenkins，对本仓库跑通「拉代码 → Maven 构建 → 单元测试 → 交付脚本 → 归档产物」。  
> 适用：Windows（Docker Desktop）/ macOS / Linux 本地或单机服务器。  
>  
> **职责划分**  
> - **本仓库根目录**：业务代码、`Jenkinsfile`（构建本应用）  
> - **本仓库 `jenkins-controller/`**：`docker-compose.yml` / `Dockerfile` / `plugins.txt`（启动 Jenkins 控制器）  
> - **本仓库 `jenkins/scripts/`**：官方 Deliver 脚本（与控制器目录无关）  
> - 本机若仍保留旧目录 `jenkins-demo`，仅作对照；**以本仓库 `jenkins-controller/` 为准**

---

## 目录

1. [方案概述](#1-方案概述)
2. [架构与设计决策](#2-架构与设计决策)
3. [前置条件](#3-前置条件)（含 [国内镜像加速](#331-国内-docker-镜像加速本机实测可用)）
4. [仓库文件说明](#4-仓库文件说明)
5. [部署 Jenkins](#5-部署-jenkins)
6. [首次初始化](#6-首次初始化)
7. [准备业务仓库](#7-准备业务仓库)
8. [创建 Pipeline 任务](#8-创建-pipeline-任务)
9. [首次构建与验收](#9-首次构建与验收)
10. [（可选）GitHub Webhook 自动构建](#10-可选github-webhook-自动构建)
11. [运维手册](#11-运维手册)
12. [故障排查](#12-故障排查)
13. [安全建议](#13-安全建议)
14. [验收检查清单](#14-验收检查清单)
15. [实战踩坑记录](#15-实战踩坑记录)（本机跑通过程中的真实问题）
16. [回退手册](#16-回退手册)

---



## 1. 方案概述


| 项           | 选择                                                     |
| ----------- | ------------------------------------------------------ |
| CI 平台       | Jenkins LTS（官方镜像 `lts-jdk21`，内置 Java 21）               |
| 运行方式        | Docker Compose 单节点                                     |
| 构建方式        | Docker Pipeline：`maven:3.9.9-eclipse-temurin-21` 容器内执行 |
| 示例项目        | 本仓库 `xiaoyvcheng/simple-java-maven-app`（Fork 自官方示例） |
| 流水线定义       | 仓库根目录 `Jenkinsfile`（Pipeline script from SCM）          |
| 数据持久化       | Docker named volume：`jenkins_home`                     |
| Docker 调用方式 | 挂载宿主机 `/var/run/docker.sock`（Docker-out-of-Docker）     |


**不采用**「在 Manage Jenkins → Tools 里配 `jdk17` / `maven3`」的方式，避免工具链与节点环境耦合。JDK/Maven 版本由 Jenkinsfile 中的镜像 tag 决定。

---



## 2. 架构与设计决策



### 2.1 拓扑

```text
┌─────────────────────────────────────────────────────────────┐
│  宿主机（Windows Docker Desktop / Linux）                    │
│                                                             │
│  ┌──────────────────────┐      docker.sock                  │
│  │ Jenkins 容器         │◄────────────────────────────────┐ │
│  │ :8080  Web UI        │                                 │ │
│  │ :50000 Agent 端口    │     ┌───────────────────────┐   │ │
│  │ + docker CLI         │────►│ maven:3.9-temurin-21  │───┘ │
│  │ volume: jenkins_home │     │ mvn clean/package/test│     │
│  └──────────┬───────────┘     └───────────────────────┘     │
│             │ git clone                                      │
└─────────────┼────────────────────────────────────────────────┘
              ▼
     GitHub：https://github.com/xiaoyvcheng/simple-java-maven-app
```



### 2.2 为什么这样设计


| 决策                      | 原因                                   |
| ----------------------- | ------------------------------------ |
| Jenkins 跑在 Docker 里     | 一键起停、易重建、环境可复现                       |
| 构建用 Maven 容器            | 不污染 Jenkins 镜像；换 JDK/Maven 只改 tag    |
| 挂载 `docker.sock`        | 复用宿主机 Docker，无需 DinD 特权嵌套            |
| 自定义镜像预装 docker CLI + 插件 | 减少「docker: not found」和手工装插件          |
| `user: root`            | 演示环境访问 sock 最省事（生产见[安全建议](#13-安全建议)） |




### 2.3 流水线阶段


| Stage        | 命令 / 动作                            | 产出                     |
| ------------ | ---------------------------------- | ---------------------- |
| Build        | `mvn -B -DskipTests clean package` | `target/*.jar`         |
| Test         | `mvn test` + `junit`               | Surefire / Test Result |
| Deliver      | `./jenkins/scripts/deliver.sh`     | 交付脚本日志                 |
| post success | `archiveArtifacts`                 | 可下载的 jar               |


---



## 3. 前置条件



### 3.1 软件


| 软件             | 要求                   | 说明                                                |
| -------------- | -------------------- | ------------------------------------------------- |
| Docker         | 24+ 推荐               | Windows/macOS 用 Docker Desktop                    |
| Docker Compose | v2（`docker compose`） | Desktop 已内置                                       |
| Git            | 任意近期版本               | 克隆 / 推送 Fork 仓库                                   |
| 浏览器            | Chrome / Edge 等      | 访问 [http://localhost:8081](http://localhost:8081) |
| GitHub 账号      | 可 Fork 公开仓库          | 私有库需 PAT                                          |




### 3.2 资源建议


| 资源  | 最低       | 推荐                       |
| --- | -------- | ------------------------ |
| CPU | 2 核      | 4 核                      |
| 内存  | 4 GB     | 8 GB（Jenkins + Maven 容器） |
| 磁盘  | 20 GB 空闲 | 40 GB+（镜像与构建缓存）          |




### 3.3 网络

- 能拉取 Docker Hub 镜像：`jenkins/jenkins`、`maven`
- 能访问 GitHub（克隆示例仓库）
- 国内网络若直连 Docker Hub 超时或极慢，先配置镜像加速（见下）



#### 3.3.1 国内 Docker 镜像加速（本机实测可用）

> 验证环境：本机 Windows + Docker Desktop，约 2026-09-15。  
> 当时直连 `registry-1.docker.io` **超时**；下列三个源均可完成 `hello-world` manifest 拉取。


| 镜像源                            | 说明       | 本机实测                                 |
| ------------------------------ | -------- | ------------------------------------ |
| `https://docker.m.daocloud.io` | DaoCloud | 可用；大层下载约 10MB/s（本机实测更稳）              |
| `https://docker.1ms.run`       | 毫秒镜像     | 可用；manifest 快，部分大层可能很慢               |
| `https://docker.xuanyuan.me`   | 轩辕镜像     | 可用（作 `registry-mirrors`）；按路径直拉可能 403 |


**勿再配置（本机验证失败或已停服）：**


| 地址                                   | 结果                  |
| ------------------------------------ | ------------------- |
| `https://hub-mirror.c.163.com`       | DNS 失败（网易公共加速已停）    |
| `https://docker.mirrors.ustc.edu.cn` | DNS 失败（中科大已停同步）     |
| `https://mirror.ccs.tencentyun.com`  | 本机不可达（多仅腾讯云内网）      |
| `https://registry.docker-cn.com`     | 连接超时                |
| 阿里云 `*.mirror.aliyuncs.com`          | 需控制台专属地址，且常限阿里云 ECS |


**Docker Desktop 配置步骤：**

1. 打开 **Settings → Docker Engine**
2. 在 JSON 中合并（保留你已有的其它字段）：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me"
  ]
}
```

1. 点击 **Apply & Restart**
2. 验证：

```bash
docker info
docker pull hello-world
```

`docker info` 中应出现 `Registry Mirrors` 列表；`hello-world` 能正常拉取即表示加速生效。

> 说明：`registry-mirrors` 主要加速 **Docker Hub（docker.io）**。  
> 拉 `ghcr.io`、`registry.k8s.io` 等需另配对应代理，不在本方案范围内。  
> 公益/第三方镜像站可能变更，若日后失效，用同样方式对 `/v2/` 与小镜像 pull 做复测后再改配置。



### 3.4 端口


| 端口    | 用途                        | 冲突处理                                                     |
| ----- | ------------------------- | -------------------------------------------------------- |
| 8081  | Jenkins Web UI（映射容器 8080） | 本仓库默认 `8081:8080`，避免与本机其他占用 8080 的容器冲突；可改回 `"8080:8080"` |
| 50000 | 入站 Agent（本方案暂不用）          | 可保留或删除映射                                                 |


---



## 4. 仓库文件说明

文档所在仓库（`simple-java-maven-app`）同时包含 **业务代码 + 流水线** 与 **Jenkins 控制器部署文件**。

```text
simple-java-maven-app/
├── Jenkinsfile                 # 构建本应用（SCM）
├── pom.xml / src/
├── jenkins/scripts/            # 官方 Deliver 脚本（勿与控制器目录混淆）
├── jenkins-controller/         # Jenkins 控制器
│   ├── docker-compose.yml
│   ├── Dockerfile
│   ├── plugins.txt
│   └── README.md
├── README.md
└── docs/
    └── DEPLOYMENT.md           # 本文档
```


| 文件 | 作用 |
| ---- | ---- |
| `jenkins-controller/docker-compose.yml` | 启动 Jenkins；持久化 `jenkins_home`；挂载 sock |
| `jenkins-controller/Dockerfile` | lts-jdk21 + `docker-cli` |
| `jenkins-controller/plugins.txt` | 插件清单 |
| `Jenkinsfile`（仓库根目录） | Docker agent 流水线，SCM 直接读取 |


---



## 5. 部署 Jenkins



### 5.1 获取本仓库

请进入本仓库的 **`jenkins-controller`** 子目录（不要在业务根目录盲跑 compose）：

```bash
cd /path/to/simple-java-maven-app/jenkins-controller
```

Windows Git Bash / PowerShell 示例：

```bash
cd C:/Users/bing/Desktop/python/jenkins-demo/.cache/simple-java-maven-app/jenkins-controller
# 或 clone 后：cd /path/to/simple-java-maven-app/jenkins-controller
```



### 5.2 构建并启动

```bash
docker compose up -d --build
```

首次会：

1. 构建 `jenkins-controller:lts-jdk21-docker`（含 docker CLI，耗时数分钟）
2. 创建 named volume（V1：在 `jenkins-controller` 目录首次 up，常见名为 `jenkins-controller_jenkins_home`；以 `docker volume ls` 为准）
3. 以后台方式启动容器 `jenkins`



### 5.3 确认容器状态

```bash
docker compose ps
docker compose logs -f jenkins
```

期望：

- `STATE` 为 `running`
- 日志出现 Jenkins 启动完成相关信息（如 `Jenkins is fully up and running`）

读取初始管理员密码：

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

复制输出的一串字符，供下一步登录使用。

### 5.4 停止 / 重启 / 销毁

```bash
# 停止（保留 volume 与配置）
docker compose stop

# 启动
docker compose start

# 重启
docker compose restart

# 删除容器但保留数据 volume
docker compose down

# 删除容器且删除数据（危险：清空 Jenkins 配置与历史构建）
docker compose down -v
```

---



## 6. 首次初始化

1. 浏览器打开：**[http://localhost:8081](http://localhost:8081)**

（本仓库默认把 Jenkins 映射到 **8081**，避免与本机其他占用 8080 的容器冲突。）
2. 粘贴 **initialAdminPassword**
3. 插件安装：

- 推荐选 **Install suggested plugins**
- 本镜像已通过 `plugins.txt` 预装 Docker Pipeline、Git、JUnit 等；向导装一套也不冲突  
- **不要**再装 Blue Ocean（已官方弃用）

1. 创建管理员用户（记住用户名与密码）
2. Instance Configuration：URL 保持 `http://localhost:8081/` 即可 → Save and Finish → Start using Jenkins



### 6.1 确认关键插件

**Manage Jenkins → Plugins → Installed**，确认至少存在：


| 插件                  | 用途                         |
| ------------------- | -------------------------- |
| Docker Pipeline     | `agent { docker { ... } }` |
| Git                 | 从 SCM 拉代码                  |
| Pipeline            | 声明式流水线                     |
| JUnit               | 测试报告                       |
| Workspace Cleanup   | `cleanWs()`                |
| Timestamper         | 日志时间戳                      |
| Pipeline Stage View | 阶段可视化（替代已弃用的 Blue Ocean）   |


若缺失 Docker Pipeline：Available 中搜索安装后重启 Jenkins（或 `docker compose restart`）。

### 6.2 本方案无需配置全局工具

**不要**再去 Tools 里配 `jdk17` / `maven3`。  
构建环境由 Jenkinsfile 中的 Maven 镜像提供。

---



## 7. 准备业务仓库（本仓库）

> 你正在阅读的仓库 **已经是** Fork 后的业务仓。若从官方仓重新 Fork，仍可按下列步骤操作；本仓可跳过 7.1，确认根目录 `Jenkinsfile` 为 Docker agent 版即可。



### 7.1 Fork

1. 打开 [https://github.com/jenkins-docs/simple-java-maven-app](https://github.com/jenkins-docs/simple-java-maven-app)
2. 点击右上角 **Fork**，复制到自己的账号
3. 记下地址，例如：

```text
https://github.com/xiaoyvcheng/simple-java-maven-app.git
```

> 官方仓库默认分支一般为 `master`（不是 `main`）。创建任务时 Branch Specifier 用 `*/master`。



### 7.2 覆盖 Docker 版 Jenkinsfile

```bash
git clone https://github.com/xiaoyvcheng/simple-java-maven-app.git
cd simple-java-maven-app

# 根目录已有 Jenkinsfile；一般无需再从别处覆盖
# cp /path/to/other/Jenkinsfile ./Jenkinsfile

git add Jenkinsfile
git commit -m "ci: use Docker agent for Maven build on Jenkins"
git push origin master
```



### 7.3 Jenkinsfile 行为说明

流水线关键片段：

```groovy
agent {
    docker {
        image 'maven:3.9.9-eclipse-temurin-21'
        args '-v $HOME/.m2:/root/.m2'
    }
}
```

含义：

- Jenkins 通过宿主机 Docker 启动临时 Maven 容器  
- 工作区挂载进该容器执行 `mvn`  
- `$HOME/.m2` 挂载用于缓存依赖，加速二次构建

完整文件见仓库根目录 `Jenkinsfile`。

---



## 8. 创建 Pipeline 任务

1. Jenkins 首页 → **New Item**
2. 名称：`simple-java-maven-app`（可自定义，建议与仓库名一致）
3. 类型选 **Pipeline** → **OK**
4. 滚动到 **Pipeline** 区域，按表填写：


| 配置项              | 值                                                      |
| ---------------- | ------------------------------------------------------ |
| Definition       | Pipeline script from SCM                               |
| SCM              | Git                                                    |
| Repository URL   | `https://github.com/xiaoyvcheng/simple-java-maven-app.git` |
| Credentials      | 公开库可留空；失败则添加 GitHub PAT（见下）                            |
| Branch Specifier | `*/master`                                             |
| Script Path      | `Jenkinsfile`                                          |


1. **Save**



### 8.1 （按需）添加 GitHub 凭据

若出现认证失败或 API 限流：

1. GitHub → Settings → Developer settings → Personal access tokens
  - Fine-grained 或 classic均可；classic 至少勾选 `repo`（公开只读可更窄）
2. Jenkins → Manage Jenkins → Credentials → （global）→ Add Credentials
  - Kind：Username with password  
  - Username：GitHub 用户名  
  - Password：PAT
3. 回到任务配置，Credentials 选刚添加的项

---



## 9. 首次构建与验收



### 9.1 触发构建

打开任务 `simple-java-maven-app` → **Build Now**。

首次构建会：

1. 克隆 Git 仓库
2. 拉取 `maven:3.9.9-eclipse-temurin-21`（可能较久）
3. 下载 Maven 依赖
4. 执行 Build → Test → Deliver



### 9.2 查看结果


| 入口                            | 期望                                                      |
| ----------------------------- | ------------------------------------------------------- |
| 任务页构建历史                       | 蓝色球 / SUCCESS                                           |
| Console Output                | 三阶段均成功；无 `docker: not found` / `mvn: command not found` |
| Test Result                   | 测试通过（官方示例含简单单元测试）                                       |
| Build Artifacts               | 可下载 `target/*.jar`                                      |
| Pipeline Stage View / Console | 阶段与完整日志                                                 |




### 9.3 成功标准

- [ ] Jenkins 容器稳定运行，8080 可访问  
- [ ] Pipeline 使用 Docker agent 拉起 Maven 镜像  
- [ ] Build / Test / Deliver 全部 SUCCESS  
- [ ] JUnit 报告可见  
- [ ] jar 已归档  

---



## 10. （可选）GitHub Webhook 自动构建

本地 `localhost` 无法被 GitHub 直接回调，任选其一：


| 方式                                | 适用    |
| --------------------------------- | ----- |
| 内网穿透（ngrok / Cloudflare Tunnel 等） | 本地学习  |
| 公网 IP / 域名反代到 8080                | 服务器部署 |




### 10.1 Jenkins 侧

1. 安装插件 **GitHub** / **GitHub Integration**（`plugins.txt` 已含 `github`）
2. 任务配置 → **Build Triggers** →勾选 **GitHub hook trigger for GITScm polling** → Save



### 10.2 GitHub 侧

仓库 → Settings → Webhooks → Add webhook：


| 项            | 值                                   |
| ------------ | ----------------------------------- |
| Payload URL  | `https://<公网或隧道域名>/github-webhook/` |
| Content type | `application/json`                  |
| Events       | Just the push event                 |
| Active       | 勾选                                  |


Push 一次代码后，在 Webhook Recent Deliveries 与 Jenkins 构建历史中确认是否自动触发。

---



## 11. 运维手册



### 11.1 常用命令

```bash
docker compose up -d --build   # 构建并启动
docker compose ps              # 状态
docker compose logs -f jenkins # 日志
docker compose restart         # 重启
docker compose down            # 停止并删容器（保留 volume）
```



### 11.2 备份 Jenkins 数据

数据在 named volume 中（名称可用 `docker volume ls | grep jenkins` 确认）：

**Volume 策略（V1）**

- 在 `jenkins-controller/` 下执行 `docker compose up` 时，compose 使用本地名 `jenkins_home`，Docker 通常创建 **`<项目目录名>_jenkins_home`**（常见：`jenkins-controller_jenkins_home`）。
- 这与早期在 `jenkins-demo/` 目录启动时的 `jenkins-demo_jenkins_home` **不是同一个 volume**（除非你改过配置）。
- **沿用旧数据**：可临时把 `docker-compose.yml` 中 volumes 改为已有外部卷，例如：

```yaml
volumes:
  jenkins_home:
    external: true
    name: jenkins-demo_jenkins_home
```

- **接受新 volume**：直接 up 即可（等于新的一份 Jenkins 家目录）。
- **不要**同时在 `jenkins-demo` 与 `jenkins-controller` 两处 `compose up`（会抢 `container_name: jenkins` / 端口）。

```bash
# 备份（在 jenkins-controller 目录执行，便于 $(pwd) 落在该目录）
# Windows Git Bash 必须加 MSYS_NO_PATHCONV=1，否则 /backup 会被改写成
# C:/Program Files/Git/backup/... 导致失败
MSYS_NO_PATHCONV=1 docker run --rm \
  -v jenkins-controller_jenkins_home:/data \
  -v "$(pwd)":/backup \
  alpine tar czf /backup/jenkins_home_backup.tgz -C /data .
```

PowerShell / Linux / macOS 可去掉行首的 `MSYS_NO_PATHCONV=1`。

### 11.3 恢复

```bash
docker compose down
MSYS_NO_PATHCONV=1 docker run --rm \
  -v jenkins-controller_jenkins_home:/data \
  -v "$(pwd)":/backup \
  alpine sh -c "cd /data && tar xzf /backup/jenkins_home_backup.tgz"
docker compose up -d
```



### 11.4 升级 Jenkins 镜像（JDK / 安全补丁）

本仓库默认基础镜像为 `jenkins/jenkins:lts-jdk21`（当前验证：Jenkins **2.568.3** + Java **21**）。

升级步骤：

1. 建议先备份 volume（见上一节）
2. 确认 `Dockerfile` 中 `FROM jenkins/jenkins:lts-jdk21`（或更具体的版本 tag）
3. 国内网络建议先用 DaoCloud/crane 拉取基础镜像并 `docker tag` 为 `jenkins/jenkins:lts-jdk21`
4. 执行：

```bash
docker compose up -d --build
```

1. 登录后打开 **Manage Jenkins**，确认：
  - 不再提示 Java 17 / 核心版本过旧
  - **Plugins** → 如仍有 Updates，点 **Download now and install after restart** 后重启
2. 也可用容器内命令批量更新插件后重启：

```bash
docker exec jenkins sh -c 'ls /var/jenkins_home/plugins/*.jpi | xargs -n1 basename | sed "s/\\.jpi$//" > /tmp/plist.txt && jenkins-plugin-cli --plugin-file /tmp/plist.txt --plugin-download-directory /var/jenkins_home/plugins --latest true'
docker compose restart jenkins
```

> 说明：Jenkins 控制器的 Java 与流水线里 Maven 镜像的 JDK **无关**。业务构建需与项目 `pom.xml` 的 enforcer 一致（本示例要求 **JDK 21+**，对应 `maven:*-temurin-21`）。



### 11.5 磁盘清理

```bash
# 清理无用镜像/容器（勿在生产忙时误删仍需缓存）
docker system df
docker image prune
docker system prune
```

流水线已配置 `buildDiscarder(numToKeepStr: '10')`，限制 Jenkins 侧历史构建数量。

### 11.6 查看本机 Maven 缓存与构建容器

构建时临时容器由 Docker Pipeline 创建，结束后一般会删除。  
依赖缓存来自挂载的 `.m2` 目录（Jenkins 容器内用户 home 下）。

---



## 12. 故障排查


| 现象                                              | 可能原因                                     | 处理                                                                          |
| ----------------------------------------------- | ---------------------------------------- | --------------------------------------------------------------------------- |
| 8080 / 8081 打不开                                 | 容器未起 / 端口占用                              | `docker compose ps`、`logs`；本机若已有服务占 8080，用 `8081:8080`                      |
| 忘记管理员密码                                         | —                                        | 仍可用 `initialAdminPassword`（若向导未完成）；否则重置见 Jenkins 官方文档                       |
| `docker: not found`                             | 镜像未装 CLI，或装了 `docker.io` 却无 `docker-cli` | `jenkins-controller/Dockerfile` 使用 `docker-cli`；`compose up -d --build`                      |
| `permission denied` on docker.sock              | 权限不足                                     | 确认 compose 中 `user: root`；Linux 检查 sock 权限                                  |
| 无法 pull 镜像 / 极慢 / 某层卡住                          | 直连 Docker Hub 失败；部分加速源大层很慢               | 见 [3.3.1](#331-国内-docker-镜像加速本机实测可用)；优先 DaoCloud；可用 crane 拉取后 `docker load` |
| Git clone 失败                                    | 地址错 / 需认证 / 网络                           | 核对 URL；加 PAT；检查代理                                                           |
| `git push` 401 / Permission denied              | 本机 Git 登录了别的 GitHub 账号                   | 凭据管理器删除旧 `github.com` 凭据；用正确账号 + PAT 再推                                     |
| Branch not found                                | 分支名错误                                    | 该示例默认 `*/master`，勿写 `*/main`                                                |
| `mvn: command not found`                        | 误用 `agent any` 且未装 Maven                 | 确认已推送 Docker 版 Jenkinsfile                                                  |
| RequireJavaVersion `[21,)` 失败                   | 构建镜像仍是 JDK 17                            | Jenkinsfile 改为 `maven:*-eclipse-temurin-21`                                 |
| 阶段全绿但构建失败；`No artifacts found ... target/*.jar` | `post.always { cleanWs() }` 先于归档删掉产物     | 归档放 `success`，清理放 `cleanup`（见 [§15.8](#158-阶段全绿但整次失败no-artifacts-found)）    |
| Manage 页提示 Java 17 / 核心有漏洞                      | 控制器镜像过旧                                  | 升级 `FROM jenkins/jenkins:lts-jdk21` 后重建                                     |
| 大量 Blue Ocean 插件「已弃用」                           | 仍安装 blueocean 系列                         | 从 `plugins.txt` 与 `jenkins_home/plugins` 删除；重建勿再预装                          |
| 测试无报告                                           | surefire 路径不对或测试未跑                       | 看 Console 是否执行 `mvn test`；确认 `target/surefire-reports/*.xml`                |
| `deliver.sh` Permission denied                  | 无执行位                                     | Jenkinsfile 已含 `chmod +x`                                                   |
| 构建极慢                                            | 首次拉镜像与依赖                                 | 二次构建应明显加快；确认 `.m2` 挂载                                                       |
| Windows 下 sock 异常                               | Docker Desktop 未开                        | 启动 Desktop，确认 Linux engine 运行中                                              |
| 构建显示「没有变化」且很快失败；Console 有 `github.com:443` | Jenkins/本机访问不了 GitHub                     | 恢复网络或代理后再 Build；见 [§15.10](#1510-构建显示没有变化--连不上-github) |
| 构建成功但看不到「正在运行的项目」                              | Deliver 只是一次性跑 jar，不是常驻服务               | 见 [§15.11](#1511-构建成功但看不到运行中的项目)；看 Console / Artifacts 即可验证 |
| Git Bash 备份报 `C:/Program Files/Git/backup/...` | 路径被 MSYS 改写                               | 命令前加 `MSYS_NO_PATHCONV=1`；见 [§11.2](#112-备份-jenkins-数据) |




### 12.1 快速自检脚本

```bash
docker compose ps
docker exec jenkins docker version
docker exec jenkins docker images
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8081/login
```

期望：`docker version` 能连上 Engine；`login` 返回 `200` 或 `403`（均表示服务已起来）。

---



## 13. 安全建议

本方案面向 **本地学习 / 演示**。若用于共享或生产，请至少：

1. **不要**对公网裸奔暴露 8080；前面加 HTTPS 反代（Nginx / Caddy）并限制来源 IP
2. 避免长期 `user: root`：改用 docker 组 GID + `group_add`，以非 root 跑 Jenkins
3. 凭据只用最小权限 PAT，定期轮换；勿把 Token 写进仓库
4. 限制谁能配置 Pipeline（防任意 `sh` / docker 逃逸）
5. 定期备份 `jenkins_home`，并控制 volume 与镜像占用
6. 生产更推荐：独立 Agent、或 Kubernetes + Jenkins Controller，而非 Controller 直接挂宿主机 sock

---



## 14. 验收检查清单

部署完成后，按序勾选：

**平台**

- [x] `docker compose up -d --build` 成功  
- [x] [http://localhost:8081](http://localhost:8081) 可登录  
- [x] `docker exec jenkins docker version` 正常  
- [x] Docker Pipeline 等插件已安装  

**业务流水线**

- [x] 已 Fork `simple-java-maven-app`  
- [x] 已推送 Docker 版 `Jenkinsfile`  
- [x] Pipeline 任务 SCM / 分支 / Script Path 配置正确  
- [x] **Build Now** 成功（Build + Test + Deliver）  
- [ ] Test Result 与 Artifacts 可见  

**运维（建议）**

- [ ] 会执行 backup 命令  
- [ ] 知道 `compose down` 与 `down -v` 的区别  
- [ ] （可选）Webhook 推送可自动触发  

---



## 附录 A：端口修改示例

`jenkins-controller/docker-compose.yml`：

```yaml
ports:
  - "8081:8080"
  - "50000:50000"
```

访问改为 [http://localhost:8081](http://localhost:8081) 。

## 附录 B：与「全局工具」方案对比


|      | Tools：`jdk17` + `maven3` | 本方案：Docker agent |
| ---- | ------------------------ | ---------------- |
| 配置位置 | Manage Jenkins → Tools   | Jenkinsfile      |
| 节点依赖 | Jenkins 容器/Agent 需能装工具   | 只需 Docker        |
| 换版本  | 改全局工具并影响多任务              | 改镜像 tag，按仓库隔离    |
| 适用   | 传统静态 Agent               | 容器化 CI（推荐本 demo） |




## 附录 C：参考链接

- [Jenkins 官方 Docker 安装](https://www.jenkins.io/doc/book/installing/docker/)  
- [Docker Pipeline 插件](https://plugins.jenkins.io/docker-workflow/)  
- [示例仓库 simple-java-maven-app](https://github.com/jenkins-docs/simple-java-maven-app)  
- [声明式流水线语法](https://www.jenkins.io/doc/book/pipeline/syntax/)

---



## 15. 实战踩坑记录

> 来源：本机 Windows + Docker Desktop 跑通  
> `simple-java-maven-app`（仓库示例：`xiaoyvcheng/simple-java-maven-app`）全程记录。  
> 最终成功构建可见产物 `my-app-1.0-SNAPSHOT.jar`，阶段 Build / Test / Deliver 均为绿色。



### 15.1 直连 Docker Hub 超时，镜像拉不下来

**现象**

- `docker pull jenkins/jenkins:...` 长时间卡住或超时  
- `curl https://registry-1.docker.io/v2/` 连接超时

**原因**

国内网络访问 Docker Hub 不稳定。

**处理**

1. Docker Desktop → Settings → Docker Engine 配置 `registry-mirrors`（见 [§3.3.1](#331-国内-docker-镜像加速本机实测可用)）
2. **Apply & Restart**（仅改 `~/.docker/daemon.json` 有时不生效）
3. 大层下载：本机实测 **DaoCloud** 明显快于部分「manifest 很快、blob 很慢」的源
4. 仍卡住时用 crane 拉取再导入：

```bash
crane pull --platform linux/amd64 docker.m.daocloud.io/jenkins/jenkins:lts-jdk21 jenkins.tar
docker load -i jenkins.tar
docker tag docker.m.daocloud.io/jenkins/jenkins:lts-jdk21 jenkins/jenkins:lts-jdk21
```

---



### 15.2 本机 8080 已被占用

**现象**

- 启动 Jenkins 报端口绑定失败，或与其它容器（如已有 Java 应用）冲突

**处理**

`jenkins-controller/docker-compose.yml` 使用：

```yaml
ports:
  - "8081:8080"
```

访问 **[http://localhost:8081](http://localhost:8081)**。

---



### 15.3 容器内 `docker: not found`

**现象**

Pipeline 使用 `agent { docker { ... } }` 时报找不到 `docker` 命令。

**原因**

Debian 新版包装拆分：安装 `docker.io` 往往只有 `dockerd`，**不含**客户端；需要 `docker-cli`。

**处理**

Dockerfile 中：

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends docker-cli \
 && rm -rf /var/lib/apt/lists/*
```

并挂载 `/var/run/docker.sock`，`user: root`（演示环境）。

---



### 15.4 Manage 页：Java 17 过时、Jenkins 有安全告警

**现象**

- Manage Jenkins 提示控制器运行在不受支持 / 过时的 Java 17  
- 提示 Jenkins 核心存在安全漏洞、建议升级

**处理**

1. `Dockerfile`：`FROM jenkins/jenkins:lts-jdk21`
2. `docker compose up -d --build`（保留 `jenkins_home` volume）
3. 本机验证示例：Jenkins **2.568.3** + Java **21.0.x**
4. 插件：Manage → Plugins 更新，或用 `jenkins-plugin-cli --latest` 后重启

---



### 15.5 Blue Ocean 整组插件「已弃用」

**现象**

Installed 列表大量 Blue Ocean / Design Language 显示 Deprecated。

**处理**

1. `plugins.txt` **不要**再写 `blueocean`
2. 停止容器后清理 volume 内插件，再启动：

```bash
docker compose stop
docker run --rm -v jenkins-controller_jenkins_home:/var/jenkins_home alpine \
  sh -c 'rm -rf /var/jenkins_home/plugins/blueocean* /var/jenkins_home/plugins/jenkins-design-language*'
docker compose up -d --build
```

1. 若镜像曾 `jenkins-plugin-cli` 预装过 Blue Ocean，重建时需去掉该层，否则启动会从 `/usr/share/jenkins/ref/plugins` **再次拷回**

可视化可用 **Pipeline Stage View**（任务页阶段视图）。

---



### 15.6 `git push` 401 / Permission denied to 其他账号

**现象**

```text
remote: Permission to xiaoyvcheng/simple-java-maven-app.git denied to weibingc.
fatal: ... 403
# 或
fatal: 响应状态代码不指示成功: 401 (Unauthorized)
```

（有时先 401，凭据刷新后仍可能 push 成功，以最后是否出现 `master -> master` 为准。）

**原因**

Windows 凭据管理器里缓存了另一个 GitHub 账号。

**处理**

1. 凭据管理器 → Windows 凭据 → 删除 `git:https://github.com`
2. 再用仓库所有者账号 + **PAT**（不要用登录密码）推送
3. 或网页 Upload `Jenkinsfile`

---



### 15.7 Build 失败：JDK 17 不满足 `[21,)`

**现象**

```text
Rule 1: RequireJavaVersion failed
Detected JDK ... version 17.x which is not in the allowed range [21,).
```

阶段视图：Checkout 绿，Build / Test / Deliver 红。

**原因**

- 控制器用 Java 21 ≠ 构建容器 JDK  
- 项目 `pom.xml`（maven-enforcer）要求 JDK 21+  
- Jenkinsfile 仍写 `maven:3.9.9-eclipse-temurin-17`

**处理**

```groovy
agent {
    docker {
        image 'maven:3.9.9-eclipse-temurin-21'
        args '-v $HOME/.m2:/root/.m2'
    }
}
```

提交推送后重新 **Build Now**。

---



### 15.8 阶段全绿但整次失败：`No artifacts found`

**现象**

- Stage View 中 Build / Test / Deliver 均为绿色  
- 任务仍显示失败（红叉）  
- Console：

```text
[WS-CLEANUP] Deleting project workspace...
Archiving artifacts
'target/*.jar' doesn't match anything
ERROR: No artifacts found that match the file pattern "target/*.jar"
```

**原因**

Declarative `post` 中 `always` **先于** `success` **执行**。  
若写成：

```groovy
post {
    success { archiveArtifacts 'target/*.jar' }
    always  { cleanWs() }   // 先清空工作区 → 归档失败 → 整次 FAILURE
}
```

**处理**（本仓库 Jenkinsfile 已采用）：

```groovy
post {
    success {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
    failure {
        echo 'Pipeline failed — check Console Output'
    }
    cleanup {   // 在其它 post 之后执行
        cleanWs()
    }
}
```

---



### 15.9 成功验收对照（本机）


| 项       | 结果                                                                     |
| ------- | ---------------------------------------------------------------------- |
| Jenkins | [http://localhost:8081，`lts-jdk21`](http://localhost:8081，`lts-jdk21`) |
| 业务仓库    | 含 Docker 版 `Jenkinsfile`（temurin-21）                                   |
| 流水线     | Checkout → Build → Test → Deliver → Post 成功                            |
| 产物      | `my-app-1.0-SNAPSHOT.jar` 可下载                                          |
| 测试      | Test Result 趋势为通过                                                      |


常见失败序号对照：#1 JDK 版本不对；#2 手动中止；归档顺序错误会导致「阶段绿、结果红」；网络不通 GitHub 会导致 Checkout 失败且显示「没有变化」。

---

### 15.10 构建显示「没有变化」、很快失败：连不上 GitHub

**现象**

- 构建历史红叉，变更写「没有变化」
- Console 类似：

```text
git fetch ...
fatal: unable to access 'https://github.com/.../':
Failed to connect to github.com port 443 ... Could not connect to server
```

**原因**

Checkout 阶段就失败，根本没拉到新 commit，所以页面显示没有变化。  
与「代码没改」无关；本机/Jenkins 容器当时访问不了 GitHub。

**处理**

1. 浏览器能否打开对应仓库页面  
2. 恢复网络 / VPN / 代理后再点 **立即构建**  
3. 长期不通可为 Jenkins 配置 HTTP 代理（进阶，本 demo 未展开）

---

### 15.11 构建成功但看不到「运行中的项目」

**现象**

流水线全绿，有 jar 产物，但没有网页、没有常驻进程可访问。

**原因**

本示例 Deliver 阶段执行 `deliver.sh`：一次性 `java -jar ...`，打印 `Hello World!` / `Hello Jenkins!` 后进程退出。  
**不是**部署 Web 服务，也不会占用端口长期运行。

**如何验证成功**

- Console → Deliver 中的打印输出  
- Build Artifacts 中的 `my-app-1.0-SNAPSHOT.jar`  
- Test Result  

若要「构建后一直能访问」，需另加 Deploy 阶段（起容器、拷贝到服务器、K8s 等），超出本官方示例范围。

---

### 15.12 Git Bash 备份路径被改写

**现象**

```text
tar: can't open 'C:/Program Files/Git/backup/jenkins_home_backup.tgz': No such file or directory
```

**原因**

Git Bash 把容器路径 `/backup` 映射成了 Git 安装目录。

**处理**

见 [§11.2](#112-备份-jenkins-数据)：命令前加 `MSYS_NO_PATHCONV=1`。  
备份文件默认落在执行命令时的当前目录，例如 `jenkins-controller/jenkins_home_backup.tgz`。

---

## 16. 回退手册

出故障时先判断坏在哪一层，再选对应回退方式。

### 16.1 回退前先判断

| 症状 | 多半回退什么 |
|------|----------------|
| 某次改代码后构建红了 / 输出不对 | **GitHub 业务代码**（§16.2） |
| 误改任务配置、插件装挂、Jenkins 异常但 volume 还在 | **Jenkins 数据备份恢复**（§16.3 / §11.3） |
| 升级镜像后控制器起不来 | **Jenkins 镜像版本**（§16.4） |
| 误执行 `compose down -v` | 只能靠事先 `.tgz` 备份；无备份则配置丢失 |

本 demo **没有**长期运行的线上服务，因此「回退线上进程」不存在；代码回退 + 再构建即可验证。

### 16.2 GitHub 代码回退

流水线从 SCM 拉代码，**回退 = 仓库回到好版本，再 Jenkins 构建**。Jenkins 不会自动撤销 Git 提交。

#### 方式 A：网页 Revert（推荐、可追溯）

1. 打开业务仓库 → **Commits**  
2. 找到有问题的提交 → **Revert**  
3. 确认后 push/merge 到 `master`  
4. Jenkins **立即构建**

#### 方式 B：检出旧提交的文件 + 新 commit（不 force，本机曾用此法）

适合：回退到某个已知好的 SHA（例如 `2046a70...`），且不想改写远端历史。

```bash
cd /path/to/simple-java-maven-app
git checkout master
git pull origin master

# 把工作区文件恢复成目标提交中的内容（不移动分支指针）
git checkout 2046a70da8b14462b3da9bd6bb1b117c7d6af737 -- .

git add -A
git commit -m "rollback: restore tree to 2046a70"
git push origin master
```

然后 Jenkins **立即构建**。  
说明：会多一个 rollback 提交；历史仍保留中间错误提交，便于审计。

#### 方式 C：`git revert <commit>`

```bash
git log --oneline
git revert <要撤销的commit>
git push origin master
```

与网页 Revert 同类，生成「反向修改」的新提交。

#### 方式 D：`reset --hard` + force（不推荐）

```bash
git reset --hard <好的commit>
git push --force origin master   # 改写远端历史，多人协作禁用
```

仅个人练手仓库且明确知道后果时使用。本仓库文档流程**默认不用 force push**。

#### 回退成功怎么确认

1. GitHub 上打开关键文件（如 `App.java`）内容已是目标版本  
2. Jenkins 构建成功，Console Deliver 输出与代码一致  

### 16.3 Jenkins 数据回退

对应 [§11.2 备份](#112-备份-jenkins-数据) / [§11.3 恢复](#113-恢复)。

- 恢复的是整份 `jenkins_home`（任务、插件、用户、历史等）  
- **不会**自动修改 GitHub 上的源码  
- 与 §16.2 代码回退是两件事，可按需组合  

概念流程：`compose down` → 解压 `.tgz` 回 volume → `compose up -d`。  
切勿用 `down -v` 当回退（会删 volume）。

### 16.4 Jenkins 镜像版本回退

1. 建议先做 §11.2 备份  
2. 将 `Dockerfile` 的 `FROM` 改回上一可用 tag（如曾验证过的版本）  
3. `docker compose up -d --build`  
4. 仍异常再配合 §11.3 恢复数据  

注意：大版本写过的数据，旧镜像未必能读；关键升级前备份。

### 16.5 `compose down` / `down -v` 与回退

| 操作 | 是不是回退 | 说明 |
|------|------------|------|
| `docker compose down` | 否 | 停容器，保留 volume |
| `docker compose down -v` | **破坏** | 删除 volume，不是回退 |
| 恢复 `jenkins_home_backup.tgz` | 是 | 数据回退 |
| Git revert / checkout 旧树 + push + 构建 | 是 | 代码/流水线回退 |

### 16.6 建议应急顺序

```text
构建失败 / 输出不对
  └─ 看 Console：Git 拉代码？还是编译/测试/归档？
       ├─ github.com:443 超时 → 先修网络，不必回退代码
       ├─ 某次提交引入 → §16.2 回退代码 → 再构建
       └─ Jenkins 本身异常 → 停服务 → §11.3 恢复备份 → 再 up

大改配置 / 升级镜像之前
  └─ 先执行 §11.2 备份
```

---

文档版本：与本仓库 `jenkins-controller/`（Dockerfile / docker-compose.yml / plugins.txt）及根目录 `Jenkinsfile` 同步维护。  
变更配置后请同步更新本节与 README 快速入口。
