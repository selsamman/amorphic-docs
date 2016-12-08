---
layout: page
title: "setup"
category: intro
date: 2014-01-01 00:00:00
order: 0
---

### Installation

There are two ways to get started:

* Fork the ticket demo
* install into an existing project

The ticket demo has some ready examples of complex relationships in the model so this can be a good bet for those who want to delve in quickly.  If you already know amorphic or want to take a more methodical approach or already have some existing code then installing Amorphic as an npm is best:

    npm install amorphic

Amorphic is based on connect middleware. It creates a connect applicaiton and listens on it.  All you need in your app.js is to call amorphic:
 
    require('amorphic').listen(__dirname, sessionStore, preSessionCallback, postSessionCallback);
    
Parameters are:

* **root directory** which should always be __dirname
* **sessionStore** a session store object which defaults to MemoryStore
* **preSessionCallback** a call back that passes in the app object returned from connect() before any .use events are attached.  This is where any special requirements for static files or other requests that don't need the bodyParser or sessions can be attached via .use
* **postSessionCallback** a call back that passes in the app object returned from connect() after app.session() and the body parser have been called such that requests that require session can be attached.

Note that **preSessionCallback** and **postSessionCallback** not normally needed in an Amorphic application because communication with the server is typically handled by object model methods declared live on the server.  This communication is handled automatically.

### Global config.json

In the root of your project there is a **config.json** file which represents the "global" configuration for all applications:
* **conflictMode** - Whether to data synchronization vs only message synchronization is enabled.  Defaults to 'soft'.  Every data value transmitted when synchronizing between the browser and server is sent in two parts - the expected current value and the new value.  If the expected current value is incorrect a warning is generated (conflictMode 'soft') or an error is generated (conflictMode 'hard')
* **compressSession** if set to true session data is compressd before being serialized.  This makes storage smaller but adds time to each server call
* **compressXHR** if set to true the server responses are compressed
* **sourceMode** Whether source files are to minified 'prod' or left as is 'debug'

Anything you specify in config.json can also be specified as a starting parameter to node.js

In addition there is a configuration file for each application in the app directory which contains:
* **port** - the port you want to listen on
* **sessionSeconds** - how long before a session expires (in seconds)
* **objectCacheSeconds** - how long to keep your objects cached
* **sessionSecret** - a random string for hashing sessions
* **applications** - list your applications and their root directories here
* **application** - the default application

Example:

    {
        "port": 3001,
        "sessionSeconds" : 3600,
        "objectCacheSeconds" : 600,
        "sessionSecret" : "Ey0veeljPcLsHitdxCG2",
        "applications" : {"ticket": "apps/ticket"},
        "application"  : "ticket"
    }

### Daemons (background tasks)
 
Sometimes you need things running on the server that are not related to a browser session.  To do that include this in your config.json file:
 
       "isDaemon": "true",

Your controller will be created immediately upon startup and the serverInit function will be called for it to get started.
 
     module.exports.controller = function (objectTemplate, getTemplate)
     {
        var BaseController = getTemplate('./baseController.js').BaseController;
     
         Controller = BaseController.extend(
        {
             serverInit: function () {
                 setInterval(function () {this.interval()}.bind(this), 5000);
             },
             interval: function () {
                 console.log("I'm a batch task");
             }
     
         });
     
         return {Controller: Controller};
     }

 You can also listen on any port at this point but you don't have access to the main server that is listening for http requests from the browser.
 
 At present daemons are executed on the same thread as the main application.  The thinking is that in a production environment you would probably deploy them separately on separate node.js processes.  You can control what gets started by overriding the applications property of your config.json with an environment variable or node.js startup parameter.