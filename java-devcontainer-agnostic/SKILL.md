---
name: java-devcontainer-agnostic
user-invocable: true
description: "Create or update a Java devcontainer configuration that is agnostic to the exact JDK version and can be generated based on the current Java project files. Use this skill when the project may target different Java versions and the container should adapt to the detected project settings."
---

# Java Devcontainer Version-Agnostic Skill

Use this skill to produce a `.devcontainer` configuration that does not hard-code a single JDK version, but instead detects the Java version from the current project and creates a container setup compatible with it.

## What this skill should do

1. Before anything else, verify that Docker is available and usable on the current machine:
   - run a Docker availability check such as `docker version` or `docker info`.
   - if Docker is available and the user has permission, proceed.
   - if Docker is unavailable, ask the user to install Docker or offer to install it for them when possible.
   - if Docker is installed but the current user lacks permission, request the necessary authorization or instruct to add the user to the Docker group.
   - return the exact Docker check output to the agent so the user can see the failure reason.
2. Inspect the project root for Java build files and version metadata:
   - `pom.xml`
   - `build.gradle`, `build.gradle.kts`
   - `settings.gradle`, `settings.gradle.kts`
   - `gradle.properties`
   - `src/main/resources/application.properties` / `application.yml` if relevant
   - `.java-version`, `toolchains.xml`, `maven.compiler.source`, `maven.compiler.target`, `java.version`, `sourceCompatibility`, `targetCompatibility`
3. Determine the most likely Java major version required by the project.
4. Produce a `.devcontainer/devcontainer.json` and/or `.devcontainer/Dockerfile` that:
   - uses a generic base image or build arguments instead of hard-coded JDK version strings,
   - supports selecting the appropriate JDK at container build time,
   - exposes a `JAVA_HOME` compatible with the installed JDK,
   - installs common Java tooling such as Maven or Gradle only when needed,
   - for Maven projects, prefer `./mvnw` when present and otherwise install Maven;
   - for Gradle projects, prefer `./gradlew` when present and otherwise install Gradle or Gradle wrapper dependencies.
5. If the project cannot be clearly mapped to a specific Java version, choose a sensible default such as Java 17 and document the fallback.

## Output format

The skill should return a complete `.devcontainer/devcontainer.json` and `.devcontainer/Dockerfile` pair, or a clear change plan if the existing `.devcontainer` already exists.

### Recommended pattern

- `devcontainer.json` should include an `args` field such as `JAVA_VERSION` or `JDK_VERSION`.
- `Dockerfile` should use that build arg to install the requested JDK.
- Prefer stable package installation paths (Ubuntu OpenJDK packages or official Eclipse Temurin images) over brittle external version strings.
- If the project is Maven-based, include Maven installation; if Gradle-based, include Gradle as needed.

## Boilerplate examples

### Example `.devcontainer/devcontainer.json`

```json
{
  "name": "Java Dev Container",
  "build": {
    "dockerfile": "Dockerfile",
    "args": {
      "JDK_VERSION": "${localEnv:JDK_VERSION:-17}"
    }
  },
  "settings": {
    "terminal.integrated.shell.linux": "/bin/bash",
    "java.home": "/usr/lib/jvm/java-${localEnv:JDK_VERSION:-17}-openjdk-amd64"
  },
  "extensions": [
    "vscjava.vscode-java-pack",
    "vscjava.vscode-maven"
  ],
  "mounts": [
    "source=m2,target=/root/.m2,type=volume"
  ],
  "postCreateCommand": "./.devcontainer/setup.sh",
  "remoteUser": "vscode"
}
```

### Example `.devcontainer/Dockerfile`

```dockerfile
ARG JDK_VERSION=17
FROM mcr.microsoft.com/vscode/devcontainers/base:ubuntu-22.04

ARG JDK_VERSION
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    git \
    unzip \
    wget \
    openjdk-${JDK_VERSION}-jdk-headless \
    maven \
    && rm -rf /var/lib/apt/lists/*

ENV JAVA_HOME=/usr/lib/jvm/java-${JDK_VERSION}-openjdk-amd64
ENV PATH="$JAVA_HOME/bin:${PATH}"

RUN useradd -m vscode && echo "vscode ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/vscode && chmod 0440 /etc/sudoers.d/vscode
USER vscode
```

### Notes for generation

- If the project has `pom.xml`, treat it as Maven-based:
  - detect `maven.compiler.source`, `maven.compiler.target`, `java.version`, `toolchains.xml`, or `.java-version`.
  - if `./mvnw` exists, prefer the Maven wrapper and do not install Maven globally unless necessary.
  - otherwise install Maven in the container and set `MAVEN_HOME`/`PATH` if needed.
- If the project has `build.gradle`, `build.gradle.kts`, or `gradle.properties`, treat it as Gradle-based:
  - detect `sourceCompatibility`, `targetCompatibility`, `java.sourceCompatibility`, `java.targetCompatibility`, or `org.gradle.java.home`.
  - if `./gradlew` exists, prefer the Gradle wrapper and only install wrapper support dependencies (e.g. `unzip`, `wget`).
  - otherwise install Gradle globally or use an official Gradle image variant.
- When the project cannot be reliably detected, prefer `17` or a long-term-support release.
- For both Maven and Gradle, document the wrapper preference and explain that the container should use the project-local wrapper if present.

## Example trigger phrases

- "Create a version-agnostic Java devcontainer"
- "Generate devcontainer config from Java project metadata"
- "Make `.devcontainer` adapt to the project Java version"
- "Make .devcontainer environment to my Java project"
