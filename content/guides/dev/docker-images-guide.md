---
title: "Guidelines to build sane Docker images"
date: 2019-03-21
description: "Some basic (and not so basic) recommendations and guidelines to ensure that your Docker builds follow best practices"
featured_image: "/images/dev/docker.png"
categories:
- Tutorial
- Docker
tags: ["Docker", "Dockerfile"]
include_toc: true
---

# Building docker containers is easy, dangerous and can cause death

You can build a Docker container with a lot of different ways, some are good, some are bad
and others are REALLY bad.

The following document will try to give some basic rules and guidelines on how
to build a Docker container that is as light as possible, simple, has valuable metadata,
and doesn't contain any unwanted secrets.

# Rules

These are rules. Rules have exceptions, but make sure that you understand the need for the rule,
so you can really understand if you are an exception, or there is a workaround.

* [Thou shalt not use docker commit, because it is the root of all evil](#docker-commit)
* [You should ***never*** add credentials to a Docker image (or other unwanted files)](#credentials)

# Best practices
Not as hard as the rules, but always "nice to have".


* [You should add "metadata" to your container (Use of the VOLUME, PORT and LABEL directives)](#metadata)
* [You should not include non-used packages (it is ok for testing, but wastes space on production builds)](#non-used-packages)
* [You should version your images (do not re-use the "latest" tag for everything)](#version)
* [Your image should have a "default" entrypoint](#entrypoint)
* [You should use multistage builds to enable smaller final size on images](#multistage)
* [You should use a non-root user for your containers](#non-root)
* [Use a .dockerignore to ignore paths that are not included in the Docker container](#dockerignore)


# Thou shalt not use docker commit, because it is the root of all evil {#docker-commit}

When building a Docker image, there is always the temptation to start with an image that
we know:

```bash
docker run -it --entrypoint=/bin/bash ubuntu:latest
```

And then install packages, add configurations... And commit the image!
But wait... you did a terrible thing, because I have no idea of what packages you installed,
which versions, in what order or for what reason. And this is bad!

The reasons to never use docker commit are:

- You cannot reproduce the build with ease. If you go on vacation and your team has to make a
  critical change to the image, there is only one way of doing it... Doing a commit again, or
  creating a Dockerfile with your image as base.

- You cannot clearly see what are the contents of the image: does it have a keylogger? does it
  have your personal access token because you forgot to delete it? what packages are installed?
  
  
The alternative to this is easy, is the way that the guides tell you to build a Docker container
and is the good way that the gods intended. Just use a Dockerfile.

***Note***: To every rule there is always an exception. Sometimes installing packages require
that you accept some kind of license, or require some kind of input that cannot be easily automated.
In this case, MAYBE the only way to build the container is docker commit. Either way, you should
document the process, and try really hard to avoid it.

# You should ***never*** add credentials to a Docker image (or other unwanted files){#credentials}

You are probably thinking... I am not an idiot, I will NEVER ever add my credentials to a 
Docker image... Well, there are a lot of cases of people accidentally pushing secrets to
public GitHub repositories, so... shit happens, we are human and even a little mistake
can cost a lot:


- [What happens if you accidentally commit your aws access token to public github]

The usual suspects of adding credentials are not "well known" credentials, but automatic files. An example
of this are:

- ***~/.aws/credentials***: this one is hard to commit, and usually people know where it is and to treat
  it with respect. It is also not very common to build docker containers in the home directory, this
  (and other cloud provider credentials) are an easy way of getting a hefty bill from AWS.
  
- ***requirements.txt***: Because if you configure your servers and credentials in the requirements.txt, you
  could end up exposing them to the public.

 ```text
   index-url = http://user:password@localhost:8081/artifactory/api/pypi/pypi-virtual/simple
  ```

- other: there are a ton of files that are expected on the "root" of a repository that contains credentials.
  These files will then be sent to the docker daemon (if not ignored), and once they are there, there is a
  high chance that they end up in the final image.

Just ensure that you ***really really*** know what is going inside your Docker image at all times, and if
you have any doubt, start the container with an interactive shell and check manually!

# You should add "metadata" to your container (Use of the VOLUME, PORT and LABEL directives){#metadata}

This is not a rule, but it is always nice to quickly see what volumes your image uses, what port
exposes, and other useful labels (such as a maintainer, commit ID or other). This information
is usually not required, but really useful:

- If someone gets a Docker image and doesn't know what port it exposes (not documented), he/she can
  easily inspect the image and see the exposed ports in the metadata. The same goes for volumes.
  
  
- With labels, you can see a maintainer, a commit ID... these are useful in the event that you need
  to debug the image, get in contact with the original developer, or simply knowing which version
  of a software is inside the container without starting it.
  
  
As with all, metadata is also used sometimes for some services that scan images and then show information
such as exposed ports without starting the container.





# You should not include non-used packages (it is ok for testing, but wastes space on production builds){#non-used-packages}

This not only wastes space, but adds unnecessary complexity and additional attack surface to your containers.
It's okey to include net-tools when you are debugging inside a container, but remember to remove them
for the final (production) image.



# You should version your images (do not re-use the "latest" tag for everything){#version}

There is a lot of controversy here, but I PERSONALLY like the following:

- Latest tag is the latest, stable version of the image. I am expected to run this image if I want to
  test what the image does.

- Additional tags should also be added in notorious "landmarks". Usually, the (my) preferred way is something like:
  ```text
   my-image:v1.0
  ```
  But there is a lot of people that want to include additional information such as the commit ID for automatically
  generated images, so it ends up with something like:
  ```text
  my-image:c81bbc89804e9780121c4e9789322f008eea6e7f
  my-image:v1.0-c81bbc89804e9780121c4e9789322f008eea6e7f
  my-image:v1.0-c81bbc89804
  my-image:v1.0-c81bbc89804-RC
  ...
  ```
  You get the idea.
  
  
This ensures basically two things:

- Users of your image can select a version if they want to rely on it (e.g: v1.2.5) with a known
  working configuration, that should not have breaking changes.

- Adventurous users can quickly deploy the latest version without knowing if your versions start at 1,
  at 15.3.2, or you use named releases (like OpenStack). Or if you don't even have named/versioned releases.
  
  
This is nothing new, it has been applied to software since the beginning of the universe. Just apply it to Docker!


# Your image should have a "default" entrypoint {#entrypoint}

This is really helpful for two things:

- Users can immediately see what your image does without relying on documentation.
- Users can inspect the entrypoint to see what the image does. Some images have a basic shell script
  that calls the actual process and just wraps the starter (usually checks environment variables or
  does some kind of preparation of the container).

# You should use multistage builds to enable smaller final size on images {#multistage}

This is a no-brainer, and really recommended. You basically use a container to build an artifact (think
of binaries in go or c/c++), and then copy these artifacts to a very small, final image.

This way, you ensure that your final image is extremely small (usually the size of the binary + a little bit more).

There are a lot of good articles on how to generate multi-stage containers:

- [Official docker multistage build]
- [@tonistiigi on medium]
- [Bruno paz on dev.to]

# You should use a non-root user for your containers {#non-root}

Usage of the least privilege pattern is a good thing. By using a non-root user for our images, we limit
potential security breaches to what the running user can see inside the container (the worst problem is for
containers that mount the host filesystem, as root). Some really good articles on this:

- [@mccode on medium: UID and GID on docker]
- [@mccode on medium: do not run containers as root]

# Use a .dockerignore to ignore paths that are not included in the Docker container {#dockerignore}

By using a .dockerignore, we obtain the following benefits:

- We can exclude sensitive files from the final Docker image.
- We can exclude generated/non-necessary files from the final Docker image.
- The context sent to the Docker daemon will be smaller, hence faster.
- Do not accidentally expose your .git folder to the world (this usually happens with Python images).

[@mccode on medium: UID and GID on docker]: https://medium.com/@mccode/understanding-how-uid-and-gid-work-in-docker-containers-c37a01d01cf
[@mccode on medium: do not run containers as root]: https://medium.com/@mccode/processes-in-containers-should-not-run-as-root-2feae3f0df3b
[Bruno paz on dev.to]: https://dev.to/brpaz/using-docker-multi-stage-builds-during-development-35bc
[@tonistiigi on medium]: https://medium.com/@tonistiigi/advanced-multi-stage-build-patterns-6f741b852fae
[Official docker multistage build]: https://docs.docker.com/develop/develop-images/multistage-build/
[What happens if you accidentally commit your aws access token to public github]:  https://medium.com/@selvaganesh93/what-happens-if-you-accidentally-commit-your-aws-access-token-to-public-github-be50d378b4c7