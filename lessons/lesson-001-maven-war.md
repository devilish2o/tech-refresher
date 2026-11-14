# Lesson 001 — Build a Java WAR with Maven

**Domain:** Java build / Maven / CI foundations  
**Duration:** 30 minutes, excluding one-time software installation  
**Level:** Experienced engineer refresher  
**Environment:** Windows 11 PowerShell or Debian/Ubuntu Linux  
**Version assumptions:** JDK 17+, Maven 3.9.x, Jakarta Servlet API 6.0.0, Maven Compiler Plugin 3.14.1, Maven WAR Plugin 3.5.1.

> This guide is self-contained: it tells you what to install, where to create each file, which directory to run commands from, and what success looks like. If Java and Maven are already installed, verify them and skip the installation steps.

## Objective

Create a small Java web application, compile it and package it as a versioned WAR. Inspect the archive and calculate its SHA-256 checksum. Refresh the Maven lifecycle, dependency scope and artifact-promotion concepts.

## 1. Prerequisites and installation

You need a **JDK** (not just a Java runtime), Apache Maven, a terminal and a text editor. Git is useful but not required. You do not need Jenkins, Docker, an application server or a database.

### 1.1 Install/check Java

Open PowerShell on Windows or a terminal on Linux:

```text
java -version
javac -version
```

Both commands should return version information. The JDK supplies `javac`; if it is missing, install a JDK or correct your PATH.

- **Windows:** Download the Temurin 17 JDK MSI from [Eclipse Temurin](https://adoptium.net/temurin/releases/?version=17). Choose Windows x64, JDK, MSI. During setup, enable the options to add Java to PATH and set JAVA_HOME if offered. Close and reopen PowerShell.
- **Debian/Ubuntu:**

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version
javac -version
```

For other Linux distributions, use the JDK vendor's installation instructions.

### 1.2 Install/check Maven

Run:

```text
mvn -version
```

The output should identify Maven 3.9.x and the Java version Maven is using.

- **Windows:** Download the binary ZIP from [Apache Maven — Download](https://maven.apache.org/download.cgi). Extract it to a stable directory, for example `C:\Tools\apache-maven-3.9.x` (replace x with the downloaded version). Add its `bin` directory to PATH, e.g. `C:\Tools\apache-maven-3.9.x\bin`. Set JAVA_HOME to your JDK installation if it is not already set. Open a new PowerShell window and run `mvn -version` again.
- **Debian/Ubuntu:**

```bash
sudo apt update
sudo apt install -y maven
mvn -version
```

Check the reported version. If the distribution package is older than 3.9.x, install a current binary release from the Apache Maven download page.

### 1.3 Verify the toolchain before continuing

**PowerShell:**
```powershell
java -version
javac -version
mvn -version
$env:JAVA_HOME
Get-Command java
Get-Command mvn
```

**Linux:**
```bash
java -version
javac -version
mvn -version
echo "$JAVA_HOME"
command -v java
command -v mvn
```

Maven needs internet access the first time to download plugins and dependencies. Its local cache is normally `%USERPROFILE%\.m2\repository` on Windows or `~/.m2/repository` on Linux. That cache is **not** your project folder.

## 2. Create the project

Use a disposable working directory:

- Windows: `C:\DevOps\war-refresher`
- Linux: `~/labs/war-refresher`

The **project root** means the top-level directory containing `pom.xml`. Run Maven commands from this directory.

### 2.1 Create the directories

**PowerShell:**
```powershell
New-Item -ItemType Directory -Force -Path C:\DevOps\war-refresher\src\main\java\com\example
New-Item -ItemType Directory -Force -Path C:\DevOps\war-refresher\src\main\webapp
Set-Location C:\DevOps\war-refresher
```

**Linux:**
```bash
mkdir -p ~/labs/war-refresher/src/main/java/com/example
mkdir -p ~/labs/war-refresher/src/main/webapp
cd ~/labs/war-refresher
```

### 2.2 Required source layout

Create each file at exactly this relative path:

```text
war-refresher/                         <- project root; run mvn here
├── pom.xml                            <- Maven build definition
└── src/
    └── main/
        ├── java/com/example/
        │   └── HealthServlet.java
        └── webapp/
            └── index.html
```

Do not create `target` yourself; Maven creates it. Do not put source files in `.m2`.

### 2.3 Create `pom.xml` in the project root

Full path: `C:\DevOps\war-refresher\pom.xml` on Windows, or `~/labs/war-refresher/pom.xml` on Linux.

Paste this complete file:

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

### 2.4 Create the Java servlet

File path: `src/main/java/com/example/HealthServlet.java`

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

### 2.5 Create the web page

File path: `src/main/webapp/index.html`

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>WAR refresher</title>
</head>
<body>
  <h1>WAR packaging exercise</h1>
</body>
</html>
```

### 2.6 Check the files before building

**PowerShell (run from the project root):**
```powershell
Get-Location
Get-ChildItem
Test-Path .\pom.xml
Test-Path .\src\main\java\com\example\HealthServlet.java
Test-Path .\src\main\webapp\index.html
```

The three `Test-Path` commands should return `True`.

**Linux:**
```bash
pwd
find . -maxdepth 7 -type f -print
```

## 3. Build the WAR

Ensure the current directory is the project root (the directory containing `pom.xml`). Run:

```bash
mvn -B -ntp clean verify
```

- `-B`: batch mode, suitable for CI logs.
- `-ntp`: hides download progress to keep logs readable.
- `clean`: removes the previous generated `target` directory.
- `verify`: runs the lifecycle through verification, including earlier phases such as compilation, tests (if any) and packaging.

Expected final message: `BUILD SUCCESS`.

Expected artifact: `target/war-refresher-1.0.0.war`.

If it fails, inspect the first meaningful error. Common causes: Maven is using the wrong JDK, a source path/package typo, or a dependency download problem.

## 4. Inspect the generated artifact

Maven creates `target`; you should not create it manually. After a successful build, the project looks broadly like this:

```text
war-refresher/
├── pom.xml
├── src/                                  <- your source files
└── target/                               <- generated build output
    ├── classes/
    ├── war-refresher-1.0.0/
    └── war-refresher-1.0.0.war           <- WAR archive
```

List the archive:

```text
jar tf target/war-refresher-1.0.0.war
```

Confirm it contains:
- `index.html`
- `WEB-INF/classes/com/example/HealthServlet.class`
- `WEB-INF/`

The Servlet API JAR should not be bundled under `WEB-INF/lib`, because its scope is `provided`.

Calculate the checksum:

**PowerShell:**
```powershell
Get-FileHash .\target\war-refresher-1.0.0.war -Algorithm SHA256
```

**Linux:**
```bash
sha256sum target/war-refresher-1.0.0.war
```

Record the size and checksum. A checksum helps compare artifact copies; it does not on its own prove who built the artifact or that the build process was trustworthy.

## 5. Key Maven concepts

The default lifecycle includes:

`validate → compile → test → package → verify → install → deploy`

- A lifecycle **phase** is a named stage in Maven's standard build process.
- A plugin **goal** is a specific task implemented by a plugin.
- `<packaging>war</packaging>` selects WAR packaging and binds the WAR plugin's packaging goal to the `package` phase.
- The Compiler Plugin compiles Java source. The WAR Plugin assembles compiled classes and web resources into the archive.
- `provided` means the Servlet API is available at compile time but expected to be supplied by the runtime container; it is not bundled as a normal application dependency.
- Maven's `deploy` phase publishes an artifact to a Maven repository. It does **not** install the WAR into Tomcat, WebLogic or another application server.

**Jakarta versus Javax:** this exercise imports `jakarta.servlet.*` and targets Servlet 6.0. Use a Jakarta-compatible runtime such as Tomcat 10.1+ if you later choose to run it. Older Java EE-era containers expect `javax.servlet.*` and are not compatible without changes.

## 6. Production-minded review

Build once and promote the same versioned artifact through environments rather than rebuilding separately for test and production. Keep environment-specific configuration outside the immutable WAR where the application design permits it. Retain artifact identity (version and checksum) with test and deployment evidence.

No application-server deployment is part of this lesson.

## 7. Self-check

1. Why does `mvn clean verify` execute WAR packaging?
2. Why is `jakarta.servlet-api` declared with `provided` scope?
3. What is the difference between Maven's `deploy` phase and deploying a WAR to an application server?
4. Which directory contains source files? Which contains generated output?
5. What evidence helps prove that two environments received the same artifact?

## Completion checkpoint

Report the Java and Maven versions, build result, WAR file size, first 12–16 characters of its SHA-256 checksum, whether the expected archive entries are present, and confidence from 1–5. Do not share private project details. Do not mark this complete until you have actually run the build and inspected the archive.

## Official documentation

- [Eclipse Temurin JDK 17 downloads](https://adoptium.net/temurin/releases/?version=17)
- [Apache Maven — Installing Maven](https://maven.apache.org/install.html)
- [Apache Maven — Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Apache Maven WAR Plugin — Usage](https://maven.apache.org/plugins/maven-war-plugin/usage.html)
- [Jakarta Servlet 6.0](https://jakarta.ee/specifications/servlet/6.0/)
