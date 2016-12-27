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

When a user first comes to an Amorphic site and a page is loaded, one of the first things that it will do is to include the Javascript for the session.

    <script src="/amorphic/init/<application-name>&version=<qualifier>
    
This will download what is need to get the application identified by **\<application-name\>** started.  It is important to attach a unique version numbers as a **\<qualifier\>** to ensure that this script is not cached.  This is what get's downloaded and executed:
 
* **templates** - all of the templates your application needs are downloaded either as a compressed minified file (sourceMode: prod) or as a series of \<script\> statements for each template file (sourceMode: debug).  Amorphic processes these template source files and sets up template functions upon which you can execute the new operator.  They are in the global scope (e.g. properties of window).
 
* **schema** - Schema information helps in the setting up of these templates to automatically generated getter and fetcher helper methods for inter-template references.

* **controller** - In some circumstances, the actual data for a controller is sent:

    * If a session already exists as would be the case when refresh is hit in the browser or you go to a different page, the controller data is always sent and a synchronized controller is instantiated in the browser

    * **createController set to yes** If not in an existing session, the controller is instantiated on the server and synchronized to the browser

    * **createController set to #** No controller is created on the server.  Instead the browser creates the controller and it will be synchronized on the server the next time any server methods are called.  This is an important option to allow a large swath of the application to bexecuted in the browser with virtually no server resources.
    
Once the session is started it will retain data for your controller and all referenced objects until it expires or until you call

    amorphic.expireController()
    
This call should be made when you want to free up resources on the server.  It will create a new controller in the browser that will be synchronized the next time you call code on the server.    

### The Life Cycle of a Server Call

When you call a method that is declared to live on the server from the browser a series of operations happen that ultimately allow the code to execute seamlessly on the server:

* **browser: proxy executed** - the first thing that happens is that the proxy stub that Amorphic used to replace your method on the browser is executed

* **browser: changes queued** - Amorphic looks at all of your data (that is your controller and all object refrenced by the controller recursively) to determine if anything has changed since the last call to code on the server.  These changes are packaged up.

* **browser: request queued*** - The server method is packaged up and queued in the browser.  If no call was in progress the request is immediately sent to the server.  Otherwise the next element of the queue is removed when the server returns from the call.

* **server: preServerCall** - If you have a **preServerCall** method in the controller, that method is called **before** any data is synchronized from the browser.  This gives you a chance to see if any data is stale (e.g. updated in the database by another session) and to refresh it.  Normally you would begin a Persistor transaction such that anything explicitly saved would be saved as part of a transaction:

        objectTempalte.beginDefaultTransaction()

* **server: changes processed** - The changes packaged up and sent to the server are applied to the controller and referenced objects

* **method called** - Finally your method is executed.  The state of the session is now identical to the way it was in the browser

* **server: postServerCall** - If you have a postServerCall method in the controller it is executed, giving you a chance to save anything that was modified either by the browser or code on the server.  Assuming you divide your model into "documents" you can do this saving all objects in the document like this:

        myObject.save({cascade: true})
        return myObject.end()
* **server: changes packaged and returned ** - Any data changed on the server is packaged up and sent back to the browser as the payload for the server call.

* **browser: process changes & return ** - When the server call returns the changes are applied and the browser returns to code that called the server

* **browser: view rendered** - The view is scheduled for rendering in case any data values were changed

### Controller Events

* **serverInit** - A method on the server that is executed the first time the controller executes on the server.  This must not return a promise and is synchronous in nature

* **clientInit** - A method on the server that is executed the first time the controller executes on the browser.  You use this to set up any browser dependent things that might involve the DOM.

* **shutdown** - A method on the browser that is called when the controller is destroyed.  It is important to cancel any timer events you have outstanding

* **preRender** - A method that is called before the data binding layer renders updated data into the view

* **postRender** - A method that is called after the data binding layer has updated data into the view

* **refresh** - A method client code can call to force the updating of the view.  This is needed if you use timer events since normally the view is only updated when:
  * A UX event occurs such as a clieck
  * When a server method is called