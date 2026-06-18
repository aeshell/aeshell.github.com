---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Installation'
weight: 1
---

## Maven

Add the dependency to your `pom.xml`:

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>readline</artifactId>
  <version>3.15</version>
</dependency>
```

### Using the BOM

For multi-module projects, the BOM ensures consistent versions across all aesh-readline modules:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.aesh</groupId>
      <artifactId>aesh-readline-bom</artifactId>
      <version>3.15</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.aesh</groupId>
    <artifactId>readline</artifactId>
  </dependency>
</dependencies>
```

## Gradle

Add the dependency to your `build.gradle`:

```groovy
dependencies {
    implementation 'org.aesh:readline:3.15'
}
```

Or with the BOM:

```groovy
dependencies {
    implementation platform('org.aesh:aesh-readline-bom:3.15')
    implementation 'org.aesh:readline'
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

Aesh Readline includes modules for remote connectivity:

### SSH Support

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>terminal-ssh</artifactId>
  <version>3.15</version>
</dependency>
```

### Telnet Support

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>terminal-telnet</artifactId>
  <version>3.15</version>
</dependency>
```

### HTTP/WebSocket Support

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>terminal-http</artifactId>
  <version>3.15</version>
</dependency>
```

## Java Version Requirements

- **Java 8+**: Full functionality via JNI/exec-based terminal I/O
- **Java 22+**: FFM-based terminal I/O for better performance (add `--enable-native-access=ALL-UNNAMED`)
- **Maven 3.6+** for building from source

See [GraalVM Native Image]({{< relref "native-image" >}}) for native compilation support.
