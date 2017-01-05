---
layout: page
title: "Directories"
category: config
date: 2014-01-01 00:00:00
order: 1
---

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

* **dbdriver** - The database driver.  Defaults to mongo but can be set to **knex**, the underlying middleware for Postgres. 

* **dbtype** - The type of database (only needed if **dbdriver** is set to **knex**) - Should be set to **pg** for knex/Postgres support.  In theory any database supported by knex (http://knexjs.org/) is supported though testing has only been done with Postgres. 

* **dbpath** - The hostname for the database.  When running your database locally for testing purposes set this to **127.0.0.1** 

* **dbname** - The database name to connect to  

* **dbuser** - A user name if the database requires authentication. 

* **dbpassword** A password if the database requires a password

* **createControllerFor** - Because Amorphic needs to create a controller as the backbone of any browser session you may not always want to do this if the user is simply visiting static marketing pages. createControllerFor is a RegEx string that is matched against the URL to determine if a controller should be created.  Generally you will make it something that will match everything (like .*) if you always want to have a session or something that will match nothing (like #) if you don't want to create a session until the broswer first makes an explicit request of the server.

* **modules** - Declare node modules specifically designed for Amorphic. The specific configuration for configuring these modules depends on the module itself but all have a **require** property to specify the directory in node_modules that is to be required.  Usually the modules will "mixin" functionality to your object model and you configure the specific object templates that these changes are to apply to.  See the readme.md of each module for more info.

* **controller** - The file name for the root file which must contain a template called Controller.  The convention is to specify Controller.js here. 

### Defining Templates
 
All code and data in amorphic is defined as object templates using Amorphic's type system.  These files reside in:

* **app/<appname>/public/js** - the normal place for web-based applications
* **app/<appname>/js** - for non-interactive "batch" applications
* **app/common/js** - if is shared accross multiple applications
 
Of course you may use sub-folders to organize your templates.
 
You must have a controller file which would look something like this:

    // Controller.js
    module.exports.Controller = function (objectTemplate, uses)
        var Model = uses('model.js', 'Model');
    {    
        Controller = objectTemplate.create("Controller", {
            model: {type: Model}
            clientInit: function () {this.model = new Model()}
        });
    }

Templates  are structured such that Amorphic can "include" them on both the server and browsers. The structure has these elements:

* You export a property, controller in the above example that must match the name of the containing javascript file that is a function that Amorphic will call to create the templates for a particular user session.  This function is passed:

  * objectTemplate - a factory object which you can use to create templates (Controller = objectTemplate.create("Controller", {})
 
  * uses - a function that you use in place of **require** to include other template files (var World = getTemplate('./world.js').World).  You don't have to worry about circular references.  In fact you can refer to your own file if you have circular references within a file.
  
* The function returns an object with a property for each template defined in the file (return {Controller: Controller}) You can define and return multiple templates.
 
This structure allows each session to have it's own private set of templates instances such that when you do new Controller() you can be sure that the object created will be connected to your session. 

### Circular References (Legacy)

Prior to version 1.4 of Amorphic templateMode 'auto' was not supported and a different way of handling circular references was used.  In legacy mode you must return an object with a reference to each template you created in that file.  Within a template file circular references were handled using objectTemplate.mixin:
 
    // foobar.js
    module.exports.foobar = function (objectTemplate, getTemplate) {
        var Foo = objectTemplate.create("Foo", {
            count: {type: Number}
        });
        var Bar = objectTemplate.create("Bar", {
            foo: {type: Foo}
        });
        Foo.mixin({
            bar: {type: Bar}
        });
        return {
            Foo: Foo,
            Bar: Bar
        }
    }
    
Because your model may grow later and end up with circular references an alternative is to use mixins for everything.
 
    // foobar.js
    module.exports.foobar = function (objectTemplate, getTemplate) {
        var Foo = objectTemplate.create("Foo", {});
        var Bar = objectTemplate.create("Bar", {});
        Foo.mixin({
             count: {type: Number}
             bar: {type: Bar}
        });
        Bar.mixin({
             foo: {type: Foo}
        });
        return{
            Foo: Foo,
            Bar: Bar
        }
    }

Mixins deal with circular references in a single file but what if you have circular references across files?
  
Amorphic provides a solution in the form of a two-pass processing of template files:
  
* Pass 1 - your export property function (expected to be the same name as the file it is in) is executed and template definitions are returned and accumulated
* Pass 2 - If there exists an xxx_mixin export that corresponds to one of the files it is called.  In that function you can mixin properties but you cannot define new templates.  The mixin exported function is passed in an object with a property for each export that contains a further property for each template.  

This allows you to reference any templates defined in the first pass.

 
    // foo.js
    module.exports.foo = function (objectTemplate, getTemplate)
    {
        var Foo = objectTemplate.create("Foo", {
            count: {type: Number}
        });
        return{
            Foo: Foo,
        }
    }
    module.exports.foo_mixins = function (objectTemplate, requires)
    {
        var Bar = requires.foo.Foo;
        var Foo = requires.bar.Bar;
        Foo.mixin({
            bar: {type: Bar}
        });
    }
    
    // bar.js
    module.exports.foo = function (objectTemplate, getTemplate)
    {
        var Foo = getTemplate('bar.js');
        var Bar = objectTemplate.create("Bar", {});
        return {
            foo: Foo,
        }
    }
    
    
### Including server files
 
You may include files only on the server in template definitions by using require.  However be sure to test to see that require exists since the same template definitions are included on the client.