---
layout: page
title: "Persistence"
category: model
date: 2013-06-06 08:55:36
order: 0
---

# Persistor object mapper

Amorphic will manage your object relationships for you both across the wire and across persistent storage.  All of this is done without having to manage ids to maintain the relationship.  If you are familiar with Java persistence middleware like hibernate this will seem familiar.  It is different from the way that Mongoose maps persistent data to objects in these ways:

* In Mongoose a schema defines the properties of your object model and their relationship to collections.  With persistor SuperType is used to define your object model and the schema defines the relationship of the model to collections.

* Mongoose does not attempt to create unique objects for unique database instances.  For example if you retrieve a number of objects that each have references to another object and there are multiple instances of the same object, Mongoose will create separate instances.  Persistor, like Hibernate will ensure that you only have one instance of an object. 

* Persistor treats objects mapped to sub-documents much the same as those mapped to the main document. That is it gives them unique database ids and changes all references to them to references to id's.  This means you have the same instance of an sub-document object in a document and have circular references.  At present you cannot directly retrieve a sub-document though that will be supported in future versions.  This is possible because sub-documents embed the main document key in their unique id.

# The Schema

You need a schema to tell Persistor how to map your object templates to collections:
    
    {
        "Ticket": {
            "documentOf": "ticket",
            "children": {
                "ticketItems":        {"id":"ticket_id"}
            },
            "parents": {
                "creator":            {"id": "creator_id"},
                "release":            {"id": "project_release_id"},
                "project":            {"id": "project_id"}
            }
        },
        "TicketItem": {
            "documentOf": "ticketItem",
            "parents": {
                "creator":            {"id": "creator_id"},
                "ticket":             {"id": "ticket_id"}
            }
        },
        "TicketItemComment": {
            "subDocumentOf": "ticketItem",
            "children": {
                "attachments":        {"id": "ticket_item_id"}
            }
        },
        "Controller": {
        }
    }

The schema is an object where each property must the same name as you used when creating the object template.

	var Ticket = objectTemplate.create("Ticket", {property definitions ....}};
	
The schema entry for each object template contains:
	
* **documentOf** or **subDocumentOf" - the name of the collection this object template belongs to either as the main document or the sub-document

* **children** a list of the properties you have that reference arrays of objects that are part of another collection.  In other words your one-to-many relationships.  There is no need to mention references to sub-documents.  You specify the name of the id property that will be used in the reference to tie it to this object

* **parents** a list of properties you have that reference individual objects in another collection.  In other words your many-to-one or one-to-one references.  Again there is no need to reference sub-documents and you specifiy the name of the id property that be used in this object to refer to the foreign object.

A few notes on using schemas

* You must include all sub-document objects

* If you use inheritence then you must include all sub-classes but you only need specify the references (parent, children) unique to the sub-class.

* Persistor will attempt to throw a helpful error when you are missing schema entries but generally this is the first place to look if you have problems persisting or retrieving data.

* Include non-persistent templates (like controller) if they have references to persistent templates that you want have filled automatically

### Saving

To 

### Fetching 


### Cascading

### Bindster Integration

### Concurrency model