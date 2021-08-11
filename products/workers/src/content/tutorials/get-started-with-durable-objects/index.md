## Intro

Durable Objects are *addressable* Cloudflare Workers functions, with underlying state, that can be interacted with from other Workers. This means that regardless of where your Workers functions are running, or where your users are located, given a correctly-setup Durable Objects function, you can be sure that the _same_ Durable Object (and the same underlying state) can be referenced and used consistently.

Durable Objects are a transformational approach to consistent, distributed state. By being deployed on Cloudflare's edge, Durable Objects have the same performance characteristics that the whole Workers ecosystem shares - low-latency, with zero cold starts, and a powerful JavaScript-based developer experience that you can get up and running with in just a few minutes.

In this guide, you'll build a powerful Workers function that uses all of the capabilities that Durable Objects provide. Along the way, you'll learn about how Durable Objects work, avoid some common pitfalls, and use much of the tooling that we've built to help developers effectively deploy Durable Objects into production.

``<GetStartedComponent />``

## Build a Workers function

In this guide, you'll use the [`durable-objects-typescript-rollup-esm` template](https://github.com/cloudflare/durable-objects-typescript-rollup-esm) available on GitHub to get started building your first Durable Object. To begin, generate a new Workers project using Wrangler, passing in the above repository URL to the `generate` function:

```sh
$ wrangler generate my-first-durable-object https://github.com/cloudflare/durable-objects-typescript-rollup-esm
```

Your new project will contain two important files in the `src` directory that we'll expand on this guide:

1. `index.ts` is the entry-point to your Workers function, written in [Module Worker](TODO) format.
2. `counter.ts` is an example Durable Object class (`CounterTs`), which you'll modify to provide more complex behavior to your Workers function.

Deploying Durable Objects is mostly similar to publishing standard Workers functions with Wrangler, with the exception of your first time publishing. To tell the Workers platform that you're deploying a new Durable Object, you must provide the `--new-class` flag when publishing for the first time, with a value matching the class name your Durable Object uses:

```sh
$ wrangler publish --new-class CounterTs
```

## Establishing a Durable Objects binding

Durable Objects can be interfaced with from inside of a Workers function. As a quick review, a [Module Worker](TODO) will make use of its defined `fetch` function, which will be called when a user makes an HTTP request to your Workers function:

```js
---
filename: index.ts
---
export default {
  async fetch(request: Request) {
    return new Response("Hello, world!")
  }
}
```

`fetch` has `request` available as the first argument to the function. Once you begin working with Durable Objects, as well as Workers KV, you'll also begin making use of `env`, the second argument available to the `fetch` function. The `env` argument will contain any bindings provided to the Workers function. Bindings are special constants that represent any attached Durable Objects or Workers KV namespaces:

```js
---
filename: index.ts
lines: [2]
---
export default {
  async fetch(request: Request, env: Env) {
    return new Response("Hello, world!")
  }
}
```

To better understand where this `env.COUNTER` comes from, inspect `wrangler.toml`. The `durable_objects` definition at the end of `wrangler.toml` connects the `COUNTER` binding to the `CounterTs` Durable Object class:

```toml
---
filename: wrangler.toml
lines: [16, 17]
---
name = "worker"
# type = "javascript" is required to use the `[build]` section
type = "javascript"
account_id = ""
workers_dev = true
route = ""
zone_id = ""

[build]
command = "npm install && npm test && npm run build"
[build.upload]
# The "modules" upload format is required for all projects that export a Durable Objects class
format = "modules"
main = "./index.mjs"

[durable_objects]
bindings = [{name = "COUNTER", class_name = "CounterTs"}]
```

The `CounterTs` class is the definition of the Durable Objects class that you will use in this tutorial. In its simplest form, it is a JavaScript class with a `constructor` function, which exposes the underlying `state` storage, an `env` object, and the `fetch` function - more on these later:

```js
---
filename: counter.ts
---
export class CounterTs {
  state: DurableObjectState
  
  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
  }

  async fetch(request: Request) {
    return new Response("Hello, world!")
  }
}
```

After configuring the binding to correspond to this `CounterTs` class, the `env` argument in your Worker's `fetch` function will expose a binding for interacting with your Durable Object class that matches the `name` field you specified in `wrangler.toml`:

```js
---
filename: index.ts
lines: [3,4,5]
---
export default {
  async fetch(request: Request, env: Env) {
    // env.COUNTER represents the Durable Object binding
    const durableObjectId = env.COUNTER.newUniqueId()
	return new Response(durableObjectId)
  }
}
```

## Creating an instance of a Durable Object

Durable Object namespace bindings have two jobs: generating object IDs, and connecting to objects. To create an instance of a Durable Object, you first need to create an object ID. Using the same ID in subsequent requests will allow you to reference the same Durable Object: this is a unique property of Durable Objects, namely, that they are _addressable_.

There are two ways of generating IDs: you can create a named ID, based on a known value, or by allowing the Durable Object binding to generate a unique ID for you.

Named IDs are generally easier to reason about. For instance, given a string "Cloudflare", you can generate a new ID based on that string, by calling `.idFromName`:

```js
---
filename: index.ts
lines: [3,4]
---
export default {
  async fetch(request: Request, env: Env) {
    const namedId = env.COUNTER.idFromName("Cloudflare")
	return new Response(namedId)
  }
}
```

The advantage to generating a Durable Object ID based on a name is that it is _repeatable_. Given the string "Cloudflare" and your binding, the resulting ID can be repeatedly generated in future requests, meaning that you can access the same Durable Object.

Unique IDs are created by using the `.newUniqueID` function:

```js
---
filename: index.ts
lines: [3,4]
---
export default {
  async fetch(request: Request, env: Env) {
    const uniqueId = env.COUNTER.newUniqueID()
	return new Response(uniqueId)
  }
}
```

Although named IDs may seem easier in practice, there are many characteristics about unique IDs that make them a more desirable choice when working with Durable Objects.

The first of these characteristics is performance. When you construct a new unique ID, the system knows that the same ID will not be generated by another Worker running on the other side of the world at the same time. Therefore, the object can be instantiated nearby without waiting for any round-the-world synchronization.

The second characteristic is when you have jurisdictional requirements. As outlined [in this blog post](https://blog.cloudflare.com/supporting-jurisdictional-restrictions-for-durable-objects/), the `.newUniqueId` method allows developers to segment Durable Objects in particular jurisdictions, in order to conform with data privacy laws like the [GDPR](https://gdpr-info.eu). The jurisdictional restrictions feature of Durable Objects is only available when using unique IDs, and isn't currently supported with named IDs. Providing a jurisdiction can be done by including an object as the argument to `.newUniqueId`, with the `jurisdiction` key:

```js
---
filename: index.ts
lines: [3]
---
export default {
  async fetch(request: Request, env: Env) {
    const uniqueId = env.COUNTER.newUniqueID({ jurisdiction: 'eu' })
	return new Response(uniqueId)
  }
}
```

With these characteristics in mind, we generally recommend that developers use unique IDs whenever possible. Note that this requires that you store the unique ID itself after the Durable Object has been instantiated. Usually, you will do this in your client, such as a browser or mobile application.

Now that you have generated an ID for your Durable Object, you can create a Durable Objects stub. The stub is a local client that provides access to a remote Durable Object. This can be done using the `.get` method on the Durable Object binding, passing the newly created ID:

```js
---
filename: index.ts
lines: [3]
---
export default {
  async fetch(request: Request, env: Env) {
    const id = env.COUNTER.newUniqueID()
	let durableObjectStub = env.COUNTER.get(id)
	return new Response(id)
  }
}
```

The stub returned by the `.get` method has a single function available for use, `.fetch`. This method works identically to the global [`fetch`](https://developers.cloudflare.com/workers/runtime-apis/fetch) function available in Cloudflare Workers. It also corresponds to the `fetch` method that we defined in our Durable Object class `CounterTs`, which has been reprinted below:

```js
---
filename: counter.ts
lines: [8,9,10]
---
export class CounterTs {
  state: DurableObjectState
  
  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
  }

  async fetch(request: Request) {
    return new Response("Hello, world!")
  }
}
```

Now that the entire flow of generating a Durable Object ID, creating a stub, and making a `fetch` request to that stub has been outlined, it should be clear that the `fetch` function inside of the `CounterTs` class itself represents _what_ will be called when you call `stub.get(id)`. Like the root Module Worker's `fetch` function, a Durable Object's `fetch` function _also_ returns a `Response`, meaning that you can parse it inside of the Module Worker, or return it directly from the function:

```js
---
filename: index.ts
lines: [5,6]
---
export default {
  async fetch(request: Request, env: Env) {
    const id = env.COUNTER.newUniqueID()
	let durableObjectStub = env.COUNTER.get(id)
	let durableObjectResp = await durableObjectStub.fetch(request)
	return durableObjectResp
  }
}
```

## How a Durable Object's lifecycle works

> - Lack of explicit creation/deletion of DO
>   - Solution: explain the lifecycle of DO and how Objects are automatically deleted if no storage is used
>     - We should also explain billing here

## Working with WebSockets and Durable Objects

```
notes: 

Durable Objects can be used to manage a WebSocket, by allowing multiple clients to enter the same DO instance. The WebSocketPair class allows the DO to hold onto a "server" WS, and pass back a client WS as the response. This server WS can send messages down to multiple clients, and receive new information up from those client WSs.

Prior art template: https://github.com/cloudflare/websocket-template
```

## Understanding the Durable Object storage layer

```
notes: 

Add onto the existing WebSocket layer by persisting information in storage.

Durable Objects have a storage layer that can be used to persist data when the DO isn't active, or when multiple users/clients are interacting with a DO at the same time.

I think that the intricacies of that storage layer - where there used to be race conditions, and things to handle, that's been mostly fixed, via this blog post (but double check!): https://blog.cloudflare.com/durable-objects-easy-fast-correct-choose-three/ 

API: https://developers.cloudflare.com/workers/runtime-apis/durable-objects#transactional-storage-api
```

## Patterns for communicating with other Durable Objects

```
notes: Durable Objects have an `env` binding which allows them to communicate with other Durable Objects via the same patterns established in the root module Worker. Need a compelling usecase here and maybe a better explanation as to "why" besides "because you can"
```
