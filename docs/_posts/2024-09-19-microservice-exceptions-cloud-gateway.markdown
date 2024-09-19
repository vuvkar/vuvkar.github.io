---
layout: post
title: "[Spring Cloud] Exception handling with body modifiers in the route!"
date: 2024-09-19
categories: Spring Cloud
---

### Introduction

In the project I am working on, we have a microservice architecture. Over these microservices, a gateway sits to protect and handle all calls. In most cases, the gateway merely proxies requests, with no business logic involved. However, there are cases where we need to enrich the result of one microservice call by combining it with the result of another. To achieve this, we use the `modifyResponseBody` function in the route configuration.

Here’s an example of a typical route setup:

```java
return route
    .path(path)
    .and()
    .method(method)
    .filters(f -> {
        f.tokenRelay();
        f.modifyResponseBody(inClass, outClass, filter);
        return f;
    }).uri(serviceURL);
```

In this setup, `filter` refers to a class that implements the `RewriteFunction<inClass, outClass>` interface, which contains the following method:

```java
@Override
public Publisher<O> apply(ServerWebExchange serverWebExchange, I inObject) { ... }
```

Everything works well until an exception occurs in the microservice returning the `inObject`. When an error response is returned instead of the expected object, the gateway throws an error trying to convert the error response into the expected type.

### The Problem

When an exception is returned from the microservice, the gateway struggles to handle it correctly, leading to a failure when it tries to convert the error response. This happens because the body modifier expects a specific class (such as `inClass`), but the error response does not match this structure.

### The Solution

To solve this issue, we need to ensure that the response from the microservice is valid before attempting to map the body to the desired class. I considered two potential approaches:

#### 1. Using Filters

Spring Cloud has post-filters that trigger after receiving a response from the microservice. My initial idea was to use a filter to break the chain of body modifiers or other filters if the response is an exception. However, after some research, I found that it wasn't possible to break the route chain in post-filters.

#### 2. Creating an `AbstractResponseBodyModifier` Class

The second approach involved creating a base `AbstractResponseBodyModifier` class. This class checks if the response is a successful one (i.e., HTTP 2xx) before converting the response into the desired custom object. If the response is successful, we proceed with mapping the response body. Otherwise, we return the response as is without modifying it.

Here’s an example of the implementation:

```java
@Component
public abstract class AbstractResponseBodyModifier<I, O> implements RewriteFunction<DataBuffer, DataBuffer> {

    // field definitions and constructors

    @Override
    public Publisher<DataBuffer> apply(ServerWebExchange serverWebExchange, DataBuffer dataBuffer) {
        logger.trace("Applying filters to response body...");
        if (dataBuffer == null) {
            // When body is empty, data buffer object is null
            return Mono.empty();
        }
        if (serverWebExchange.getResponse().getStatusCode() != null && serverWebExchange.getResponse().getStatusCode().is2xxSuccessful()) {
            byte[] sourceBytes = new byte[dataBuffer.readableByteCount()];
            dataBuffer.read(sourceBytes);
            DataBufferUtils.release(dataBuffer);
            I object = null;
            try {
                object = objectMapper.readValue(sourceBytes, inputType);
            } catch (IOException e) {
                return Mono.error(e);
            }
            Mono<O> modifiedResponse = modifyResponseBody(serverWebExchange, object);
            return modifiedResponse.handle((e, sink) -> {
                try {
                    sink.next(rewriterUtils.toDatabuffer(e));
                } catch (JsonProcessingException ex) {
                    sink.error(new RuntimeException(ex));
                }
            });
        } else {
            logger.debug("Done applying filters to response body, response status was not successful");
            return Mono.just(dataBuffer);
        }
    }

    /**
     * Abstract method for modifying response body
     *
     * @param serverWebExchange
     * @param responseDataBuffer
     * @return
     */
    public abstract Mono<O> modifyResponseBody(ServerWebExchange serverWebExchange, I responseDataBuffer);
}
```

This solution ensures that before attempting to modify the response body, we first verify if the HTTP status code is 2xx. If the status code indicates an error, the original response is returned without modification.

### Modifying the Route Configuration

Once we implement the `AbstractResponseBodyModifier`, we need to update the route configuration to tell the gateway that we are working with `DataBuffer` objects, not custom objects.

```java
f.modifyResponseBody(DataBuffer.class, DataBuffer.class, filter);
```

This change informs the gateway that it doesn't need to perform automatic mapping, as our custom modifier will handle the conversion manually.

### The Downside

A potential downside of this approach is that we take over the responsibility of mapping objects manually. This is fine as long as we are confident about the structure and content of the expected results. However, if the structure of the responses changes, the custom mapping logic will need to be updated accordingly.

### Conclusion

Handling microservice exceptions in Spring Cloud Gateway when using body modifiers can be tricky, but by creating a custom `AbstractResponseBodyModifier`, we can ensure that error responses are handled gracefully without breaking the response pipeline. This approach keeps the gateway lightweight and ensures that business logic is managed in the microservices themselves.

I hope this solution helps anyone facing similar challenges in their Spring Cloud Gateway setups!

#SpringCloudGateway #Microservices #ExceptionHandling #BodyModifiers #RewriteFunction #AbstractResponseBodyModifier #SpringBoot #Java #GatewayArchitecture #ErrorHandling #APIGateway #SpringCloud #WebFlux
