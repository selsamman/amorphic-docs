---
layout: page
title: "installation"
category: intro
date: 2014-01-01 00:00:00
order: 0
---

### Installation

There are two ways to get started:

* Fork the ticket demo
* install into an existing project

The ticket demo has some ready examples complex relationships in the model so this can be a good bet for those who want to delve in quickly.  If you already know amorphic or want to take a more methodical approach or already have some existing code then installing Amorphic as an npm is best:

    npm install amorphic

Amorphic is based on connect middleware and it it creates an app, does everything it needs and then listens for http requests. 
 
    require('amorphic').listen(__dirname, sessionStore, preSessionCallback, postSessionCallback);
    
Parameters are:

* **root directory** which should always be __dirname
* **sessionStore** a session store object which defaults to MemoryStore
* **preSessionCallback** a call back that passes in the app object returned from connect() before any .use events are attached.  This is where any special requirements for static files or other requests that don't need the bodyParser or sessions can be attached via .use
* **postSessionCallback** a call back that passes in the app object returned from connect() after app.session() and the body parser have been called such that requests that require session can be attached.

Note that **preSessionCallback** and **postSessionCallback** not normally needed in an Amorphic application because communication with the server is typically handled by object model methods declared live on the server.  This communication is handled automatically.

### Global config.json

In the root of your project there is a config.json file which represents the "global" configuration for your applications.

    {
        "port": 3001,
        "sessionSeconds" : 3600,
        "objectCacheSeconds" : 600,
        "sessionSecret" : "Ey0veeljPcLsHitdxCG2",
        "applications" : {"ticket": "apps/ticket"},
        "application"  : "ticket"
    }
    
The elements are as follows:

* **port** - the port you want to listen on
* **sessionSeconds** - how long before a session expires (in seconds)
* **objectCacheSeconds** - how long to keep your objects cached
* **sessionSecret** - a random string for hashing sessions
* **applications** - list your applications and their root directories here
* **application** - the default application

Anything you specify in config.json can also be specified as a starting parameter to node.js so you can use config.json and is also available to your application.

### Application directory

You can have multiple applications running in Amorphic.  Each one, however, will effectively get it's own session.  You might, for example, have a customer facing application an administrative facing application and a batch application.  While testing you might wish to run all of these on the same node.js application but in production you may choose to deploy them to there own dedicated servers.  To accomplish that just provide --applications and --application parameters to override which applications get run.
 
 The application directory is structured like this
 
    apps/common - A special directory for a common data model
    apps/app1 - root for application 1
    apps/app2 - root for application 2
    
 Each app directory is further broken down into 
    
    js - a place for server only files (not currently implemented)
    public/js = a place for your object model
    public/xxx - any other directory you need (e.g. css, img etc)
    config.json - an application level configuration file
    schema.json - your persistence schema
    
 The apps/common directory has
 
    js - a directory that will always be a secondary source for model files
    schema.json - your persistence schema if you have a common model
    
### Application config.json 
 
In each application directory there is config.json file to configure each application

    {
        "dbPath":       "mongodb://localhost:27017/",
        "dbName":       "dev",
        "createControllerFor": "#",
        "ver":          "0",
        "modules":      {
            "userman":  {
                "require": "amorphic-userman",
                "controller": {"require": "controller", "template": "Controller"},
                "principal": {"require" : "Customer", "template": "Customer"},
                "validateEmail": true
            },
            "mandrill": {
                "require" : "amorphic-mandrill",
                "controller": {"require": "controller", "template": "Controller"},
                "senderEmail" : "sams@elsamman.com",
                "senderName": "Amorphic Team"
            }
        }
    }

The key parts are:

* **dbPath** - the connection string for MongoDB.  Note that this will be overridden by the global configuration property xxx_dbPath which means it can be specified as a node parameter (or environment variable) in production.

* **dbName** - the MongoDB database name which can also be overridden as xxx_dbName

* **createControllerFor** - Because Amorphic needs to create a controller as the backbone of any browser session you may not always want to do this if the user is simply visiting static marketing pages. createControllerFor is a RegEx string that is matched against the URL to determine if a controller should be created.  Generally you will make it something that will match everything (like .*) if you always want to have a session or something that will match nothing (like #) if you don't want to create a session until the broswer first makes an explicit request of the server.

* **modules** - Declare node modules specifically designed for Amorphic. The specific configuration for configuring these modules depends on the module itself but all have a *requre* property to specify the directory in node_modules that is to be required.  Usually the modules will "mixin" functionality to your object model and you configure the specific object templates that these changes are to apply to.  See the readme.md of each module for more info.

### Defining the Object Model

The object model includes both the model and the controller since they are both objects defined using SuperType which is Amorphic's type system.  The controller has some special behaviour and is not persistent where as the model is generally persistent.  Except for daemons you define your object templates in files app/xxx/public/js and every object model must have a controller.js file that returns a controller template.  Here is the helloworld controller.js

    module.exports.controller = function (objectTemplate, getTemplate)
    {
        var BaseController = getTemplate('./baseController.js').BaseController;
        var World = getTemplate('./world.js').World;
    
        Controller = BaseController.extend({
            worlds:        {type: Array, of: World, value: []},
    
            newWorld: {on: "server", body: function ()
            {
                this.worlds.push(new World());
            }}
    
        });
        return {Controller: Controller};
    }

and the model world.js

    module.exports.world = function (objectTemplate, getTemplate)
    {
        var World = objectTemplate.create({
            createdAt: {type: Date, rule: "datetime"},
            init: function () {
                this.createdAt = new Date();
            }
        });
        return {World: World}
    }


All object model files are structured such that Amorphic can "include" them on both the server and browsers.  In the future there will be a method to have templates defined only on the server but for now everything is isomorphic.  The templates chapter covers the type system in detail but here we clarify how the files must be structured such that Amorphic and manage your object model for you.

The structure has these elements:

* You export a property *controller* that must match the name of the containing javascript file.  This let's Amorphic know where to get your template definition function when it includes your file and also ensures that when executed in the browser each file will return a unique export. 

* The template definition function is executed by Amorphic to create your object templates which are akin to classes in a classical OO language. Each session effectively has it's own private set of templates instances such that when you do new XXX() you can be sure that the object created will be connected to your session.  Two parameters are passed in:

  * objectTemplate - the factory from which you create templates
  * getTemplate - function that you use in place of *requre* to include other object model files
  
* You can return as many templates as you like in an associative array. Here we are return just one.

### Circular References

Robust object models often have circular references.  Within a template file this can be accomplished using objectTemplate.mixin:
 
    var Foo = objectTemplate.create("Foo", {
        foo1: {type: Number}
        
    });
    var Bar = objectTemplate.create("Bar", {
        foo: {type: Foo}
    });
    Foo.mixin({
        bar: {type: Bar}
    });
 
If you have files that require each other Amorphic provides a solution in the form of a two-pass processing of the object model.  The first pass starts with your controller and then processes all of your templates by calling the function exported with the same name as the file being included.  (e.g. module.export.controller).  After that it looks for exports with the same name but with _mixin (e.g. module.export.controller_mixin) and calls that function.  The mixin function  
 
    // foo.js
    module.exports.foo = function (objectTemplate, getTemplate)
    {
        var Foo = objectTemplate.create("Foo", {
            foo1: {type: Number}
            
        });
    }
 
    // bar.js
    module.exports.Foo_mixins = function (objectTemplate, requires)
    {
        var Bar = requires.foo.Foo;
        var Foo = requires.bar.Bar;
     
        Foo.mixin({
            bar: {type: Bar}
        });
    }
    
### Including server files
 
You may include files only on the server in template definitions by using require.  However be sure to test to see that require exists since the same template definitions are included on the client.
 
### Background Tasks
 
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
 
 At present deamons are executed on the same thread as the main application.  The thinking is that in a production environment you would probably deploy them separately on separate node.js processes.  You can control what gets started by overriding the applications property of your config.json with an environment variable or node.js startup parameter.