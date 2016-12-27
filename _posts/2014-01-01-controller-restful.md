---
layout: page
title: "RESTful Applications"
category: controller
date: 2013-06-06 08:55:36
order: 2
---

# RESTful

You can execute RESTful code using the daemons.  In that case you simply listen for incoming calls from within the serverInit method. On each call you would need to update the database using Persistence calls.  No state is maintained between calls.  If you need state you need to manage it on your own using Connect.  Here is a sample that uses Restify:

    module.exports.controller = function (objectTemplate, getTemplate)
    {
        var BaseController = getTemplate('./baseController.js').BaseController;
    
        Controller = BaseController.extend("Controller",
            {
                serverInit: function () {
                    var restify = require('restify');
    
                    function respond(req, res, next) {
                        res.send('hello ' + req.params.name);
                        next();
                    }
                    var server = restify.createServer();
                    server.get('/hello/:name', respond);
                    server.head('/hello/:name', respond);
                    server.listen(objectTemplate.config.rport, function() {
                        objectTemplate.logger.info(server.name + ' listening at ' + server.url);
                    });
                },
            });
    
        return {Controller: Controller};
    }
    
