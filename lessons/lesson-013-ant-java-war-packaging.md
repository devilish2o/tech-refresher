# Lesson 013 — Apache Ant: compile, package and verify a Java WAR

**Date:** 2026-11-24  
**Domain:** Java build engineering / Apache Ant / artifact integrity  
**Duration:** 30 minutes  
**Environment:** Windows PowerShell 5.1 on RANGER; isolated local directory  
**Version assumptions:** Apache Ant 1.10.x and JDK 11 or newer. The `release` attribute of Ant's `javac` task requires Ant 1.9.8+ and JDK 9+. This lab has no third-party dependencies and does not deploy to an application server.

## Objective

Build a small static web application WAR with Ant, compile one Java helper class, run a smoke test, inspect the archive layout, and verify its SHA-256 checksum.

This complements Lesson 001's Maven WAR build. The key difference is **how each tool describes and executes a build**:

- **Ant is task/target oriented (procedural):** the build author writes the steps — create directories, compile, copy resources, package, verify — and defines their dependencies/order.
- **Maven is project/lifecycle oriented (declarative/convention-based):** the POM describes the project, packaging type, dependencies and configuration. Maven supplies a standard lifecycle and plugins that perform conventional steps.

Neither is universally better. Ant offers explicit control and suits customised/legacy builds. Maven standardises common Java builds and provides integrated dependency management.

## Maven versus Ant — a practical explanation

Imagine both tools must produce the same `ops-status.war`.

### Ant: you write the recipe

Ant's main build file is `build.xml`. A target groups tasks, and `depends` establishes ordering:

```xml
<project name="ops-status" default="package">
    <target name="prepare">
        <mkdir dir="build/classes"/>
    </target>

    <target name="compile" depends="prepare">
        <javac srcdir="src/main/java"
               destdir="build/classes"
               includeantruntime="false"/>
    </target>

    <target name="package" depends="compile">
        <war destfile="dist/ops-status.war"
             webxml="src/main/webapp/WEB-INF/web.xml">
            <fileset dir="src/main/webapp"
                     excludes="WEB-INF/web.xml"/>
            <classes dir="build/classes"/>
        </war>
    </target>
</project>
```

Read it literally: create an output directory; compile the Java files; put the compiled classes and web resources into a WAR. If you need another operation, you add another task/target and wire its dependency/order. Ant does not impose a standard Java lifecycle or standard source layout.

Run a target you defined:

```powershell
ant package
```

### Maven: describe the project and use its lifecycle

Maven's main file is `pom.xml`. For a conventional WAR project, this minimal POM tells Maven the coordinates, Java release and packaging type:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>ca.kairos.ops</groupId>
    <artifactId>ops-status</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>

    <properties>
        <maven.compiler.release>11</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
</project>
```

Maven conventionally expects Java source under `src/main/java`, test source under `src/test/java`, and web application content under `src/main/webapp`. It uses lifecycle phases and plugins to compile, test and package. You normally do not write the individual directory-creation and compiler commands yourself.

Run the standard lifecycle through the package phase:

```powershell
mvn clean package
```

Maven runs preceding phases in order, including validation, resource processing, compilation and testing, before packaging. A phase may have no goal bound to it in a particular project, but it remains part of the lifecycle.

### The dependency-management difference

With Ant, dependencies can be managed manually (for example, JARs in a `lib/` directory and an explicit classpath), or with an additional tool such as Apache Ivy. Ant itself does not provide Maven's integrated dependency model.

With Maven, dependencies are declared in the POM. Maven resolves them from configured repositories, including transitive dependencies:

```xml
<dependencies>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.18.2</version>
    </dependency>
</dependencies>
```

### Side-by-side

| Concern | Ant | Maven |
|---|---|---|
| Main file | `build.xml` | `pom.xml` |
| Mental model | Write the build steps | Describe project; use standard lifecycle |
| Execution units | Targets containing tasks | Lifecycle phases with plugin goals |
| Ordering | Explicit target dependencies and task order | Standard phase order; plugin goals bound to phases |
| Project layout | Build author chooses it | Standard conventions by default |
| Java compilation | Usually explicit `<javac>` task | Compiler plugin bound to lifecycle |
| WAR packaging | Usually explicit `<war>` task | WAR plugin bound to `package` |
| Dependencies | Manual or additional tooling such as Ivy | Integrated dependency resolution |
| Customisation | Very direct and flexible | Plugin configuration and lifecycle extensions |

### How Jenkins fits

Jenkins orchestrates the build; it does not replace Ant or Maven. A Jenkins stage might run `mvn clean verify` or `ant clean verify`, then archive the resulting WAR. The build tool creates and validates the artifact; Jenkins coordinates agents, stages, approvals and artifact handling.

**For a new conventional Java application, Maven is generally the easier default.** Ant remains important in established applications with bespoke build logic. Maven can invoke Ant tasks through a plugin, but that should be a deliberate compatibility choice rather than the default design.

🧸 **For Dummies:** Ant is like giving a builder every instruction in order. Maven is like telling a builder what kind of house you want and using the builder's standard construction process. Both can produce the same house; one makes you describe more of the steps.

## Timed plan

| Time | Activity |
|---|---|
| 0–4 min | Check JDK and Ant prerequisites |
| 4–10 min | Create the minimal Java web application |
| 10–19 min | Define Ant compile, smoke-test, package and verify targets |
| 19–25 min | Build and inspect the WAR |
| 25–28 min | Verify the SHA-256 sidecar and test a clean rebuild |
| 28–30 min | Self-check |

## Official documentation

1. [Apache Ant — WAR task](https://ant.apache.org/manual/Tasks/war.html) — `webxml`, `classes`, duplicate-entry handling and archive timestamps.
2. [Apache Ant — Javac task](https://ant.apache.org/manual/Tasks/javac.html) — compilation destination, `includeantruntime`, Java release targeting and incremental compilation.
3. [Maven — Introduction to the Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle) — lifecycle phases and plugin goals.
4. [Maven — Introduction to the POM](https://maven.apache.org/guides/introduction/introduction-to-the-pom.html) — project coordinates, packaging and dependencies.

## 1. Preflight — 4 minutes

Open **Windows PowerShell** normally on RANGER.

Run these separately:

```powershell
$PSVersionTable.PSVersion
java -version
javac -version
ant -version
Get-Command java, javac, ant -ErrorAction SilentlyContinue |
    Select-Object Name, Source
```

Expected:
- Java runtime and compiler are both available.
- Ant reports version 1.10.x.
- JDK is 11 or newer.

If Ant is missing, do not install or change system configuration during this lesson. Continue by reviewing the project/build files, then report that the build could not be executed because Ant is unavailable.

## 2. Create the project — 6 minutes

Use this isolated directory:

```powershell
$lab = Join-Path $HOME 'labs\tech-refresher\lesson-013-ant-war'
$javaDir = Join-Path $lab 'src\main\java\ca\kairos\ops'
$webDir = Join-Path $lab 'src\main\webapp'
$webInfDir = Join-Path $webDir 'WEB-INF'

New-Item -ItemType Directory -Path $javaDir, $webInfDir -Force |
    Out-Null

$lab
```

Create the Java helper class:

```powershell
@'
package ca.kairos.ops;

public final class BuildInfo {
    private BuildInfo() {
    }

    public static void main(String[] args) {
        System.out.println("ops-status build 1.0.0");
    }
}
'@ | Set-Content -LiteralPath (Join-Path $javaDir 'BuildInfo.java') -Encoding ASCII
```

Create a static landing page:

```powershell
@'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Operations Status</title>
</head>
<body>
  <h1>Operations Status</h1>
  <p>Local Ant WAR packaging exercise.</p>
</body>
</html>
'@ | Set-Content -LiteralPath (Join-Path $webDir 'index.html') -Encoding ASCII
```

Create a Servlet 3.1 deployment descriptor:

```powershell
@'
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <display-name>Operations Status</display-name>
</web-app>
'@ | Set-Content -LiteralPath (Join-Path $webInfDir 'web.xml') -Encoding ASCII
```

This is a packaging exercise, not a servlet implementation. The WAR contains a static page and a compiled helper class; no application server is needed.

## 3. Define the Ant build — 9 minutes

Create `build.xml` in the project root:

```powershell
$buildXml = @'
<project name="ops-status" default="verify" basedir=".">

    <property name="src.dir" location="src/main/java"/>
    <property name="web.dir" location="src/main/webapp"/>
    <property name="build.dir" location="build"/>
    <property name="classes.dir" location="${build.dir}/classes"/>
    <property name="verify.dir" location="${build.dir}/verify"/>
    <property name="dist.dir" location="dist"/>
    <property name="war.file" location="${dist.dir}/ops-status.war"/>

    <target name="clean">
        <delete dir="${build.dir}"/>
        <delete dir="${dist.dir}"/>
    </target>

    <target name="prepare">
        <mkdir dir="${classes.dir}"/>
        <mkdir dir="${dist.dir}"/>
    </target>

    <target name="compile" depends="prepare">
        <javac srcdir="${src.dir}"
               destdir="${classes.dir}"
               release="11"
               encoding="UTF-8"
               includeantruntime="false"
               debug="false"
               failonerror="true"/>
    </target>

    <target name="smoke-test" depends="compile">
        <java classname="ca.kairos.ops.BuildInfo"
              fork="true"
              failonerror="true">
            <classpath>
                <pathelement location="${classes.dir}"/>
            </classpath>
        </java>
    </target>

    <target name="package" depends="smoke-test">
        <war destfile="${war.file}"
             webxml="${web.dir}/WEB-INF/web.xml"
             duplicate="fail"
             modificationtime="2026-01-01T00:00:00Z">
            <fileset dir="${web.dir}" excludes="WEB-INF/web.xml"/>
            <classes dir="${classes.dir}"/>
        </war>

        <checksum file="${war.file}" algorithm="SHA-256"/>
    </target>

    <target name="verify" depends="package">
        <delete dir="${verify.dir}"/>
        <unzip src="${war.file}" dest="${verify.dir}"/>

        <available file="${verify.dir}/index.html"
                   type="file"
                   property="index.present"/>
        <available file="${verify.dir}/WEB-INF/web.xml"
                   type="file"
                   property="webxml.present"/>
        <available file="${verify.dir}/WEB-INF/classes/ca/kairos/ops/BuildInfo.class"
                   type="file"
                   property="class.present"/>

        <fail unless="index.present" message="WAR is missing index.html"/>
        <fail unless="webxml.present" message="WAR is missing WEB-INF/web.xml"/>
        <fail unless="class.present" message="WAR is missing BuildInfo.class"/>

        <checksum file="${war.file}"
                  algorithm="SHA-256"
                  verifyproperty="checksum.valid"/>
        <fail unless="checksum.valid" message="WAR SHA-256 verification failed"/>

        <echo message="PASS: WAR layout and SHA-256 checksum verified."/>
    </target>

</project>
'@

Set-Content -LiteralPath (Join-Path $lab 'build.xml') -Value $buildXml -Encoding ASCII
```

Build graph:

```text
clean (explicit CLI target)
  |
prepare
  |
compile
  |
smoke-test
  |
package
  |
verify
```

Operational points:
- `includeantruntime="false"` prevents Ant's own runtime libraries from silently becoming application compile dependencies.
- `release="11"` fixes the Java API/language target rather than inheriting an accidental default from the build JDK.
- `duplicate="fail"` prevents ambiguous duplicate WAR entries; Ant's WAR task otherwise defaults to adding duplicates.
- The fixed archive entry timestamp helps repeatability, but does not alone guarantee a fully reproducible build.
- The WAR contains application content only. Keep environment-specific configuration and secrets outside the artifact.

## 4. Build and inspect — 6 minutes

From the project root:

```powershell
Set-Location $lab
ant clean verify
```

Expected smoke-test output:

```text
ops-status build 1.0.0
```

Expected final message:

```text
PASS: WAR layout and SHA-256 checksum verified.
BUILD SUCCESSFUL
```

Inspect the archive:

```powershell
jar tf .\dist\ops-status.war
```

Confirm these entries exist:

- `index.html`
- `WEB-INF/web.xml`
- `WEB-INF/classes/ca/kairos/ops/BuildInfo.class`
- `META-INF/MANIFEST.MF`

Inspect the artifact and checksum sidecar:

```powershell
Get-ChildItem .\dist
Get-FileHash .\dist\ops-status.war -Algorithm SHA256
Get-Content .\dist\ops-status.war.SHA-256
```

The checksum file contains the SHA-256 digest generated by Ant. The verification target checks it again.

## 5. Clean rebuild and release thinking — 3 minutes

Run:

```powershell
ant clean verify
```

The build must succeed again from a clean state.

For a real release pipeline, retain the exact WAR and its digest as immutable build outputs. Promote that same artifact between dev/test/prod; do not rebuild separately for each environment. Keep environment configuration outside the WAR and record the artifact digest in the deployment evidence.

No deployment is performed in this lab.

## Expected outcome

You should have:
- `build.xml` with separate clean, compile, smoke-test, package and verify targets.
- A compiled Java class under `build/classes`.
- A WAR under `dist/ops-status.war`.
- A SHA-256 sidecar.
- A successful archive-layout and checksum verification.

## Quick self-check

1. In one sentence, what does Ant make you define that Maven normally supplies through conventions and its lifecycle?
2. What is the difference between an Ant target/task and a Maven lifecycle phase/plugin goal?
3. Why does Maven generally make dependency management easier?
4. Why should dev/test/prod configuration not be baked into the WAR?
5. Which build tool would you normally choose for a new conventional Java application, and why?

## Production / interview relevance

Ant remains common in established enterprise Java estates. A reliable build makes target compatibility explicit, separates compile/test/package/verify targets, fails on packaging ambiguity, and emits an artifact plus digest. The same artifact should be promoted across environments; rebuilding for each environment weakens traceability.

A concise interview answer:

**“I make the JDK target explicit, compile into a clean output directory, run a smoke test, package the WAR with duplicate-entry protection, inspect its contents and verify a SHA-256 digest. I then promote the immutable artifact while injecting environment configuration at deployment time.”**

## Completion checkpoint

Report:
- Java runtime/compiler versions.
- Ant version.
- `ant clean verify`: passed or blocked.
- Whether the smoke test printed the expected build string.
- Whether all three application entries were present in the WAR.
- Whether SHA-256 verification passed.
- Whether the Maven-versus-Ant explanation is now clear.

Do not paste environment variables containing secrets, internal repository URLs, or private configuration.

## 🧸 For Dummies

Ant is like giving a builder every instruction in order. Maven is like telling a builder what kind of house you want and using the builder's standard construction process. Both can produce the same house; one makes you describe more of the steps. 🧸
