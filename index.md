---
layout: default
title: "Amorphic"
---

### What is Amorphic?

To date web-based applications have followed one of two paradigms.  Either applications posted data to the server 
and in response a new page was returned to the browser or a single page makes xhr calls to the server which will
return data.  Amorphic is a refinement of the second paradigm for object-oriented applications.
Each instance of an object lives both in the browser and on the server.  Each method is configured to have
it's implementation on the server or the browser and one can call any method without regard to where it lives. 
Under the covers Amorphic will setup an xhr call AND synchronize the entire object graph so that the state
of your application is up-to-date before the method executes.  In other words you have to do nothing special
to execute methods on the server directly from the browser and to the extent that the data you need is a
part of your objects you need to do nothing special to pass data back and forth.

Of course being a front-to-back platform means that you need other tools to create functional applications:

- Data binding to bind the page to and from your perfectly synchronized objects

- A persistence mapper to store your objects to and fetch them from a mongoDB database

- A session-based container to serve up the application and keep every user's objects independent