---
date: '2024-10-26T13:22:03+03:00'
draft: false
title: 'Writing an HTTP Router for AWS Lambda Functions From Scratch With Go'
---
Just give me the code! [Github](https://github.com/D-Andreev/lambdamux)

## Introduction

The options for deploying your REST APIs are numerous these days, spanning a wide spectrum. On one end, you have the bare metal approach of using your own server where you have full control over the infrastructure but also have to manage much more, while on the other end, you can leverage platforms and services that abstract away much of the infrastructure management but also limit your control over the underlying technology.

For a project I'm working on I decided to use serverless and deploy my Go APIs on AWS Lambda. Serverless comes with several advantages:

- Pay-per-request pricing 
- No need to manage infrastructure
- No need to worry about scaling and capacity planning

However, there are also some downsides:

- Cold starts
- Vendor lock-in 
- Potential for higher costs (Applications with consistent high volume traffic or long running tasks are not suited for serverless)

I decided to split the work of my API into several services, each deployed on its own Lambda function. For example, I have a user service with several endpoints for user management:

 - POST /users (Create a new user)
 - GET /users/:id (Get a user by ID)
 - PUT /users/:id (Update a user)
 - DELETE /users/:id (Delete a user)

 And a product service with endpoints for managing products:

 - POST /products (Create a new product)
 - GET /products/:id (Get a product by ID)
 - PUT /products/:id (Update a product)
 - DELETE /products/:id (Delete a product)

Although this is simple example, it became apparent that I would need to do http routing inside each of the Lambda functions.

After some research I found out that the most popular approach is to use [aws-lambda-go-api-proxy](https://github.com/awslabs/aws-lambda-go-api-proxy) along with your favorite HTTP router like [Gin](https://github.com/gin-gonic/gin) or [Chi](https://github.com/go-chi/chi). While this approach let's you use a popular HTTP router, it comes with the overhead of converting the `APIGatewayProxyRequest` to `http.Request`, so that it can be processed by the router. Another good HTTP routing library I found is [LmdRouter](https://github.com/aquasecurity/lmdrouter) which works directly with the `APIGatewayProxyRequest` and doesn't have the overhead of the conversion.  The router implementation uses a hashmap to store the keys, however, matching a route to a handler is still `O(n)` where `n` is the number of routes. It iterates every route and uses a regex to match the path. Considering I wanted my lambda functions to execute as fast as possible to decrease request time and possibly reduce costs I started to wonder if I could do better.

## LambdaMux

 After some research on how HTTP routers work under the hood I decided to take a small detour from my project and implement a simple HTTP router from scratch.

The initial requirements were simple:

 - Match a request url to a handler function (Preferably better than `O(n)` time complexity)
 - Support multiple HTTP methods (GET, POST, PUT, DELETE)
 - Support URL parameters (e.g. /users/:id)
 - Don't do any conversions between the request types (`APIGatewayProxyRequest` and `http.Request`)
 - Easy to use API, similar to existing HTTP routers


## What is a radix tree?

The [radix tree](https://en.wikipedia.org/wiki/Radix_tree) is a tree data structure used to store strings that allows efficient retrieval of the longest matching prefix of a given string. Let's say we have the following routes:
- POST /pet
- POST /pet/:petId
- POST /pet/:petId/uploadImage
- PUT /pet
- GET /pet/:petId
- GET /pet/findByStatus
- GET /pet/findByTags
- DELETE /pet/:petId

A radix tree with the following routes would look like this:

![Image of Radix tree with the example rotues](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ku1v4drdrupt5fqtgzzb.png)

## Inserting Routes 

One of the key operations in building our radix tree-based router is inserting new routes. This process involves navigating and potentially modifying the existing tree structure. When inserting a new route, we encounter four main scenarios:

1. Exact match: Update an existing node's handler.
2. Prefix match: Traverse down the matching path.
3. New branch: Create a new node for a completely new route.
4. Node splitting: Split an existing node when a new route partially matches but then diverges.

Node splitting is crucial for maintaining the tree's efficiency. It creates a new branching point, preserving the common prefix and separating the divergent paths.

### Basic Insertion Process:

1. Start at the root of the tree.
2. Get the first child node that matches the first character of the new route.
3. If there is no matching child node, add the new route as a child to the current node.
4. If there is a matching child node, get the longest common prefix between the new route and the child node's path and continue traversing the tree.
5. At some point there won't be a matching child node, so just create the new route as a child to the current node.

Let's consider an example to illustrate this process, including a case where we need to split a node.

Example:
Suppose we have the following routes already in our tree:
- POST /pet
- POST /pet/:petId

Our tree might look like this:
```
root
└── POST /pet
    └── /:petId
```

Now, let's say we want to add a new route: `PUT /pet/:petId`

Here's how the insertion process would work:

1. Start at the root and compare `"PUT /pet/:petId"` with `"POST /pet"`.
2. We find a common prefix `"P"`.
3. We create a new node with the common prefix `"P"` and add the two children `"UT /pet"` and `"OST /pet"` to it.
4. We set the new split node as the child of the root node.

After splitting, our tree would look like this:

```
root
└── P
    ├── UT /pet/:petId
    └── OST /pet
        └── /:petId
```

This insertion process ensures that the tree remains balanced and optimized for route lookups.


## Matching Routes


Consider the following tree:

![Image of Radix tree with the example rotues](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ku1v4drdrupt5fqtgzzb.png)

Let's say that we send a request to `POST /pet/123/uploadImage`, we will visit the nodes in the following order:

`Root` -> `P` -> `OST /pet` -> `/:petId` -> `/uploadImage`

The additional complexity when using a radix tree for an http router comes from handling the dynamic segments in the url. For example, when sending a request to `POST /pet/123/uploadImage` we need to recognize that we've matched the `POST /pet/:petId/uploadImage` route and extract the `petId` from the url. The actual value `123` is saved and can be used in the handler function later.

By using a radix tree we can match a request url to a handler function in `O(m)` time, where `m` is the length of the url. 

## Benchmarks
I've created benchmarks to compare the performance of `lambdamux` to the other routers. The benchmarks are available [here](https://github.com/D-Andreev/lambdamux/blob/main/lambdamux_benchmark_test.go).

The router used in the benchmarks consists of 50 routes in total, some static and some dynamic.

| Router                                                                | Operations | Time per Operation | Bytes per Operation | Allocations per Operation | Using aws-lambda-go-api-proxy | % Slower than LambdaMux |
|--------------------------------------------------------------------------|------------|---------------------|---------------------|---------------------------|-------------------------------|--------------------------|
| LambdaMux                                                                | 382,891    | 3,134 ns/op         | 2,444 B/op          | 40 allocs/op              | No                            | 0%                       |
| [LmdRouter](https://github.com/aquasecurity/lmdrouter)                   | 320,187    | 3,701 ns/op         | 2,086 B/op          | 34 allocs/op              | No                            | 18.09%                   |
| [Gin](https://github.com/gin-gonic/gin)                                  | 289,932    | 4,081 ns/op         | 3,975 B/op          | 47 allocs/op              | Yes                           | 30.22%                   |
| [Chi](https://github.com/go-chi/chi)                                     | 268,380    | 4,384 ns/op         | 4,304 B/op          | 49 allocs/op              | Yes                           | 39.89%                   |
| [Fiber](https://github.com/gofiber/fiber)                                | 210,759    | 5,603 ns/op         | 6,324 B/op          | 61 allocs/op              | Yes                           | 78.78%                   |

As we can see `lambdamux` has good performance advantage over the routers using `aws-lambda-go-api-proxy`. `LmdRouter` is the closest competitor, however, because of its linear route match time complexity this difference will grow as the number of routes increase.

As for memory usage, `lambdamux` uses marginally more memory than `LmdRouter` and less than the other routers. I believe that some of this memory usage could be reduced by implementing node compression, compressing nodes with only one child into a single node, using fixed arrays instead of slices and possibly a few other optimizations.

**Note**: While `lambdamux` offers performance improvements, it's important to recognize that HTTP routers are typically not the main bottleneck in API performance. If you're looking to drastically improve your application's performance, you should primarily focus on optimizing database operations, external network calls, and other potentially time-consuming operations within your application logic. The router's performance becomes more significant in high-throughput scenarios or when dealing with very large numbers of routes.

## Next Steps
The latest released version of `lambdamux` is `v0.0.3`. I'll keep the version below `v1.0.0` until I make sure that everything works as expected and no breaking changes will be introduced. I've tried to add a comprehensive [test suite](https://github.com/D-Andreev/lambdamux/blob/main/internal/radix/radix_test.go) to cover all the different cases.

I will continue to work on my original project and use `lambdamux` as my HTTP router to see if everything is stable and more useful features could be added.

Some new feature candidates could be:
- Middleware support
- Parsing query parameters
- Not found handler

## Conclusion
Implementing an HTTP router from scratch was a fun little project and I learned some new things about how HTTP routers work under the hood. If you're using AWS Lambda functions for your REST APIs, give `lambdamux` a try and let me know what you think.

Code is available [here](https://github.com/D-Andreev/lambdamux).