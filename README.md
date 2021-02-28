# @selfage/service_client

## Install

`npm install @selfage/service_client`

## Overview

Written in TypeScript and compiled to ES6 with inline source map & source. See [@selfage/tsconfig ](https://www.npmjs.com/package/@selfage/tsconfig) for full compiler options. Provides a type-safe client to call services described by `@selfage/service_descriptor` and handled by `@selfage/service_handler`. The service here only refers to one simple kind of client-server interaction: Sending a HTTP POST request in JSON as request body and receiving a response in JSON.

## Constructor

`createWithLocalStorage()` injects a `SessionStorage` implementation using `window.localStorage` as well as `window.fetch`. Because of that, client created this way cannot be used in Nodejs environment.

```TypeScript
import { ServiceClient } from '@selfage/service_client';
import { SessionStorage } from '@selfage/service_client/session_storage';

let client = ServiceClient.createWithLocalStorage();
```

However, both of them can be replaced. In fact, our unit tests replaced `SessionStorage` implementation with a mock implementation and `window.fetch` with `node-fetch`. However, `window.fetch` and `node-fetch` have different function signatures in TypeScript and we have to cast `node-fetch` as `any`. 

```TypeScript
import { ServiceClient } from '@selfage/service_client';
import { SessionStorage } from '@selfage/service_client/session_storage';
import fetch = require('node-fetch');

let client = new ServiceClient(new class implements SessionStorage {/* ... */}, fetch as any);
```

## SessionStorage

To not confuse with `window.sessionStorage`, `SessionStorage` in this lib has nothing to do with that, and as its name suggests, it's purely a TypeScript interface to store a session string. See its [source](https://github.com/selfage/service_client/blob/9ffb717f194f98212d60d3034bd12bd0bafeddf8/session_storage.ts). If used on backend servers, you can also provide an implementation using in-memory maps, disks, or database.

As stated above, we provide a `LocalSessionStorage` implementation using `window.localStorage` as opposed to using cookie. Therefore the session string will not be sent with every request.

Read further for how it's used during authentication.

## Fetch services without authentication

`fetchUnauthed()` takes a request object as well as an `UnauthedServiceDescriptor` and returns a `Promise` of a response object. 

```TypeScript
import { GET_COMMENTS } from './get_comments';

async function run() {
  // Suppose we created a `ServiceClient`.
  let response = await client.fetchUnauthed({ videoId: "xBivT1" }, GET_COMMENTS);
}
```

`get_comments.ts`([source](https://github.com/selfage/service_client/blob/9ffb717f194f98212d60d3034bd12bd0bafeddf8/test_data/get_comments.ts)) is typically generated by installing `@selfage/cli` and runing `selfage gen get_comments`, which requires an input file `get_comments.json`([source](https://github.com/selfage/service_client/blob/9ffb717f194f98212d60d3034bd12bd0bafeddf8/test_data/get_comments.json)), specifying the url endpoint/path as `/get_comments`.

See `@selfage/service_descriptor` and `@selfage/message` for more explanation of the JSON file. Typically, `get_comments.json` and `get_comments.ts` will be shared between client-side and server-side code.

## Fetch services requiring authentication

`fetchAuthed()` takes a request object as well as an `AuthedServiceDescriptor` and returns a `Promise` of a response object.

```TypeScript
import { GET_HISTORY } from './get_history';

async function run() {
  // Suppose we created a `ServiceClient`.
  let response = await client.fetchAuthed({ page: 1 }, GET_HISTORY);
}
```

As also documented in `@selfage/service_descriptor`, an authed service requiring its request to contain `signedSession` field. See `get_history.ts`([soure](https://github.com/selfage/service_client/blob/9ffb717f194f98212d60d3034bd12bd0bafeddf8/test_data/get_history.ts)) and `get_history.json`([source](https://github.com/selfage/service_client/blob/9ffb717f194f98212d60d3034bd12bd0bafeddf8/test_data/get_history.ts)) as the example.

You don't need to explicitly set `signedSession` field. `ServiceClient` will set it with the session string by calling `read()` on `SessionStorage`.

## Catch errors

You can handle all kinds of errors for each service call using try-catch statement, including network connection errors, which are thrown by `fetch` API, and server responded errors which are thrown by `ServiceClient`.

```TypeScript
async function run() {
  try {
    // Suppose we created a `ServiceClient`.
    let response = await client.fetchUnauthed({ videoId: "xBivT1" }, GET_COMMENTS);
  } catch (e) {
    // Log error or display it to users.
  }
}
```

## Catch HttpError

`ServiceClient` will construct an `HttpError` whenever server finishes response without ok status, typically with 4xx or 5xx error code. See `@selfage/http_error` for more explanation of `HttpError`.

In addition to the try-catch statement above, you can also add a listener to it, which can serve as a global handler of `HttpError` especially when `ServiceClient` is instantiated as a global singleton.

```TypeScript
// Suppose we created a `ServiceClient`.
client.onHttpError = (httpError /* HttpError */) =>  {
  // E.g., redirect to home page.
};
```

## Catch unauthenticated error

When server finishes response with 401 error code, i.e., Unauthorized, `ServiceClient` treats it as an unauthenticated error by calling `clear()` on `SessionStorage`.

In addition to the try-catch statement above, you can also add a listener to it, which can serve as a global handler of `HttpError` especially when `ServiceClient` is instantiated as a global singleton.

```TypeScript
// Suppose we created a `ServiceClient`.
client.onUnauthenticated = (/* No args */) =>  {
  // E.g., redirect to home page.
};
```

It's often confused about unauthenticated and unauthorized error, especially because standard Http error codes did confuse them. Unauthenticated means the user cannot be identified, e.g., because of wrong password, whereas unauthorized means the user might be identified but doesn't have enough privilege/permission to access the web page/call the service, e.g., the user can read documents but cannot edit them.

The closest error code to represent unaunthenticated error is 401, although it's named as "Unauthorized". Its [spec](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/401) also says that "authentication is possible" for 401, whereas "re-authenticating will make no difference" for 403. Please keep that in mind when handling unauthenticatd/unauthorized errors on client-side as well as when returning them on server-side. 

