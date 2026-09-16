# simple-java-maven-app

本仓库 Fork 自官方教程示例 [jenkins-docs/simple-java-maven-app](https://github.com/jenkins-docs/simple-java-maven-app)，用于在本地用 **Docker 运行 Jenkins**，跑通 Maven 构建、测试、交付与产物归档。

官方教程原文：[Build a Java app with Maven](https://www.jenkins.io/doc/tutorials/build-a-java-app-with-maven/)。

---

## 完整部署方案

逐步安装 Jenkins、配置流水线、排障与回退，见：

→ **[docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)**

其中包含：国内镜像加速、实战踩坑、[回退手册](docs/DEPLOYMENT.md#16-回退手册) 等。

---

## 仓库内容

| 路径 | 说明 |
|------|------|
| `Jenkinsfile` | Docker agent 流水线（`maven:*-temurin-21`） |
| `src/` / `pom.xml` | 示例 Java 应用（控制台输出问候语） |
| `jenkins/scripts/deliver.sh` | Deliver 阶段脚本 |
| `docs/DEPLOYMENT.md` | **Jenkins + Docker 完整部署文档** |

---

## 和 Jenkins 怎么配合

1. 本机用平台包（如 `jenkins-demo`）执行 `docker compose up -d --build` 启动 Jenkins（默认 UI：http://localhost:8081）  
2. Jenkins 新建 **Pipeline** 任务，SCM 指向本仓库：

   `https://github.com/xiaoyvcheng/simple-java-maven-app.git`

   - Branch：`*/master`  
   - Script Path：`Jenkinsfile`  

3. **Build Now**，查看阶段视图、测试报告与 `target/*.jar` 产物  

流水线 **不** 在 Jenkins「全局工具」里配 JDK/Maven；版本由 `Jenkinsfile` 中的容器镜像决定。

---

## 构建成功后能看到什么

Deliver 会执行一次 `java -jar ...`（例如打印 `Hello World!`），进程随后退出。

这是 **CI 演示**，不是常驻 Web 服务。验证方式：

- Console Output（Deliver 日志）  
- Test Result  
- Build Artifacts 中的 jar  

---

## 许可证

见 [LICENSE.txt](LICENSE.txt)。
