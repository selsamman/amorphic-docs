---
layout: page
title: "Batch Applications"
category: controller
date: 2013-06-06 08:55:36
order: 1
---

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