---
title: "s3stream"
date: 2019-03-22T19:11:11Z
toc: false
draft: true
images:
tags:
  - go
  - module
  - aws
  - s3
  - streaming
---
## Setup

```go
  import "github.com/indiependente/s3stream"
  // where conf is &aws.Config
  store := s3stream.NewStoreWithClient(s3.New(session.Must(session.NewSession(conf))))
```

## Get

```go
rc, err := store.Get(prefix, bucketname, filename)
```

## Put

```go
n, err := store.Put(prefix, bucketname, filename, contentType, r)
```
