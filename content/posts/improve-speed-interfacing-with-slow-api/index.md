---
author: "Christos Panagiotakopoulos"
title: "A trick to improve speed when you are interfacing with a slow API"
date: "2020-01-19"
description: "'memoized-node-fetch' an npm package that acts as a wrapper around node-fetch"
tags: ["package", "opensource", "javascript", "typescript"]
categories: ["programming"]

cover:
    image: "memoized-node-fetch.webp"
    relative: true # To use relative path for cover image, used in hugo Page-bundles
---

> TLDR; I created a small npm package that acts as a wrapper around node-fetch, and returns the same promise for the same request, until it resolves. You can visit the repo of this package [here](https://github.com/chrispanag/memoized-node-fetch). Below, I explain my motivation, and how I tackled the issue.

**So here's the scenario:**

You have a system that interfaces with a really slow third-party API. User Bob, needs some data, so your system performs a request to the third-party API, and waits for a response. In the meantime, user Alice needs the same date and the system performs the same request to the API on behalf of her. Both users are now waiting for two requests that the only difference they have, is the execution time. 

If a request to this API has an average response time of 1 second, both users will wait 1 second. Also, you would need to occupy resources in your system and the third-party API for more than 1 second, and for 2 seconds at most!

## The solution

What if you could have both users, Bob and Alice, wait for the same request? Then, although Bob will still wait for the request for 1 second, Alice will use Bob's request, and wait less time for the response. 

To achieve that, we'll need a **promise-cache subsystem**. This subsystem will consist of a data structure to store our requests' promises and of a way to retrieve them/delete them when they are not needed.

### The data structure

We need a data structure to store our promises inside. This data structure needs to be able to store and retrieve a new promise in one operation (O(1)). So, the best choice would be a key/value store. Javascript, offers two such structures, the [basic object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object) and the [Map()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) instance. The [most preferrable data structure for our use-case among the two is the Map()](https://medium.com/front-end-weekly/es6-map-vs-object-what-and-when-b80621932373).

So, let's create it:

```typescript
const promiseCache: Map<string, Promise<Response>> = new Map();
```

### The retrieval/storage

Now, let's create a function that wraps around the request function and retrieves the same promise for the same request, if it exists. If it doesn't, it performs a new request and stores it in the cache.

```typescript
function memoizedRequest(url: string) {
    const key = url;
    if (promiseCache.has(key)) {
        return promiseCache.get(key);
    }

    const promise = request(url);
    promiseCache.set(key, promise);

    return promise;
}
```

With this, we have achieved the basic function of our promise-cache subsystem. When our system performs a request using the `memoizedRequest` function, and the request has already happened, it returns the same promise. 

But, we haven't yet implemented the mechanism for the deletion of the promise from the cache when the promise resolves (when the request returns results)

### The deletion - cache invalidation

For this, we'll create a function that awaits for the promise to resolve and then delete the promise from the cache.

```typescript
async function promiseInvalidator(key: string, promise: Promise<any>) {
    await promise;
    promiseCache.delete(key);
    
    return promise;
}
```

And then we'll modify our memoizedRequest function to include this invalidation function:

```typescript
function memoizedRequest(url: string) {
    const key = url;
    if (promiseCache.has(key)) {
        return promiseCache.get(key);
    }

    const promise = promiseInvalidator(key, request(url));
    promiseCache.set(key, promise);

    return promise;
}
```

### But what happens with more complicated requests?

Not all requests can be differentiated by just the url they are performed on. There are many other parameters that make a request different (eg: headers, body etc). 

For that, we'll need to refine our promise-cache's key and add an options object on our function:

```typescript
function memoizedRequest(url: string, options: RequestOptions) {
    const key = url + JSON.stringify(options);
    if (promiseCache.has(key)) {
        return promiseCache.get(key);
    }

    const promise = promiseInvalidator(key, request(url));
    promiseCache.set(key, promise);

    return promise;
}
```

Now, **only the requests that use exactly the same options** will return the same promise until they resolve.

With this, we implemented all the basic functionality of our package. But we haven't taken into account the possibility of a request failure. Let's add this on our code, by making the `promiseInvalidator` function to always remove the promise from the cache either when it resolves, or when it rejects.

```typescript
async function promiseInvalidator(key: string, promise: Promise<any>) {
    try {
        await promise;
    } finally {
        promiseCache.delete(key);
    }
    
    return promise;
}
```

### More improvements

This implementation has a small drawback, that can prove serious on a production system. All the requests' data, are stored within the key of our data store, highly increasing the memory requirements of our application, especially when our requests contain a lot of data. The solution to this is to use a [hash function](https://en.wikipedia.org/wiki/Hash_function) on our key, to assign a unique value to each different request, without needing to include all the actual of the request.

```typescript
const key = hasher(url + JSON.stringify(options));
```
### Caveats

This solution, isn't applicable to any situation. To use this solution, you need to ensure that the API you are interfacing with, **is not providing different responses for two different requests** in the amount of time it will take for those requests to resolve.

## The package

If you don't want to code this for yourself, I created a simple [**npm package**](https://www.npmjs.com/package/memoized-node-fetch) that does all of the above, as a wrapper to [node-fetch](https://www.npmjs.com/package/node-fetch) (or any other fetch-like function you choose). 

```typescript
import memoizedNodeFetch from 'memoized-node-fetch';

const fetch = memoizedNodeFetch();

(async () => {
    const fetch1 = fetch('https://jsonplaceholder.typicode.com/todos/1');
    const fetch2 = fetch('https://jsonplaceholder.typicode.com/todos/1');

    // This should return true because both requests return the same promise.
    console.log(fetch1 === fetch2);

    const res1 = await fetch1;
    const res2 = await fetch2;

    console.log(await res1.json());
    console.log(await res2.json());
})();
```

You can see all of the above work, on its Github repository here:

https://github.com/chrispanag/memoized-node-fetch

PS. 1: Although this can be used in the front-end, I can't find a very useful use-case for it, especially when you have other packages such as react-query/swr, that although they perform a different function than the above, can sometimes remove the need for it.

PS. 2: Special thanks to the other two contributors of this repository ([**ferrybig**](https://github.com/ferrybig) and [**Bonjur**](https://github.com/Bonjur) for their invaluable input and suggestions!