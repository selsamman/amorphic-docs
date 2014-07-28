---
layout: page
title: "installation"
category: config
date: 2014-01-01 00:00:00
order: 0
---

# Installation

There are two ways to get started:

* Fork the ticket demo

* install into an existing project

The ticket demo has some ready examples complex relationships in the model so this can be a good bet for those who want to delve in quickly.  If you already know amorphic or want to take a more methodical approach or already have some existing code then installing Amorphic as an npm is best:

 npm install amorphic

Amorphic is based on connect middleware and it it creates an app, does everything it needs and then listens for http requests.  Future versions will be more flexible in "sharing" the server setup.  You need to either use the amorphic listen function or clone it and modify it
 
   require('amorphic').listen(__dirname);

You can also pass a session store object after the directory name if you want to use a cache such as REDIS for you sessions.

# config.json

In the root of your project there is a config.json file

    {
        "port": 3001,
        "sessionSeconds" : 3600,
        "objectCacheSeconds" : 600,
        "sessionSecret" : "Ey0veeljPcLsHitdxCG2",
        "applications" : {"ticket": "apps/ticket"},
        "application"  : "ticket"
    }
    
