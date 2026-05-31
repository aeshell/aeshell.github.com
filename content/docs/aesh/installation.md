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
  <artifactId>aesh</artifactId>
  <version>3.6.1</version>
</dependency>
```

## Gradle

Add the dependency to your `build.gradle`:

```groovy
dependencies {
    implementation 'org.aesh:aesh:3.6.1'
}
```

## Annotation Processor (Optional)

To generate command metadata at compile time (5-8x faster startup, no runtime reflection, GraalVM-friendly), add the `aesh-processor` dependency. See [Annotation Processor](../annotation-processor) for details.

### Maven

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>aesh-processor</artifactId>
  <version>3.6.1</version>
  <scope>provided</scope>
</dependency>
```

### Gradle

```groovy
dependencies {
    annotationProcessor 'org.aesh:aesh-processor:3.6.1'
    compileOnly 'org.aesh:aesh-processor:3.6.1'
}
```

No code changes are required -- Aesh automatically detects and uses the generated metadata when available.

## Build from Source

Clone the repository and build with Maven:

```bash
git clone https://github.com/aeshell/aesh.git
cd aesh
mvn clean install
```

## Requirements

- Java 8 or higher
- Maven 3.6+ (if building from source)
