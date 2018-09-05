# Response classification

Within a (e.g., HTTP) service, each request may have differing properties,
especially as we think about _success_. Some routes are idempotent, some are
not. Some services consider 404s to be a failure, others do not.

It's important to have such a nuanced idea of success classification for the
observability (i.e., calculating a success rate) and reliability
(automatically retrying certain classes of failures, circuit breaking, etc).

Recent `linkerd2-proxy-api` changes provide a new API that describes response
classification in a per-request way.

This document outlines a proposal for implementing a modular response
classification system in linkerd2-proxy. This may be suitable to graduate
into the tower project if it's successful in Linkerd.


## Background

linkerd2-proxy primarily comprises two _proxies_ (inbound and outbound). Each
proxy is responsible for accepting connections on a TCP port, determining the
protocol of the connection, and forwarding it appropriately.

HTTP and HTTP/2 connections are decoded and routed. As requests are routed,
the proxy discovers the response classification scheme for each service.

We model each route as a "stack" of services. This stack is divided into a
few logical layers:

- The outermost _dst stack_ is associated with the logical destination (i.e.
  hostname). This stack will hold retries, etc. The innermost layer of this
  stack is a load balancer.

- Within each load balancer, as new endpoints are discovered, an _endpoint
  stack_ applies endpoint-specific logic as driven by the load balancer.

For example, we can describe an HTTP router as a stack of modules like the following:

```
+ HTTP Router
`-- Dst stack:
    + Measure
    + Retry
    + Balance
    `-- Endpoint stack:
        + ServicePerRequest
        + Reconnect
        + Measure
        + tower_h2 / Hyper
```

In this example, the _retry_ and _measure_ layers need to determine the
response classification for each request: _retry_ uses the response
classification to determine whether a request should be retried; and
_measure_ uses the classification to scope metrics.

## Proposal

### Goals

1. Make classification decisions uniformly for a request (i.e. so that different layers of the stack must not make independent determinations).
2. Support end-of-stream classification (i.e. for grpc-status, etc).

In order to implement classification, I propose introducing a new `Classify` API that

```rust
/// Given a request, produces a `Classify` that can produce a `Class` for the
/// response.
pub trait NewClassify<E> {
    type Class;
    type Classify: Classify<Class = Self::Class, E>;
    fn classify(&self, req: &http::request::Parts) -> Self::Classify;
}

pub trait Classify<E> {
    type Class;

    fn start(&mut self, headers: &http::response::Parts) -> Option<Self::Class>;
    fn end(&mut self, headers: Option<&http::HeaderMap>) -> Self::Class;
    fn error(&mut self, error: &E) -> Self::Class);
}
```
