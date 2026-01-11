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
  <artifactId>aesh</artifactId>
  <version>1.7</version>
</dependency>
```

## Gradle

Add the dependency to your `build.gradle`:

```groovy
dependencies {
    implementation 'org.aesh:aesh:1.7'
}
```

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
