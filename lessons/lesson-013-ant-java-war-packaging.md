# Lesson 013 — Apache Ant: compile, package and verify a Java WAR

**Date:** 2026-11-24  
**Domain:** Java build engineering / Apache Ant / artifact integrity  
**Duration:** 30 minutes  
**Environment:** Windows PowerShell 5.1 on RANGER; isolated local directory  
**Version assumptions:** Apache Ant 1.10.x and JDK 11 or newer. The `release` attribute of Ant's `javac` task requires Ant 1.9.8+ and JDK 9+. This lab has no third-party dependencies and does not deploy to an application server.

## Objective

Build a small static web application WAR with Ant, compile one Java helper class, run a smoke test, inspect the archive layout, and verify its SHA-256 checksum.

This complements Lesson 001's Maven WAR build. Maven and Ant are different build tools: Maven is convention/declarative and dependency-lifecycle oriented; Ant is task/target oriented and gives the build author explicit control over sequencing.

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

Useful supporting reference: [Apache Ant — Checksum task](https://ant.apache.org/manual/Tasks/checksum.html), for SHA-256 generation and verification.

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

1. Why is Ant's `<javac>` destination directory separate from the source tree?
2. What does `includeantruntime="false"` protect against?
3. Why should duplicate WAR entries fail the build?
4. Why is a SHA-256 digest useful when promoting an immutable artifact?
5. Why should dev/test/prod configuration not be baked into the WAR?

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
- One difference between Ant targets/tasks and Maven's lifecycle/plugins.

Do not paste environment variables containing secrets, internal repository URLs, or private configuration.

## 🧸 For Dummies

Think of a WAR file as a packed suitcase for a Java web application.

- `javac` packs the compiled Java clothes.
- The web folder adds the web pages and instructions.
- Ant puts everything into the suitcase.
- The verification step opens the suitcase and checks that nothing important is missing.
- SHA-256 gives the suitcase a fingerprint so we can check that the same suitcase reaches the next environment.

We pack once, check it, and send that same suitcase to test and production. We don't secretly repack it differently for every destination. 🧸
