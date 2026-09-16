# simple-java-maven-app

本仓库 Fork 自官方教程示例 [jenkins-docs/simple-java-maven-app](https://github.com/jenkins-docs/simple-java-maven-app)，用于在本地用 **Docker 运行 Jenkins**，跑通 Maven 构建、测试、交付与产物归档。

官方教程原文：[Build a Java app with Maven](https://www.jenkins.io/doc/tutorials/build-a-java-app-with-maven/)。

---

## 完整部署方案

→ **[docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)**  
（含国内镜像加速、实战踩坑、[回退手册](docs/DEPLOYMENT.md#16-回退手册)）

---

## 仓库内容

| 路径 | 说明 |
|------|------|
| `Jenkinsfile` | 构建**本应用**的流水线（Docker agent / temurin-21） |
| `src/` / `pom.xml` | 示例 Java 应用 |
| `jenkins/scripts/` | 官方 Deliver 脚本（与控制器目录无关） |
| `jenkins-controller/` | **Jenkins 控制器**：compose / Dockerfile / 插件清单 |
| `docs/DEPLOYMENT.md` | 完整部署文档 |

---

## 快速开始

### 1. 启动 Jenkins 控制器

```bash
cd jenkins-controller
docker compose up -d --build
```

- UI：http://localhost:8081  
- 密码：见 [`jenkins-controller/README.md`](jenkins-controller/README.md)  
- 详细步骤：[`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md)

> Volume（V1）：在本目录首次 `up` 时，一般会创建新的 named volume（如 `jenkins-controller_jenkins_home`）。  
> 若要沿用旧的 `jenkins-demo_jenkins_home`，见部署文档备份/volume 说明。

### 2. 创建 Pipeline 任务

- SCM：`https://github.com/xiaoyvcheng/simple-java-maven-app.git`  
- Branch：`*/master`  
- Script Path：`Jenkinsfile`  

### 3. Build Now

查看阶段视图、Test Result、Artifacts（`*.jar`）。

流水线**不**在 Jenkins 全局工具里配 JDK/Maven；版本由 `Jenkinsfile` 镜像决定。

---

## 构建成功后能看到什么

Deliver 一次性执行 `java -jar ...`（打印问候语）后退出，**不是**常驻 Web 服务。  
验证看 Console / Test Result / Build Artifacts。

---

## 许可证

见 [LICENSE.txt](LICENSE.txt)。
