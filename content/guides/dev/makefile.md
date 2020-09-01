---
title: "Example super basic Makefile"
date: 2019-03-21
description: "A very small and simple example Makefile to simplify repetitive tasks when developing (such as testing, building...)"
featured_image: "/images/dev/makefile.png"
categories:
- Tutorial
- Makefile
tags: ["make", "makefile"]
include_toc: true
---
# Example makefile

A valid Makefile can be REALLY hard, but can also be REALLY simple.
The following minimal Makefile allows for the following logic:

- It has a default goal set, "help". Invoking "make" will give you a very basic help output.
- The help goal prints some nice information about what the makefile does.
- The image target prints a variable.

Be careful if you copy-paste this, Makefile requires tabs and not spaces!
 
```makefile
## Makefile directives
.PHONY: help image
.DEFAULT_GOAL := help

## Variables:
IMAGE := mydocker/image:3-alpine

help: ## Shows this help message
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

image: ## Prints the name of the image
    @echo $(IMAGE)
```


If we save this in a file "Makefile", and then invoke it:

```bash
make
```

Make will run the default goal ("help"). That results in the following output:
```bash
help                           Shows this help message
image                          Prints the name of the image


DOCKER_IMAGE                   mydocker/image:3-alpine
```

As we can see it prints the name of the goal, and the comment corresponding to the same goal.
We can then use this to expand on. As an example, if I have a repository that contains a Docker image, I like
to include the following Makefile:

```makefile
## Makefile directives
.PHONY: help image
.DEFAULT_GOAL := help

## Variables:
IMAGE :=test/example:latest

help: ## Shows this help message
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

build: ## Build docker image
    @docker build -t $(IMAGE) .
    
push: build ## Push the docker image to the docker registry
    @docker push $(IMAGE)

```

And then if I want to quickly push a new Docker image, I can just:

```bash
make push
```

It will perform the following actions:

- Build the docker image (because the push goal has a dependency on **build**).
- Push the image.

Obviously, these examples are really simple, but they also have a LOT of implicit information:

- The name of the Docker image.
- The repository where it is going to be pushed.
- Other build-time details:
    - Sometimes you have to build image with special parameters, such as --network=host. With this approach,
      if someone leaves your team you can automatically know these details that are otherwise very hard to catch,
      and let's be real, nobody documents these.
      
      
It basically serves as a self-documenting build script. I try to keep it as simple
as possible and avoid bash/make magic syntax. For me, a simple Makefile helps a lot when
getting started with a new project.

Some recommendations:

- Keep it as simple as possible. The Makefile syntax is not easily understandable,
  keep it as simple as you can. If you need something complex... use a complex build system
  like maven, npm, CMake...

- Avoid "Makefile" magic. As a rule of thumb: if you have to look up what something is doing
  either add a comment, or avoid using it.