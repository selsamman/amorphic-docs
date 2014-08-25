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

* Persistor treats objects mapped to sub-documents much the same as those mapped to the main document. That is it gives them unique database ids and changes all references to them to references to id's.  This means you have the same instance of an sub-document object in a document and have circular references.  As long as you define foreign key relationships in the schema you have one-to-many and one-to-one relationships involving sub-documents that span a document.

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

To save an object to the database use the persistSave method that is injected into every object:

    Ticket.persistSave().then(function () { // ticket has been saved at this point });
    
When you save an object, Persistor will ensure that all foreign keys are inserted to maintain the same relationships in the database that you haved defined in memory.  Specifically this means:
  
* Where you reference another individual template (e.g. project: {type: Project}) and include the foreign key relationship in the schema (e.g. parents: {project: {id: 'project_id'}) the id of the referenced object will be updated in the object your are saving.
   
* Where you reference an array of templates (e.g. attachments: {type: Array, of: TicketAttachment}) and include the foreign key relationship in the the schema (e.g. children: {attachments: {id: 'ticket_id'}}) the id of the referencing object will be inserted in the referenced object
  
The key maintenance functions are cascaded down as far as needed and any objects whose keys were modified are automatically saved.
  
Knowing what to save does involve some knowledge of the document/sub-document relationships.  You save the objects that are the primary object in the document (e.g. are declared with documentOf in the schema) or objects where you have defined parent/child relationships from the primary object down to the object you are saving.  Attempting to save an object that is a sub-document where no such relationship has been declared will throw an error.

### Fetching 

There are three ways to fetch fetch an object.  These methods, by design, are only permitted on the server

If you know it's id (_id) value you can use **getFromPersistWithId**

    Customer.getFromPersistWithId(idValue).then (function (customer) {
        // Do something with customer
    });
    
If you want get an array of objects that match a given criteria use **fetchFromPeristWithQuery**

    Transaction.getFromPersistWithQuery(
    {'$or':[{type: 'debit'}, {type: 'credit'}]}).then (function (transactions) {
        // Do something with transactions returned
    });

If you want to fetch a related object (and it was not automatically fetched because of cascading) you use just fetch on the object itself:

    return customer.roles[0].fetch({account: true}).then( function () {
        //customer.roles[0] will now be populated with it's related account
    });  
    
Note that you can specify multiple objects to fetch
 
 
### Cascading

When you have a reference to another object in the schema you can have Peristor automatically fetch that reference even if it is another document by applying the **fetch** option in one of these three ways:

* In the reference itself (e.g. **roles: {type: Array, of: Role, value: [], fetch: true})**

* In the schema parent or children declarations (e.g. **"children": {"ticketItems": {"id":"ticket_id", "fetch":"true}})**

* When you get or fetch the object (e.g **Customer.getFromPersistWithId(sam._id, {roles: true}).then ()**

Note that the value of fetch can in fact specify further sub-levels to automatically fetch

    return customer.roles[0].fetch({account: {fetch: {roles: {fetch: {customer: true}}}}}})

### Remote Integration

There are times when code in the browser wants to fetch related objects that were not returned because of cascading.  This is referred to as remote cascading. To allow this to happen you must use a fetch type of "remote" as in

    roles: {type: Array, of: Role, value: [], fetch: "remote"})**
    
This can, of course, be specified anywhere you can apply **fetch** including in the schema or in the getFromPeristWithQuery/Id

When you specify fetching to be **remote** a **getter** function is generated that has the same name as your property and you can refer to this anywhere

    customer.rolesGet().then (roles) {
        // Do something
    }

The getter is smart enough to not go to the server if the result as already been returned or the fetch is in progress. Under the covers the getter calls the fetcher. A  **fetcher** function is generated that has the same name as your property and you can refer to this anywhere.  

When using this within Amorphic's data binding framework (Bindster) you can use the getter to test to see if a related entity exists or not and have it automatically fetched only when that markup is actually displayed.  This avoids having to have code to fetch related objects for the benefit of the browser.

Imagine that a customer is related to a set of roles and each role is linked to an account owned by that customer.  Imagine that no cascading is specified and instead but we want to list all the accounts owned by a customer.
     
     <b:iterate on="customer.rolesGet()" with="role">
        <div b:bind="role.account.number" b:showif="role.accountGet()">
     </b:iterate>

While this might not be the most efficient way to structure things it will have the effect of displaying all of the accounts because rolesGet will cause the roles to be fetched and then each role's account number is displayed only if the the account is fetched.  The fetching of the account is initiated by accountGet().

### Concurrency model

MongoDB has not concept of transactions and it's atomicity is limited to a single collection.  This puts the burden on the developer to not be sensitive to changes that must be made across collections.  In addition isolation is only available through the findAndModify pattern which unfortunately does not support all cases of where you might want to update sub-documents in an atomic fashion.  Persistor makes heavy use of sub-documents and so this is not a practical way of updating documents.  Instead Persistor uses a form of optimistic locking as follows:
 
 * Every collection has a sequence number _sequence
 
 * The first time a record is saved _sequence is zero
 
 * Every update is subject to query that tests to see that _sequence has the same value as when the data was originally retrieved
 
 * Every update attempts to increment _sequence by providing a new value that is one more than the old one
 
 * If no records are updated then another client has intervened and updated the data.  In that case the data is refreshed, the changes made since the data was retrieved are overlaid and the update process is repeated
 
 * Optionally a callback can be invoked which can determine whether or not to actually repeat the update operation and also has the opportunity to examine the new data and manually adjust the data before repeating (e.g. to ensure integrity of the data)
 
 Note that if you don't provide a call back and don't update sub-documents then the version number is not checked and only incremented.