---
layout: page
title: "setup"
category: config
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

* **templateMode** When set to 'auto' templates are defined in a fashion where circular references are handles automatically.

* **port** - the port you want to listen on

* **sessionSeconds** - how long before a session expires (in seconds)

* **objectCacheSeconds** - how long to keep your objects cached

* **sessionSecret** - a random string for hashing sessions

* **applications** - list your applications and their root directories here

* **application** - the default application

Anything you specify in config.json can also be specified as a starting parameter to node.js


Example:

    {
        "port": 3001,
        "sessionSeconds" : 3600,
        "objectCacheSeconds" : 600,
        "sessionSecret" : "Ey0veeljPcLsHitdxCG2",
        "applications" : {"ticket": "apps/ticket"},
        "application"  : "ticket"
    }

