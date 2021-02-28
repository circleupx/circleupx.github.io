---
title: Chinook API Project Now Hosted on Heroku
layout: post
tags: [Chinook, Heroku]
readtime: true
---

My [Chinook JSON:API project](https://github.com/circleupx/Chinook) is now in a good enough state that I feel comfortable hosting it on a live server. Here is the base url, [https://chinook-jsonapi.herokuapp.com/](https://chinook-jsonapi.herokuapp.com/), I highly recommend using some kind of JSON viewer if you want to interact with the API. If you are on a Chromium base browser then I recommend using [JSON Viewer](https://chrome.google.com/webstore/detail/json-viewer/gbmdgpbipfallnflgajpaliibnhdgobh). 

Remember, the API does not support filtering, pagination, sorting or include resolvers and it only supports READ operations. I'm hoping to add filtering soon but I first want to dedicate a blog post or two on building dynamic LINQ queries using expression trees. That should be fun!