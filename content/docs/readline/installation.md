---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Installation'
---

## Maven

Add the dependency to your `pom.xml`:

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>readline</artifactId>
  <version>3.5</version>
</dependency>
```

## Gradle

Add the dependency to your `build.gradle`:

```groovy
dependencies {
    implementation 'org.aesh:readline:3.5'
}
```

## Build from Source

Clone the repository and build with Maven:

```bash
git clone https://github.com/aeshell/aesh-readline.git
cd aesh-readline
mvn clean install
```

## Terminal Connectivity

Æsh Readline includes modules for remote connectivity:

### SSH Support

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>terminal-ssh</artifactId>
  <version>3.5</version>
</dependency>
```

### Telnet Support

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>terminal-telnet</artifactId>
  <version>3.5</version>
</dependency>
```

### HTTP/WebSocket Support

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>terminal-http</artifactId>
  <version>3.5</version>
</dependency>
```

## Requirements

- Java 8 or higher
- Maven 3.6+ (if building from source)
