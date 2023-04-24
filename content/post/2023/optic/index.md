---
title: "Tracking API changes with Optic"
tags: [optic]
author: "Yunier"
date: "2023-04-17"
description: "Preventing API breaking changes using Optic."
---

Over the last few months, I have been brushing up on [API testing](https://www.postman.com/api-platform/api-testing/), specifically around contract testing.

According to Postman, an API contract is a human- and machine-readable representation of an API's intended functionality. It establishes a single source of truth for what each request and response should look like—and forms the basis of service-level agreements (SLAs) between producers and consumers. API contract testing helps ensure that new releases don't violate the contract by checking the content and format of requests and responses.

If you begin to walk down the road of API contrast testing, one question that naturally comes up is, what tools can I use to identify breaking changes? After all, reviewing a large YAML file on a pull request is not necessarily the best way for humans to spot breaking changes to API contracts.

While researching API Contract testing I came across two tools that help identify breaking changes, [Akita](https://www.akitasoftware.com/) and [Optic](https://www.useoptic.com/). In today's post, I would like to explore Optic and demonstrate some of its capabilities and how it can be used to identify breaking changes.

According to their [documentation](https://www.useoptic.com/docs/core-concepts), Optic is a version of control tool, just like Git. In fact, it relies on Git in order to find breaking changes, it invokes a [git show](https://git-scm.com/docs/git-show) command to be able to see a diff between OpenAPI spec files across branches, which means you can use Optic in your existing pull request and since Optic is an NPM package you can import it into any CI provider.

To get started, installed the Optic CLI using NPM.

```sh
npm -g install @useoptic/optic
```

As of April 2023, the command above will install version [0.42.13](https://www.npmjs.com/package/@useoptic/optic/v/0.42.13). With Optic now installed you can use the diff command to compare two OpenAPI specs. Below you will find an OpenAPI specification file that I have created for this demo.

```YAML
openapi: 3.1.0
x-stoplight:
  id: tdmzx0hqhylls
info:
  title: Users
  version: '1.0'
servers:
  - url: 'http://localhost:3000'
paths:
  '/users/{userId}':
    parameters:
      - schema:
          type: integer
        name: userId
        in: path
        required: true
        description: Id of an existing user.
    get:
      summary: Get User Info by User ID
      tags: []
      responses:
        '200':
          description: User Found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
              examples:
                Get User Alice Smith:
                  value:
                    id: 142
                    firstName: Alice
                    lastName: Smith
                    email: alice.smith@gmail.com
                    dateOfBirth: '1997-10-31'
                    emailVerified: true
                    signUpDate: '2019-08-24'
        '404':
          description: User Not Found
      operationId: get-users-userId
      description: Retrieve the information of the user with the matching user ID.
components:
  schemas:
    User:
      title: User
      type: object
      description: ''
      examples:
        - id: 142
          firstName: Alice
          lastName: Smith
          email: alice.smith@gmail.com
          dateOfBirth: '1997-10-31'
          emailVerified: true
          signUpDate: '2019-08-24'
      properties:
        id:
          type: integer
          description: Unique identifier for the given user.
        firstName:
          type: string
        lastName:
          type: string
        email:
          type: string
          format: email
        dateOfBirth:
          type: string
          format: date
          example: '1997-10-31'
        emailVerified:
          type: boolean
          description: Set to true if the user's email has been verified.
        createDate:
          type: string
          format: date
          description: The date that the user was created.
      required:
        - id
        - firstName
        - lastName
        - email
        - emailVerified

```

Might be hard to see from the YAML, but basically, the API defined in the YAML is an API with a /users endpoint, an HTTP client can send a GET request to retrieve a specific user. The users resource exposes the following model.

```JSON
{
  "firstName": "Bob",
  "lastName": "Fellow",
  "email": "bob.fellow@gmail.com",
  "dateOfBirth": "1996-08-24"
}
```

Let's assume I want to change the API, say I want to rename the field dateOfBirth to dob, breaking the existing contract exposed by the API, I will modify the contract and save my changes in a change.yaml file and the existing contract will be saved to a main.yaml file. To have Optic identify breaking changes on the contract I can use the following command.

```sh
optic diff main.yaml change.yaml
```

The command above results in the following output. 

```sh
GET /users/{userId}:
  - response 200:
    - body application/json:
      - /schema/properties/dob added
      - /schema/properties/dateOfBirth removed
```

Don't know if you can tell from the output, but Optic has successfully identified the change, appending a --web to the command we ran will output the results in a web browser for better visual aid as seen in the screenshot below.

![optic](/post/2023/optic/optic-1.png)

You can also run Optic across git branches using the diff command, you simply would change to the branch containing the API changes and then run the command. 

```sh
optic diff openapi.yml --base main --check --web
```

Where --base is the name of the main branch, in this case, the main branch is called main. The --check parameter tells Optic to run the default breaking changes. See optic also offers the ability to apply rules, allowing you to create API standards, for example, imagine that you wanted all properties to be written using snake casing. In optic, you would define the properties casing as snake casing then you would run the rules against all pull requests. Let's see how that would work, you will find a custom rule definition, I want all my properties to follow snake casing because it is easier to read than camel casing.

```YAML
ruleset:
  # Enforce consistent cases in your API
  - naming:
      properties: snake_case
```

Now that I have my rule I can apply it to the previous command.

```sh
optic diff main.yaml change.yaml --check --ruleset ./ruleset.yaml
```

```sh
   FAIL  GET /users/{userId}
    requirement rule [error]: response property naming check
      x firstName is not snake_case
      at paths > /users/{userId} > get > responses > 200 > content > application/json > schema > properties > firstName

    requirement rule [error]: response property naming check
      x lastName is not snake_case
      at paths > /users/{userId} > get > responses > 200 > content > application/json > schema > properties > lastName

    requirement rule [error]: response property naming check
      x emailVerified is not snake_case
      at paths > /users/{userId} > get > responses > 200 > content > application/json > schema > properties > emailVerified

    requirement rule [error]: response property naming check
      x createDate is not snake_case
      at paths > /users/{userId} > get > responses > 200 > content > application/json > schema > properties > createDate

3 operations changed
0 passed
29 errors
```

Optic has identified all properties that do not follow the naming convention defined by my rule. This is great, I can now enforce naming conventions across all resources on my users API.

On an upcoming blost I will explore [Akita](https://www.akitasoftware.com/), the second tools I found for API contract testing, till next time.

Cheerio. 
