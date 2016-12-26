---
layout: page
title: "Online Applications"
category: controller
date: 2013-06-06 08:55:36
order: 0
---

# Online Applications

### The Role of the Controller

The controller object is the central object in an online application.  A single instances is created for each session and it serves as the anchor for all other objects in the session.  That is to say that any object that is to be part of the session must be referenced directly or indirectly by the controller. 

Practically speaking you may organize your application with smaller controllers that controller a smaller scope of work.  Generally this is based on a scope of work rather than an individual page since controllers are in no way associated with "pages".

These smaller referenced controllers, it should be noted, are not controllers as far as Amorphic is concerned.  Amorphic only knows about your main controller.  You are responsible for managing the life cycle of these composed controllers.

### The Life Cycle of a Session

### The Life Cycle of a Server Call

### Controller Events