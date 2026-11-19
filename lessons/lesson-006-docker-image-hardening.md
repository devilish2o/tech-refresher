# Lesson 006 — Docker: build a smaller, non-root Java image

**Domain:** Docker / container build and runtime security  
**Duration:** 30 minutes  
**Environment:** Windows 11, PowerShell, with Docker Desktop already installed  
**Version assumptions:** Docker Engine 23+ (BuildKit default), Java 21, Maven 3.9 image. Official image tags checked 2026-11-17.

## Objective

Build a small Java application into a container with a multi-stage Dockerfile, then run it with restricted runtime settings.

Practise:
1. Separating the Maven/JDK build stage from the Java runtime stage.
2. Ordering instructions to preserve useful build-cache layers.
3. Excluding irrelevant files and secrets from the build context with `.dockerignore`.
4. Running as a non-root numeric UID/GID.
5. Using a read-only root filesystem, dropped Linux capabilities and no-new-privileges.
6. Verifying Maven is absent from the final image.

Use an existing Docker Desktop installation on Windows 11. Do not install another Docker engine or run this lab on a shared or production host. The lab uses a unique image tag, no host ports or volumes, and does not modify existing containers.

## 30-minute plan

| Time | Activity |
|---:|---|
| 0–3 min | Check Docker Desktop and create the isolated project |
| 3–10 min | Create the Java 21 application and Maven build |
| 10–17 min | Write the multi-stage Dockerfile and .dockerignore |
| 17–22 min | Build and inspect the image |
| 22–27 min | Run with restrictions and verify Maven is absent |
| 27–30 min | Clean up and self-check |
| **Total** | **30 minutes** |

## 1. Check Docker and create the lab (3 minutes)

Open **PowerShell on Windows 11** (not a WSL terminal).

```powershell
docker version
docker buildx version
```

Expected: Docker Client and Server sections are both present; Buildx prints a version. If the Server section is missing, start the already-installed Docker Desktop and retry. If Docker Desktop is not already installed or the engine remains unavailable, do not install it for this lesson: complete the source/Dockerfile review and leave the build/runtime checks pending until an existing disposable Docker engine is available.

Create the isolated project directory:

```powershell
$Lab = Join-Path $HOME "labs\tech-refresher\lesson-006-docker-hardening"
New-Item -ItemType Directory -Force -Path "$Lab\src\main\java\lab" | Out-Null
Set-Location $Lab
Get-Location
```

Expected path ends with `labs\tech-refresher\lesson-006-docker-hardening`. Use the same PowerShell window so `$Lab` remains defined.

## 2. Create a tiny Java application (7 minutes)

Create `pom.xml` with Vim:

```powershell
vim pom.xml
```

Paste:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>lab</groupId>
  <artifactId>orders-api</artifactId>
  <version>1.0.0</version>
  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  <build>
    <finalName>orders-api</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.14.1</version>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.4.2</version>
        <configuration>
          <archive><manifest><mainClass>lab.ReleaseInfo</mainClass></manifest></archive>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

Create the Java class:

```powershell
vim src/main/java/lab/ReleaseInfo.java
```

Paste:

```java
package lab;

public final class ReleaseInfo {
    public static void main(String[] args) {
        String environment = System.getenv().getOrDefault("APP_ENV", "not-set");
        String release = System.getenv().getOrDefault("RELEASE_ID", "not-set");

        System.out.printf(
            "app=orders-api env=%s release=%s java=%s%n",
            environment,
            release,
            Runtime.version()
        );
    }
}
```

The application reads environment-specific values at runtime; they are not baked into the image.

## 3. Create the multi-stage Dockerfile and .dockerignore (7 minutes)

Create `Dockerfile`:

```powershell
vim Dockerfile
```

Paste:

```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: build with Maven and a full JDK.
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /workspace
COPY pom.xml .
RUN mvn -B -q dependency:go-offline
COPY src ./src
RUN mvn -B -DskipTests package

# Stage 2: runtime only. Maven and the JDK build environment stay behind.
FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app
RUN groupadd --gid 10001 app && useradd --uid 10001 --gid 10001 --no-create-home --shell /usr/sbin/nologin app
COPY --from=build --chown=10001:10001 /workspace/target/orders-api.jar /app/orders-api.jar
USER 10001:10001
ENTRYPOINT ["java", "-jar", "/app/orders-api.jar"]
```

Create `.dockerignore`:

```powershell
vim .dockerignore
```

Paste:

```text
.git
.idea
.vscode
target
*.log
.env
.env.*
*.pem
*.key
*.p12
*.pfx
*.tfstate
*.tfstate.*
*.tfplan
```

Review the choices:
- The first stage uses Maven and a JDK to build the JAR.
- The final stage starts from a JRE image and copies only the JAR.
- Copying `pom.xml` before `src` lets source changes reuse the dependency-download layer.
- UID/GID 10001 is used instead of root.
- `.dockerignore` excludes irrelevant files and common secret-file formats. It is a safeguard, not a replacement for secret management.

## 4. Build and inspect the image (5 minutes)

```powershell
docker build --pull --progress=plain -t orders-api-lab:lesson-006 .
docker image inspect --format '{{.Config.User}}' orders-api-lab:lesson-006
docker image ls orders-api-lab
```

Expected:
- Build completes and tags the image.
- The configured user is `10001:10001`.
- The image appears in the local image list.

Do not expect a universal image-size target: base-image patch levels and platform affect size. The important design check is that the final stage contains the JRE and application JAR, not the Maven build toolchain.

## 5. Run with restricted settings and verify the runtime (5 minutes)

Run without publishing a port or mounting host directories:

```powershell
docker run --rm --read-only --tmpfs /tmp:rw,noexec,nosuid,size=16m --cap-drop=ALL --security-opt=no-new-privileges -e APP_ENV=dev -e RELEASE_ID=lesson-006 orders-api-lab:lesson-006
```

Expected output starts like this (the exact Java patch/build suffix can differ):

```text
app=orders-api env=dev release=lesson-006 java=21...
```

Prove Maven is absent from the final image:

```powershell
docker run --rm --entrypoint sh orders-api-lab:lesson-006 -c "if command -v mvn >/dev/null 2>&1; then echo 'ERROR: Maven is present'; exit 1; else echo 'OK: Maven is absent'; fi; command -v java"
```

Expected:

```text
OK: Maven is absent
/opt/java/openjdk/bin/java
```

The first run tests the application with a read-only root filesystem and reduced privileges. The second command overrides the normal entrypoint only for that temporary inspection container.

## 6. Clean up (3 minutes)

Remove only the image created by this lesson:

```powershell
docker image rm orders-api-lab:lesson-006
```

Both test containers used `--rm`, so they were removed automatically. Do **not** run `docker system prune` or `docker image prune`; those could remove unrelated images or build cache used by other work.

You can retain the source directory for review. If you later remove it, verify the path first with `Get-Location`.

## Quick self-check

1. Why does the final stage use a JRE rather than the Maven/JDK build image?
2. Why is `pom.xml` copied before `src`?
3. What does `USER 10001:10001` protect against?
4. What do `--read-only`, `--cap-drop=ALL` and `--security-opt=no-new-privileges` restrict?
5. Why should production pipelines pin base images by digest and scan the resulting image rather than trusting a tag alone?

**Expected outcome:** a Java 21 JAR built in a Maven stage; a separate JRE-only final image; a non-root configured user; a successful restricted run; confirmation that Maven is absent from the final image; and removal of only the lesson image.

## Production and interview relevance

Multi-stage builds reduce the tools and files shipped to runtime. Non-root execution, read-only filesystems, dropped capabilities and no-new-privileges reduce the impact of a compromised process. These controls complement, but do not replace, image scanning, SBOM/provenance generation, digest pinning, signed-image verification, resource limits, patch management and external secret management.

A Docker image tag is a mutable name, not an immutable identity. For controlled production promotion, build once, identify the image by digest, scan and approve it, then promote that same digest through environments. Keep environment configuration outside the image; never put secrets in Dockerfile `ARG`, `ENV` or build-context files.

## In simple words — what did we actually do?

We took a small Java application and packed it into a Docker image that can run consistently without needing Maven or a full Java development kit on the destination machine.

We deliberately used **two stages**:

1. **Build stage:** Maven and the full JDK compile and package the application.
2. **Runtime stage:** a smaller Java runtime image receives only the finished JAR.

The build tools are left behind, so the running image has fewer unnecessary components.

We then added safety restrictions:

- **Non-root user:** the application does not run with administrator-level privileges inside the container.
- **Read-only filesystem:** the application cannot casually modify its container filesystem.
- **Dropped capabilities:** the process gets fewer low-level Linux privileges.
- **No-new-privileges:** the process cannot gain additional privileges through mechanisms such as setuid.
- **Separate runtime configuration:** the same image can run as DEV or TEST by passing different non-secret environment values.

### Why does this matter?

A container is not automatically secure just because it is isolated. If an application is compromised, unnecessary tools, root privileges and writable filesystems can make the incident worse. Reducing what is inside the image and what the process is allowed to do limits the available attack surface.

### How does this fit into your DevOps toolchain?

| Technology | Responsibility |
|---|---|
| Maven | Produces the Java artifact. |
| Jenkins | Automates the build, image creation, tests and promotion. |
| Docker | Packages the artifact and its runtime into an image. |
| Image registry | Stores the versioned, approved image. |
| Kubernetes | Runs the image with deployment-level controls. |
| Terraform | Can provision the infrastructure on which the platform runs. |

### Three things to remember

- **Build separately from runtime:** do not ship compilers and build tools unless the application genuinely needs them.
- **Run with minimum privileges:** non-root and restricted runtime settings should be normal defaults.
- **Promote the same image:** keep environment configuration separate and use an immutable digest for controlled promotion.

This was a local container-build and runtime-hardening exercise. It did not deploy anything to a server or alter existing containers.

## Completion checkpoint

Record these results when you complete the exercise:

- Docker Client and Server versions.
- Whether Buildx was available.
- Whether the image build succeeded.
- Output of the image configured-user check.
- Whether the restricted container run succeeded.
- Whether the final image reported Maven absent.
- Whether the lesson image was removed.
- One additional hardening control you would require in production.
- Confidence from 1–5.

Do not paste credentials, private image-registry details, secret files or sensitive configuration. Do not mark the lesson complete until the checks have actually been run.

## References

- [Docker Docs — Building best practices](https://docs.docker.com/build/building/best-practices/) — `.dockerignore`, build cache, base-image selection, non-root `USER`, and digest pinning.
- [Docker Docs — Multi-stage builds](https://docs.docker.com/build/building/multi-stage/) — named stages and copying only the required artifact into the final image.


## 🧸 For Dummies

Imagine you want to make and deliver one pizza. You need a big kitchen with ovens, mixers, ingredients and tools to prepare it. But when the pizza is finished, you do not send the entire kitchen to the customer. You send only the pizza in a box.

That is exactly the idea behind this lesson.

### 1. The big kitchen: building the application

Maven and the full Java kit are our big kitchen. They have all the tools needed to turn the Java instructions we wrote into a finished file called a **JAR**.

Think of the JAR as the finished pizza. It is the part we actually want to deliver.

### 2. The delivery box: the Docker image

Docker takes that finished JAR and puts it into a much smaller package that also contains the basic Java equipment needed to run it.

We use two stages so that the big kitchen stays behind. The customer gets the pizza, not the ovens and mixers!

That means the final Docker image does not need Maven or the full development kit.

### 3. The worker: running without administrator powers

Inside the package, our application runs as an ordinary worker, not as the boss of the whole building.

Why? If the worker makes a mistake or someone tricks it, we do not want it to have the keys to everything.

That is what **non-root** means here: the application is not running as the all-powerful administrator inside the container.

### 4. The safety rules

We also give our worker some simple rules:

- **Read-only:** it can look at its normal files, but it cannot freely scribble over them.
- **Fewer permissions:** it is not given powerful keys it does not need.
- **No extra privileges:** it cannot promote itself to a more powerful worker.
- **Separate settings:** we can tell the same application whether it is running in DEV or TEST without rebuilding the package.

### 5. What did we prove?

We built the package, checked that it was configured to run as an ordinary user, started it with the safety rules, and checked that Maven was not inside the final package.

We then removed only the package made for this exercise.

### The whole story in one sentence

**We prepared our Java application in a fully equipped kitchen, packed only what it needs to run, gave it limited permissions, and checked that the unnecessary tools were left behind.**

That is Docker image hardening: make the package smaller and give the running application only the permissions it needs. 🧸
