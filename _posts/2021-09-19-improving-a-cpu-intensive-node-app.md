---
title: Improving a CPU-intensive Node.js app
tags: [Node.js, Microservices]
published: true
---

Recently I was asked to review a Web API written in Node.js. The API exposes an authentication endpoint, this authentication endpoint must be highly available, responsive, and it cannot become a bottleneck, otherwise, the user experience is severely impacted. Unfortunately, the endpoint had become a bottleneck and was impacting the overall performance of the application. Upon further review, it was determined that the problem was coming from a hashing function that takes the user's password, hashes it, and compares the result with the stored hashed password from the database. Here is the code without the implementation details.

```javascript
var app = module.exports = express();

hash(function (err, pass, salt, hash) {
    // implementations omitted for brevity.
});

function authenticate(name, pass, fn) {
  hash({ password: pass, salt: user.salt }, function (err, pass, salt, hash) {
    // implementations omitted for brevity.
  });
}

app.post('/login', function(req, res){
  authenticate(req.body.username, req.body.password, function(err, user){
    // implementations omitted for brevity.
  });
});
```

The hash function used a CPU-intensive algorithm and Node.js is notorious for being bad at heavy CPU operations, it will block the thread. This is because [Node.js is single threaded](https://www.youtube.com/watch?v=ztspvPYybIY). To ensure the authentication Web API is never blocked by a CPU intensive operation the hashing function was off-loaded to a [microservice](https://microservices.io/). This microservice is dedicated to just computing the output of the hash function. The benefit of this approach is that now the authentication Web API will no longer be blocked by CPU work, additionally, the hashing microservice can now be scaled up or down based on load. As it turns out, this is a well-known pattern in Node.js. See [this](https://medium.com/geekculture/dealing-with-node-js-high-cpu-in-production-71c432d8bece) post by [Alex Vasilyev](https://luckylibora.medium.com/) to learn more.

{: .notice--warning}
Authentication Web API high-level overview of the new architecture.

![Node.js hashing microservices architecture](/assets/img/NodeMicroservices.png)