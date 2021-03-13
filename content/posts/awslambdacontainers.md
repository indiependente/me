---
title: " ƛ📦 AWS Lambdas containers and Go"
date: 2021-03-13T15:05:00Z
toc: true
images:
tags:
  - aws
  - lambda
  - container
  - docker
  - golang
  - distroless
  - apigateway
---

## TL;DR

AWS official base images for lambdas ship entire distros, use scratch or distroless base images for statically compiled languages like Go and use the [aws-lambda-runtime-interface-emulator](https://github.com/aws/aws-lambda-runtime-interface-emulator/) as the entrypoint.

## Intro

In December 2020, [AWS announced](https://aws.amazon.com/blogs/aws/new-for-aws-lambda-container-image-support/) support for packaging and running lambdas in containers.

In this blog post I'll try to dig a little bit deeper on how it works and how we can improve what's described in the official article, so that we can run an example Go application as a Dockerised lambda both on AWS and locally in the most space efficient way.

Reading the [official docs](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html) by AWS might be beneficial if you're new to the topic.

You can find all the code which I'm referring to in this repo: [github.com/indiependente/aws-lambda-container](https://github.com/indiependente/aws-lambda-container).

## Don't use AWS base image

Why? You may ask.
Because ***it's huge!***

![Huge docker image](https://media.giphy.com/media/fSYmbgG5Ug8S11K0FU/giphy.gif)

The official image `public.ecr.aws./lambda/go` provided by AWS ships a whole Linux distro and weights about **670MB**, which is definitely too much.

## Distroless images

Using the images provided by the *Google Container Tools* here <https://github.com/GoogleContainerTools/distroless> makes a substantial difference.

In particular, the [example Dockerfile](https://github.com/indiependente/aws-lambda-container/blob/master/Dockerfile) uses the `gcr.io/distroless/static` image which is ideal for statically compiled languages like Go.

The resulting image weights about **10MB**.

## Local execution

So the whole reason for this being interesting is that it allows developers to improve their feedback loop when working with lambdas, without having to use external tools like [SAM](https://aws.amazon.com/serverless/sam/).

**But**, in order to use a custom image, you need to either bake the [aws-lambda-runtime-interface-emulator](https://github.com/aws/aws-lambda-runtime-interface-emulator/) into the image or install it on the host machine and point the Docker entrypoint to that executable.

The approach that resulted more convenient for me was to bake it into the local test image, following these steps:

- build the app and copy it into a distroless image (multi-stage docker build)
  - this is the lambda image that can be pushed to ECR
- build a test image that uses the lambda image as a base layer and uses the `aws-lambda-rie` as entrypoint.

Let's follow this process step by step.

## How can I test this lambda locally?

0. **Dependencies**

    - `docker`
    - `make`
    - `git clone https://github.com/indiependente/aws-lambda-container.git`

1. **Package the application**

    ````bash
    make lambda
    ````

    **OR**

    ```bash
    docker build -f Dockerfile -t fastfib:latest .
    ```

    This will build the docker image and tag it as `fastfib:latest`

2. **Package the test image that will be executed locally**

    ```bash
    make testlambda
    ```

    **OR**

    ```bash
    docker build -f Dockerfile.test -t testfastfib:latest .
    ```

    This will build the docker image and tag it as `testfastfib:latest`

3. **Run the lambda locally**

    ```bash
    docker run -p 9000:8080 testfastfib:latest
    ```

    Here I'm mapping port `8080` of the container to `9000` on the host machine but that's not strictly necessary.

4. **Send a request to the lambda**

    The code itself implements a fast fibonacci sequence algorithm based on <https://www.nayuki.io/page/fast-fibonacci-algorithms>.

    So the lambda will reply with the *n-th* element of the Fibonacci sequence in JSON content encoding.

    But, where is the lambda listening on?

    As the AWS docs show, the lambda is listening on port `9000` (that I've explicitly set) at the following address:
    `http://localhost:9000/2015-03-31/functions/function/invocations`

    **/2015-03-31/functions/function/invocations**

    *That makes total sense.*

    ![Jim doesn't like that endpoint](https://media.giphy.com/media/HP7mtfNa1E4CEqNbNL/giphy.gif)

    Anyway, let's `curl` that endpoint.

    Request:

    ```bash
    curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{"n":7}'
    ```

    Response:

    ```json
    {"result":13}
    ```

Yes! We've got our JSON response back!

## API Gateway Proxy Request/Response

Lambdas can be sitting behind an API gateway or a load balancer, so let's try to simulate that interaction.

In this example I've added an API gateway handler to the lambda code that can be used by passing the `--apigw` flag when running the binary.

There is a specific `CMD` in the [Dockerfile.test](https://github.com/indiependente/aws-lambda-container/blob/master/Dockerfile.test) file, that can be enabled to test this type of event.

```Dockerfile
CMD ["--apigw"]
```

In order to test the API gateway handler, we need to send an API gateway JSON event.
There is a sample one called [apigw_request.json](https://github.com/indiependente/aws-lambda-container/blob/master/apigw_request.json) in the repo, that can be used to test locally.

Request:

```bash
curl -i -X POST localhost:9000/2015-03-31/functions/function/invocations \
  -H "Content-Type: application/json" \
  --data-binary "@apigw_request.json"
```

Response:

```http
HTTP/1.1 200 OK
Date: Wed, 02 Dec 2020 18:44:33 GMT
Content-Length: 115
Content-Type: text/plain; charset=utf-8

{"statusCode":200,"headers":{"Content-Type":"application/json"},"multiValueHeaders":null,"body":"{\"result\":233}"}
```

You might notice here that the response's content type is `text/plain`, that's because we're simulating the interaction happening between lambda and API Gateway.

The API Gateway will then unbox that response and craft a proper HTTP response to send to the client.

## Conclusions

We've seen how to build and run a lambda function built as a 10MB Docker image and simulate the interaction with one of the possible lambda triggers.

Hopefully this post helped someone who, like me, wanted to use a custom base image with a Go lambda function and run it locally.

Thanks for reading.

![Thanks](https://media.giphy.com/media/a3IWyhkEC0p32/giphy.gif)

## References

- [AWS announcement](https://aws.amazon.com/blogs/aws/new-for-aws-lambda-container-image-support/)
- [AWS Docs](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)
- [AWS Lambda runtime interface emulator](https://github.com/aws/aws-lambda-runtime-interface-emulator/)
- [Google Distroless Images](https://github.com/GoogleContainerTools/distroless)
