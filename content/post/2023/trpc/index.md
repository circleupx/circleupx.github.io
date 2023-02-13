---
title: tRPC
tags: [tRPC]
author: "Yunier"
date: "2023-02-05"
description: "A first look into the highly popular tRPC library."
---

In the last few months, my Twitter feed has been dominated by one topic, tRPC. [tRPC](https://trpc.io/) is a library that provides type-safety between your front end and backend, in theory, it allows you to quickly build applications.

In today's post, I would like to explore tRPC, its capabilities and features, and how it could be used in a project. To get started, I will create a new Node.js app using [Express](https://expressjs.com/). If you prefer to use React or Next.js see [the official docs](https://trpc.io/docs/quickstart#next-steps).


## Prerequisites

To get started, I'm going to initiate a new [Node](https://nodejs.org/en/) project using the following command.

```bash
npm init
```

The command will create a package.json file for your application, the command will prompt a series of questions, like the package name, version, description, and so on. See [package.json](https://docs.npmjs.com/cli/v9/configuring-npm/package-json) for more information. Feel free to stick with the default information.

Next, I need to install express, it can be done using the following command.

```bash
npm install express
```

There are additional dependencies required for this project. I'm going to need ts-node, TypeScript, and nodemon to be able to build this app using TypeScript, to install these dependencies run the following commands.

```bash
npm i typescript ts-node nodemon --save-dev
```

Don't forget to include @types/express.

```bash
npm i @types/node @types/express
```

Now I need to set my folder structure given that a client and server will be required for this demo, I am going to create a client and server folder to host the files corresponding to each app. I will also need a tsconfig.json to set tRPC to run in a strict mode as tRPC only support strict mode. The following tsconfig file should do the trick.

```json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "rootDir": "./",
    "outDir": "./build",
    "esModuleInterop": true,
    "strict": true
  }
}
```

I'll run the npm init command again but this time within the client folder and I'll add an index.ts file to the directory. Then I'll switch over to the server directory, and add an index.ts here as well.

Here is the project structure as of now.

```
root
  client
    -index.ts
    -package.json
    -/node_modules
  server
    -index.ts
-package.json
-/node_modules
```

Your server directory doesn't need scripts, a package.json, or any node_modules as they will be available globally from the root directory.

Back in the server directory, open the index.ts file and place the following code there.

```typescript
import express from 'express';
 
const app: express.Application = express();
 
const port: number = 3000;
 
app.get('/', (_req, _res) => {
    _res.send("TypeScript With Express");
});
 
app.listen(port, () => {
    console.log(`TypeScript with Express
         http://localhost:${port}/`);
});
```

The code above is a sample express app, to run it open the package.json located in the root directory and add the following under the script.

```json
{
  "scripts": {
    "server": "nodemon --quiet ./server/index.ts",
  }
}
```

Now to confirm that everything has been installed and configured correctly run the following command.

```bash
npm run server
```

The server is now listening for traffic on port 3000. To confirm, run the following command.

```bash
curl http://localhost:3000
```

You should then see "TypeScript With Express" as the API response. If you did, then good, it means you have everything installed correctly.

## tRPC

Time to configure tRPC.

First, I'm going to need to define a router. What is a router? In tRPC a [router](https://trpc.io/docs/router) acts like an API resource, something like "customers", I come from a .NET background, and a router to me is a [controller](https://www.tutorialsteacher.com/webapi/web-api-controller). In .NET you would have a "CustomerController" class.

I'll add a router.ts file under the server directory. Here is the content of that file.

```ts
import { initTRPC } from '@trpc/server';

export const t = initTRPC.create();
export const appRouter = t.router({});

export type AppRouter = typeof appRouter;
```

As you can see from the code above, I imported tRPC, initiated the library by calling the create() method, and then created a router with no procedures. A [procedure](https://trpc.io/docs/procedures) is an endpoint exposed within our router. For example, in REST you may have a resource called orders and that resource might be exposed through an endpoint like /orders, the procedure is that endpoint. 

A quick note on tRPC, the library should be instantiated once per app having multiple instances will cause issues. So make sure to only call create() once.

Next, I am going to add a [context](https://trpc.io/docs/context), a tRPC context to me feels like an abstraction over the concept of a request middleware, but at the same time it doesn't feel like a request/response middleware. A context can be used to share data or logic with all the procedures, a common example is authentication. Before your procedure is invoked, the request would reach the context which can then validate that the user or client is authorized to make the request.

Since I don't want to deal with authentication in this example, I'm going to keep my context empty as shown in the following code snippet.

```ts
import { inferAsyncReturnType } from '@trpc/server';
import * as trpcExpress from '@trpc/server/adapters/express';

export const createContext = ({
  req, 
  res,
}: trpcExpress.CreateExpressContextOptions) => ({}); // no context

export type Context = inferAsyncReturnType<typeof createContext>;
```

The last thing I need to do now is to modify the index.ts file, I need to replace the code that exists there with one that uses the context and router created above.

Here is the updated index.ts file.

```ts
import { initTRPC } from '@trpc/server';
import express from 'express';
import { Context, createContext } from './server/context';
import * as trpcExpress from '@trpc/server/adapters/express';
import { appRouter } from './server/router';

const t = initTRPC.context<Context>().create();
const app = express();
const port: number = 3000;

app.use(
  '/trpc',
  trpcExpress.createExpressMiddleware({
    router: appRouter,
    createContext,
  }),
);

app.listen(port);
```

In the code above we use a helper function to inject tRPC into the Express middleware pipelines. The helper function takes the router and context created above. Then the API listens for traffic in port 3000. The time has come to expose some procedures. I'm going to create a procedure that supports getting a list of cars and another procedure that supports adding a new car.

Back in the router.ts file, I am going to add two new endpoints "getCars" and "createCar". 

```ts
import { initTRPC } from '@trpc/server';

export const t = initTRPC.create();
export const appRouter = t.router({
  getCars: t.procedure
    .query((req) => {
      return [{
        id: 1, name: 'Toyota',
      }, {
        id: 2, name: 'Honda',
      }];
    }),
  createCar: t.procedure
    .mutation(async (req) => {
      return {
        name: 'Nissan'
      }
    }),
});

export type AppRouter = typeof appRouter;
```

Let's examine the new endpoints, the first procedure exposed is "getCars", it is then mapped to a query function. One thing to note about procedures in tRPC is that they are case-sensitive, meaning the following request will work.

```bash
curl http://api.example.com/trpc/getCars
```

but the following request will fail.

```bash
curl http://api.example.com/trpc/getcars
```

It will return the following JSON document.

```json 
{
  "error": {
    "message": "No \"query\"-procedure on path \"getcars\"",
    "code": -32004,
    "data": {
      "code": "NOT_FOUND",
      "httpStatus": 404,
      "stack": "Full stack trace ommited for brevity",
      "path": "getcars"
    }
  }
}
```

The document doesn't appear to be a [Problem Details](/content/post/2021/problem-details-for-HTTP-APIs/) document, I would love it if as developers we stopped reinventing the wheel and used known standards.

The next thing to note is that tRPC uses [query, mutation and subcriptions](https://spec.graphql.org/October2021/#sec-Executing-Operations) like [GraphQL](https://graphql.org/). In fact, by [their own admittion](https://trpc.io/docs/further-reading#relationship-to-graphql), tRPC borrows a few ideas from GraphQL. Since our "getCars" is meant to retrieve data I mapped it to a query function. The query function itself doesn't do much in my example, it simply returns an in-memory collection of cars. In a real-world application, the in-memory collection would be replaced with your data abstraction logic, i.e. repositories.

The second procedure exposes a "createCar" endpoint, this endpoint accepts an HTTP POST request, just like the "getCars" procedure, and it turns an in-memory object. Something else to note here is that like GraphQL, tRPC does not use the HTTP PUT, PATCH, or DELETE, all those operations in tRPC would be done via an HTTP POST.

Our API is now ready to serve traffic, let's create our client app, back in the client folder I'll update the index.ts file with the following code.

```ts
import { createTRPCProxyClient, httpBatchLink } from '@trpc/client';
import { AppRouter } from '../server/router';
import fetch from 'node-fetch';

// polyfill
const globalAny = global as any;
globalAny.AbortController = AbortController;
globalAny.fetch = fetch as any;

async function main() {
    const trpc = createTRPCProxyClient<AppRouter>({
        links: [
            httpBatchLink({
                url: 'http://localhost:3000/trpc',
            }),
        ],
    });

    const cars = await trpc.getCars.query();
    console.log(cars);
}

main();
```

To run the API server and client at the same time I will install 

```bash
npm i concurrently --save-dev
```

With the package concurrently now installed I can now run the API server and client at the same time, just need to update the root package.json file.

```json
{
  "name": "fun-trpc",
  "version": "1.0.0",
  "description": "testing trpc",
  "main": "index.js",
  "scripts": {
    "server": "nodemon --quiet ./server/index.ts",
    "client": "npm start --prefix client",
    "start": "concurrently \"npm run server\" \"npm run client\""
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@trpc/server": "^10.10.0",
    "@types/express": "^4.17.17",
    "@types/node": "^18.11.19",
    "express": "^4.18.2",
    "zod": "^3.20.2"
  },
  "devDependencies": {
    "concurrently": "^7.6.0",
    "nodemon": "^2.0.20",
    "ts-node": "^10.9.1",
    "typescript": "^4.9.5"
  }
}
```

I can now run the following command to confirm the API and client are working.

```bash
$ npm run start

> fun-trpc@1.0.0 start
> concurrently "npm run server" "npm run client"

[0]
[0] > fun-trpc@1.0.0 server
[0] > nodemon --quiet ./server/index.ts
[0]
[1]
[1] > fun-trpc@1.0.0 client
[1] > npm start --prefix client
[1]
[1]
[1] > trpc-client@1.0.0 start
[1] > nodemon client.ts
[1]
[1] [nodemon] 2.0.20
[1] [nodemon] to restart at any time, enter `rs`
[1] [nodemon] watching path(s): *.*
[1] [nodemon] watching extensions: ts,json
[1] [nodemon] starting `ts-node client.ts`
[1] [ { id: 1, name: 'Toyota' }, { id: 2, name: 'Honda' } ]
[1] [nodemon] clean exit - waiting for changes before the restart
```

As you can see from the console output above my client was able to use connect to the API server and pull data. Since I am working with a collection the console output had two cars, but let's say that instead of I wanted the first car, and from the first car I wanted the name, the client's main function could be updated as follows.

```ts
async function main() {
    const trpc = createTRPCProxyClient<AppRouter>({
        links: [
            httpBatchLink({
                url: 'http://localhost:3000/trpc',
            }),
        ],
    });

    const cars = await trpc.getCars.query();
    console.log(cars[0].name); // type-safety here. I'm able to infer my API types from the client thanks to tRCP. 
}
```

This is where tRPC shines, full type-safety in the front end using types defines in the back end. Now, this isn't something new, type-safety in front-end code has been done before. For example, if you are not using tRPC you may build a copy of the API type in your front-end code then right after you call the API you would transform the API response into its type equivalent, for example, imagine you are using Axios to call an API as shown below.

```ts
import axios from 'axios';

async function getCars() {
  try {
    const { data, status } = await axios.get('https://api.example.com/cars');
    return data;
  } catch (error) {
      console.log('error message: ', error.message);
      return error.message;
    }
  }
}

getUsers();
```

The code above doesn't offer any type-safety, if have been programming long enough in TypeScript then you know that type-safety can be easily added by modifying the code as shown below.

```ts
import axios from 'axios';

type Car = {
  id: number;
  name: string;
};

async function getCars() {
  try {
    const { data, status } = await axios.get<Cars[]>('https://api.example.com/cars');

    return data;
  } catch (error) {
      return error.message;
  }
}

getCars();
```

Now the data return from getCars is type-safe. Again nothing new here, pretty standard. I do think that while tRPC approach offers a better developer experience I am not too sure how many people can take advantage of tRPC. For once, your API code and client code must live together, additionally, newer frameworks like Remix offer the same type-safety out of the box without having to use tRPC.

## Conclusion

Still, I enjoyed doing this demo, it helped me understand why everyone has been hyping tRPC. I think the library can be beneficial to many out there, the only time I would not recommend using tRPC is if you are starting a new project on a framework like Remix or for obvious reasons, if your front-end and backend are written in different languages, but who knows, tRPC is very young, in the future, it may expand to other languages.

I do encourage you to learn more by visiting the [official website](https://trpc.io/), the example written here just scratches the surface. 