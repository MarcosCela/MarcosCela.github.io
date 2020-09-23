---
title: "Setting up GO_PROXY for private repositories"
date: 2020-09-23
description: "Setting up your GO_PRXOY and GNOSUMDB to access private repositories via a proxy"
featured_image: "/images/dev/gopher-proxy.png"
categories:
- Tutorial
- GO
tags: ["Go", "Tutorial"]
include_toc: true
---

# Using GO_PROXY, GONOSUMDB and friends

Since go 1.13, we can configure a proxy that will be used to download packages. This can be done via the
environment variable ```GOPROXY```. By default, the value is```GOPROXY=https://proxy.golang.org,direct```.
This means that packages will be fetched via the given proxy, and falling back to "direct" (via git) if
the proxy cannot find the package.

This way of downloading files also integrates with a checksum database, by default at ```sum.golang.org```.
Every time a module gets downloaded via GOPROXY, a request is made to the checksum database, ensuring
that the downloaded content checksum matches the one registered.
This avoids MitM attacks, and moves the "trust" to a GOSUMDB, instead of the proxy.


## Configuring GOPROXY on CI environments

Let's say that you have a valid Golang proxy serving at ***https://my.proxy.org/api/go/myrepo***. This proxy transparently
proxies other endpoints such as github or gocenter, and additionally, it proxies private github repositories. This is the
case for jfrog, when you can select a private repository (myrepo in our case), that will be proxying to multiple locations,
such as github, gocenter or other private repositories under the same virtual repository.

We need to configure our ```GOPROXY``` variable to point to this proxy, setting credentials if required:

```shell script
export GOPROXY=https://https://myuser:mypassword@my.proxy.org/api/go/myrepo
```

With this, every module will be downloaded via the GOPROXY. We can also set some fallbacks separated by commas:

```shell script
export GOPROXY=https://https://myuser:mypassword@my.proxy.org/api/go/myrepo,direct
```


The exception is that ***direct*** terminates the list, so you can't set more proxies after direct. Direct means
downloading the module via ***git***.


With this setup, we are basically performing the following actions:

- Authenticate with ***my.proxy.org***.
- The ***my.proxy.org*** will download packages from several sources, and some sources might be private, but the
  proxy is configured with valid credentials (e.g: private repositories in github).


The problem now is that we might get errors for mismatched go.sum files and hashes. This is because by default,
go will look up ```sum.golang.org``` to check the hash of the downloaded module. If our module is private, it won't
be present in ```sum.golang.org```, and it will fail.

To avoid this we can either set the following variable:

```shell script
export GONOSUMDB=""
```

That will force go to "search" for a valid hash/sum in the proxy (if the proxy supports it). If your proxy does not
support it or you want to turn it off, you can define a set of modules that won't be validated against a sumdb (they
will be validated against your local go.sum). E.g:

```shell script
# It allows globbing!
export GONOSUMDB="github.com/myorg/*,github.com/anotherprivateorg/*"
```
