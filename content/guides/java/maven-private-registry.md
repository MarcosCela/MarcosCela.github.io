---
title: "Setting up a private maven mirror"
date: 2019-03-04
description: "Setting up a private maven mirror, for faster downloads and reduced latency wen downloading dependencies."
featured_image: "/images/java/maven.png"
categories:
- Java
- Tutorial
- Maven
tags: ["maven", "java"]
include_toc: true
---
# What is a private Maven mirror?

When you download dependencies with maven, you usually do it via [maven central], and this
might not be the best solution. Having a local copy of the maven repository is specially
useful if you continually need to download dependencies:

- You avoid going through the internet, this can have a huge impact in the speed
  at you download dependencies.
- If you have a CI system that "starts from scratch" because it uses VM's or containers on
  demand and always starts from a fresh filesystem, you will need to download dependencies
  every time. This is one of the places where you will see a bigger difference.
  
# How does it work?

The usual implementations rely on a local cache. This means that every time you download
a dependency, your mirror will download it from the central maven repository, save it, and
then serve it to you. Subsequent downloads of the same dependency will not be downloaded again,
and therefore you save time, bandwidth, and increase your coolness factor (results may vary).

# Actual tutorial

Okey, maven saves configuration files in ***~/.m2/settings.xml***, so we go there and
do domething like this:
```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" 
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
  https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>central-mirror</id>
      <username>myuser</username>
      <password>{encrypted password}</password>
    </server>
  </servers>

  <mirrors>
    <mirror>
      <id>central-mirror</id>
      <name>Maven central mirror</name>
      <url>https://my.nexus.org/repository/maven-central</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
</settings>
```

And how the hell do you obtain the "encrypted password", well, first of all you will
obtain a master maven password:

```bash
mvn --encrypt-master-password <very safe password>
>{AvE5...ME=}
```
This basically creates a main encryption password, that is used to encrypt other passwords
so you don't store plaintext credentials in your ***setings.xml***. This is useful because
you can then copy your ***setings.xml*** to your CI system (obviously NOT to a public
place, this is intended for your organization internal CI system)..

The given password goes in a   **~/.m2/settings-security.xml** file like this:

```xml
<settingsSecurity>
  <master>{AvE5...ME=}</master>
</settingsSecurity>
```

With this file in place, you can then create your user password and encrypt it:
```bash
mvn --encrypt-password <my-user-password>
```

And off we go, you can now easily download dependencies using your private mirror.

[maven central]: https://search.maven.org/