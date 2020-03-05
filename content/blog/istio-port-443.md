---
author: "Dominic Gunn"
title: "Istio, Port 443 and SSL errors."
date: 2020-02-05T20:25:03+01:00
categories:
  - "istio"
  - "kubernetes"
  - "service mesh"
  - "SSL"
  - "routines:SSL23_GET_SERVER_HELLO:unknown"
---

It's been a while since I've written one of these, but this issue irked me and I felt I had to write something. [Turnitin](https://turnitin.com) is a pretty early adopter of some new technologies, Kubernetes and Istio haven't been around all that long but in the companies shift towards the cloud they're at the forefront of our charge. Bleeding edge isn't always fantastic.

I like to think that we've got a pretty fleshed out CI/CD pipeline that enables our engineering teams to get their code from their laptop to our regional Kubernetes clusters safely, quickly and importantly in a way that fits within the teams own SDLC. To support this, we have a set of tools that allow engineers to provide us with nothing more than a set of `value.yaml` files that we dump into a highly configurable Helm chart, we got the idea from [TicketMaster](https://ticketmaster.com), they give a great talk on it [here](https://www.youtube.com/watch?v=HzJ9ycX1h0c). Our "plug and play" type setup is very similar.

## Istio!

We've been running Istio for a little while now, we like it. It replaced a service mesh management system that we built ourselves in the early days of Kubernetes and has solved a lot of our problems. In our customizable helm chart, we've done our best to make sure engineers are agnostic to the fact that Istio is running, and instead simply define routing rules in a way that we as operators would be able to take those values and template them against any sort of mesh technology, take the following example.

To expose an endpoint for their service, and have Istio issue retries on certain policies they simply have to give us the following set of values. This is great! The engineers love it, we can template it safely and everybody is a winner.

```yaml
externalRouting:
 pathConfiguration:
   - path: /my-retry-path
     retryOn: 5xx
     retryAttempts: 2
```

### Read the documentation

Did you know that Istio has some very strict rules on how ports on services [should be named?](https://istio.io/docs/ops/configuration/traffic-management/protocol-selection/). We did, we even wrote code to guard against deploying a service with a port name that Istio wouldn't accept. It's super simple and looks like this.

```go
govalidator.TagMap["portName"] = govalidator.Validator(func(str string) bool {
    for _, prefix := range []string{"grpc", "grpc-web", "http", "http2", .... } {
        if strings.HasPrefix(str, prefix) {
            return true
        }
    }
    return false
})
```

There, nice and safe.

### [SSL: WRONG_VERSION_NUMBER] (_ssl.c:852)

Uh oh. Suddenly everything in the cluster is dead. We can't make any calls to SSL/TLS services over HTTPS anywhere, to anything internal or external. All calls are failing.

```bash
$ curl https://www.google.com
curl: (35) error:140770FC:SSL routines:SSL23_GET_SERVER_HELLO:unknown protocol.
```

### What happened? 

Somebody deployed a `Service` with the following port block.

```yaml
  ports:
  - number: 443
    name: http
    protocol: HTTP
```

Perhaps innocent looking at first, listen on port 443 using the `http` protocol. A little backward I admit and not something I'd expect, but why is everything dead? What happened? Take a look at [this](https://github.com/istio/istio/issues/16458), a still open and unresolved issue. 

It turns out, if you have anything listening on port 443 that doesn't have a port name prefixed with _https_ then you're going to have a really bad time. Be careful [kiam users](https://github.com/uswitch/kiam/blob/master/helm/kiam/templates/server-service.yaml#L38)!

### What next?

So how did we protect against this? Remember engineering teams give us all of this information to template for them. We write more code.

```go
if p.Port == 443 {
    if p.Name != "https" && !strings.HasPrefix(p.Name, "https-") {
        msg := fmt.Sprintf("`name` must begin with 'https' since `port` is 443 but is '%s'", p.Name)
        return errors.New(msg)
    }
}
```

I hope this helps somebody else, it caught us off guard.