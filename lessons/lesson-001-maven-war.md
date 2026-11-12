# Lesson 001 — Build a Java WAR with Maven

**Domain:** Java build / CI foundations  
**Duration:** 30 minutes  
**Assumption:** JDK 17, Maven 3.9.x, Jakarta Servlet 6.0, Maven WAR Plugin 3.5.1. This is a local packaging exercise only; no application server or production target is involved.

## Objective

Build a small Java web application into a versioned WAR, inspect its contents, and calculate a checksum. Understand which Maven lifecycle phase compiles code and which packages the WAR.

## 30-minute plan

- **0–5 min — Recall:** distinguish Maven lifecycle phases from plugin goals. Review `validate → compile → test → package → verify → install → deploy`.
- **5–10 min — Configure:** create the minimal project and inspect the POM.
- **10–20 min — Build:** run a clean verification build and diagnose any failure.
- **20–26 min — Inspect:** list the WAR contents and verify the servlet class and web resources.
- **26–30 min — Operational review:** record the artifact name, checksum and one deployment-safety observation.

## 1. Create the project

Create this structure in a disposable working directory:

```text
war-refresher/
├── pom.xml
└── src/
    └── main/
        ├── java/com/example/HealthServlet.java
        └── webapp/index.html
```

### `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>war-refresher</artifactId>
  <version>1.0.0</version>
  <packaging>war</packaging>

  <properties>
    <maven.compiler.release>17</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>jakarta.servlet</groupId>
      <artifactId>jakarta.servlet-api</artifactId>
      <version>6.0.0</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.14.1</version>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-war-plugin</artifactId>
        <version>3.5.1</version>
      </plugin>
    </plugins>
  </build>
</project>
```

### `src/main/java/com/example/HealthServlet.java`

```java
package com.example;

import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebServlet("/health")
public class HealthServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response) throws IOException {
        response.setContentType("text/plain");
        response.getWriter().println("OK");
    }
}
```

### `src/main/webapp/index.html`

```html
<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>WAR refresher</title></head>
<body><h1>WAR packaging exercise</h1></body>
</html>
```

## 2. Build and inspect

From the project root:

```bash
java -version
mvn -version
mvn -B clean verify
```

Expected artifact:

```text
target/war-refresher-1.0.0.war
```

Inspect it:

```bash
jar tf target/war-refresher-1.0.0.war
sha256sum target/war-refresher-1.0.0.war
```

On Windows PowerShell, use `Get-FileHash .\target\war-refresher-1.0.0.war -Algorithm SHA256` for the checksum.

Confirm the archive contains:
- `index.html` at the web application root;
- `WEB-INF/classes/com/example/HealthServlet.class`;
- `WEB-INF/lib` does **not** contain the Servlet API JAR, because the dependency is `provided` by the target servlet container.

## Key points

- `<packaging>war</packaging>` binds the WAR plugin's `war:war` goal to Maven's `package` phase.
- Maven runs earlier lifecycle phases first, including resource processing, compilation and tests.
- The WAR plugin packages classes and web resources; it does not compile Java source itself.
- `provided` means the servlet container supplies the API at runtime; it is available for compilation but is not bundled as an application dependency.
- Maven's `deploy` phase publishes an artifact to a Maven repository. It does **not** deploy the application to Tomcat, WebLogic or another runtime.
- A build artifact should be promoted unchanged between environments. Environment-specific configuration belongs outside the immutable WAR wherever the application architecture allows.

## Self-check

1. Why does `mvn clean verify` invoke the WAR packaging step?
2. Why is `jakarta.servlet-api` marked `provided`?
3. What is the difference between Maven's `deploy` phase and deploying a WAR to an application server?
4. What evidence would you retain to prove that test and production-like environments received the same artifact?

## Completion checkpoint

Report:
- build result (success/failure);
- WAR file size;
- first 12–16 characters of the SHA-256 checksum (do not share private project details);
- whether the three expected archive entries were present;
- confidence from 1–5.

Do not mark the lesson complete until you have actually run the build and inspection.

## Official references

- [Maven Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Maven WAR Plugin — Usage](https://maven.apache.org/plugins/maven-war-plugin/usage.html)
