---
layout: default
title: "Amorphic"
category: "config"
---

### Welcome to Amorphic

Amorphic is a full-stack, isomorphic framework for creating web-based applications with Node.js and MongoDb.

The goal of Amorphic is to let you focus on the logic of your application with minimal regard for:

* Where code executes - code executes in the browser or on the server with seamless calls between the two
* Dealing with a DB - objects and their relationships can be saved and retrieved automatically
* How the DOM works - two way data binding means no glue code

### How it works

You define objects that live on both in the browser and on the server as part of user's session with the server.  Your object methods are defined either as executing in the browser or on the server.  You may call methods on the server from within methods on the browser. When you do so all properties are synchronized and your call is executed on the server. The upshot is that you choose where to execute your code based criteria such as speed, proximity to persistent storage or security but the structure and coding style does not have to vary (except for access to server only resources).

To keep track of all this you need to define your objects using Amorphic's templating system which lets you define object, their properties and relationships to other objects.  As a bonus this definition also serves as the core a database schema that is lightly augmented with an external schema file to take care of data base dependent considerations like object to table mappings and foreign keys.  You can save and and retrieve objects and their relationships to any depth with a simple call with full ACID support if Postgres is chosen as the database.

Finally mapping your objects to the screen uses data-binding that is analogous to Angular except that it has tighter integration with your object definitions.  For example you can define values and descriptions for a multi-valued property as part of your object template, bind that to a select control and it will automatically populate the options.

### Cascading Persistence

The persistence model in Amorphic is simple.  Your model defines the relationships between objects (one-to-many and many-to-one), you create your object graph using normal references and arrays (for one-to-many) and then when you save, Amorphic walks through graph and saves what it has to. Then when you retrieve objects, Amorphic will fetch all referenced objects either immediately or upon request.

You need to define what object templates belong in what document collections and what object templates are sub-documents.  You also need to define foreign-key references but you don't have to have a property-level schema.  

### Model, View, Controller

![Model, View, Controller](img/mvc.png)

Amorphic is built on a model, view, controller paradgym

* **Model** - Amorphic make it possible to create rich object models that are easily persisted with minimal schema information.
 
* **View** - Is the HTML adorned with data-bind attributes and augmented by special HTML tags (similar to directives in Angular) that give you iteration and conditional constructs.

* **Controller** - In Amorphic there is one main controller that is at the root of the application and is essentially a session for a single user. You may compose more granular controllers to keep things modular.  The main controller is the scope for data binding in the view.

### Session Oriented
 
 Each web browser session gets it's own session and it's own "object space".  That is all object instances are unique to a user's session.  You cannot share objects between sessions.  If you want to share data across multiple users this is done via a cache (REDIS) or the database.  This ensures that as you scale your application past a single server you won't have created dependencies on two sessions living on the same physical server.
 
 Amorphic manages sessions using Connect and it's session semantics.  This means that if you configure Redis as your session store you can load balance across multiple servers.  Amorphic is optimized for sticky sessions.  
  

