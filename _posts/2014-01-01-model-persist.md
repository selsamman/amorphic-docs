---
layout: page
title: "Persistence"
category: model
date: 2013-06-06 08:55:36
order: 0
---

# Persistor object mapper
Persistor stores objects in a database and retrieves them.  I manages one-to-one and one-to-many relationships between objects and less you navigate these relationships synchronously or asynchronously (lazy loading).  In order to use persistor you need to:
 * Create your templates and their properties.
    * a property can refer to another template which forms a one-to-one relationship
    * a property can refer to an array of templates which forms a one-to-many relationship
 * Create a schema which describes how the templates are mapped to the store
    * which table or collection will store them
    * which additional columns or properties need are used as foreign keys
 * Use template and object level functions to read/write to the database
    * template-level functions are used for reading a set of objects and returning them
    * object level functions are used for saving the data to the database
 * Additionally for databases the support transactions there are semantics for
    * performing set of operations in a single transaction
    * rolling back
    * managing update conflicts
# Defining templates

Individual template properties have these special options for persistance:
* **persist** a boolean value that defaults to true indicating whether or not the property should be saved to the database
* **fetch** when applied to a reference to another templated object, indicates whether or not the referenced object should be fetched automatically. See cascading below.
* **notnull** a boolean that when true indicates that the property must be not-null at the time the object is saved to the database

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
        }
    }

The schema is an object where each first-level property is the name of a template as you specified when creating the template.

	var Ticket = objectTemplate.create({name: "Ticket"}, {property definitions ....}};
	
The schema entry for each object template contains:
	
* **documentOf** or **subDocumentOf** - the name of the collection this object template belongs to either as the main document or the sub-document.  When using a SQL-based database the documentOf let's you treat a group of templates as a collection for the purpose of saving them all as a group.

* **children** an object with a proporty for each one-to-many relationships to other templates with subordinate properties:

    * **id** to define a column that will form the relationship.  This column is automatically added to the child table.
     
    * **fetch** an parameter that specifies whether the parent should automatically be fetched (see cascade below).
   
  There is no need to reference parents in the same document when using a document-centric database such as MongoDB.  Here is the template property for ticketItem of the Ticket template that corresponds to the schema above:
  
        ticketItem: {type: Array, of: TicketItem}

* **parents** an object with a property for each one-to-one relationship defined in the template with a subordinate properties:

    * **id** to define a column that will form the relationship.  This column is automatically added to the table of the template containing the reference to the parent.
     
    * **fetch** an parameter that specifies whether the parent should automatically be fetched (see cascade below)
     
  There is no need to reference parents in the same document when using a document-centric database such as MongoDB.  Here is the template property for the creator property of the TicketItem template that corresponds to the schema above:
   
        creator: {type: Ticket}

A few notes on using schemas

* If you extend templates then you only need to include extended templates that have relationships not in the base templates and then only need to specify the references unique to the extended template (e.g. parent, children) unique to the sub-class.

* Persistor will attempt to throw a helpful error when you are missing schema entries but generally this is the first place to look if you have problems persisting or retrieving data.

### Saving

To save an object to the database use the save() method.

    ticket.save({options}).then(function () { post-save code })

The save options are:

* **transaction** for databases that provide transactions, a transaction that was returned by the objectTemplate.beginTransaction.  Note that if you don't specify a transaction and a default transaction was started by calling objectTemplate.beginDefaultTransaction(), the default transaction is used.  With a transaction the save is deferred until you call objectTemplate.commit().

* **cascade** for SQL databases, a true value indicates that all related objects that are in the same document (have same documentOf, subDocumentOf values in the schema) are also to be save.  This is only applicable to non-daemon (e.g. online) applications that have a change mechanism as this will only save objects that have been modified. Only the online applications have a mechanism to detect changes. With Daemon applications you need to save each object that you modify. With MongoDB all sub-documents are saved automatically.

* **logger** you may pass in a supertype logger created by objectTemplate.createLogger() or createChildLogger that will be used to log any data.  Usually you create a child logger and pass in context information you want logged.

Although a promise is returned from save(), the promise will be resolved **before** the data is actually written if the save is in the context of a transaction.  If no in the context of transaction the promise is resolved once the data is written.  If you need to execute code within the context of a transaction after everything is commited use the promise returned from objectTemplate.commit().  
   
When you save an object, Persistor will ensure that all foreign keys are inserted to maintain the same relationships in the database that you haved defined in memory.  Specifically this means:
  
* Where you reference another individual template (e.g. project: {type: Project}) and include the foreign key relationship in the schema (e.g. parents: {project: {id: 'project_id'}) the id of the referenced object will be updated in the object your are saving.
   
* Where you reference an array of templates (e.g. attachments: {type: Array, of: TicketAttachment}) and include the foreign key relationship in the the schema (e.g. children: {attachments: {id: 'ticket_id'}}) the id of the referencing object will be inserted in the referenced object
  
The key maintenance functions are cascaded down as far as needed and any objects whose keys were modified are automatically saved.
  
Knowing what to save does involve some knowledge of the document/sub-document relationships.  You save the objects that are the primary object in the document (e.g. are declared with documentOf in the schema) or objects where you have defined parent/child relationships from the primary object down to the object you are saving.  Attempting to save an object that is a sub-document where no such relationship has been declared will throw an error.

### Fetching 

Data is read from the database with the fetch() method on the template.

    <template>.fetchById(id, {options}).then(function(result) {});

    <template>.fetch(query, {options}).then(function(result) {});

The parameters are:

* **query** results will contain an array of objects that meet the query constraints.  The query may be a MongoDB query string (a subset of which is supported in SQL-based databases) or for knex-based database a callback that is passed the knex object and can support the knex-based functions to create complex queries.

* **id** results will contain a single object.  The database id is passed here (retrieved from the _id property of a saved object).  **id** and **query** are mutually exclusive.

* **options** an object with properties representing the various options:

    * **fetch** a specification identical to the fetch parameter when defining templates or the fetch option in the schema which specifies whether or not to also fetch related objects.
    
    * **start** the zero-based offset of the first object in the result set to be returned
    
    * **limit** the maximum number of objects to be returned
    
    * **order** a set of property names whose value is -1 to indicated sorting down and +1 to indicate sorting up.  The set is ordered in terms of precedence (e.g. {mostImportantProp: 1, secondary: 1})  
    
    * **transient** if set to true, the results are not to take up space in the session (does not apply to daemons).  This both prevents the data form being transported to the browser and allows the object to be garbage collected at the end of a server call. This applies to all objects included sub-ordinate objects fetched as well.
    
    * **logger** you may pass in a supertype logger created by objectTemplate.createLogger() or createChildLogger that will be used to log any data.  Usually you create a child logger and pass in context information you want logged.

* **result(s)** either an array of results (fetch) or an individual result (fetchById)

Examples:

    Customer.fetch({query: {email: this.email}}).then(function (customers) {
        console.log(customers[0].firstName);
    });

    Workflow.fetchById(this.workflowId).then (....)
    
    Workflow.fetchById(this.workflowId, {fetch: {conversations: true}).then (....)

If you already have an object and it references other objects that were not automatically fetched via the fetch parameter you can fetch them with

    <object>.fetch({options}).then(function() {code executed after everything fetched});

The only supported options are **fetch** and **logger**.

Examples:
 
    customer.fetch({fetch: {policies: {owner: true}}}).then (....)
    
### Delete

### Fetch Cascading

When you have a reference to another object in the schema you can have Persistor automatically fetch that reference even if it is another document by applying the **fetch** option in one of these three ways:

* In the reference itself (e.g. **roles: {type: Array, of: Role, value: [], fetch: true})**

* In the schema parent or children declarations (e.g. **"children": {"ticketItems": {"id":"ticket_id", "fetch":"true}})**

* When you get or fetch the object (e.g **Customer.getFromPersistWithId(sam._id, {roles: true}).then ()**

Note that the value of fetch can in fact specify further sub-levels to automatically fetch

    return customer.roles[0].fetch({account: {fetch: {roles: {fetch: {customer: true}}}}}})

### Remote Integration

There are times when code in the browser wants to fetch related objects that were not automatically fetched.

    roles: {type: Array, of: Role, value: [], fetch: false})
    
For every reference to a persistent template Amorphic will add two additional member functions that can facilitate fecthing form the browser:

* **xxxGet()** where xxx is the property name (rolesGet in the above example). This will fire a call to the server xxxFetch() to retrieve the related object if the call has not already been fired.  This is a synchronous call designed to be embedded in HTML using Bindster.  After firing the call it returns either a null value or empty array depending on whether this is a scalar or array reference.  You use it to defer displaying the object until it is fetched.  If you call it again and the data has made it's way back to the browser the data will be returned.  Since pages are generally re-rendered any time a server call is completed (including the xxxFetch), the data will be displayed on the next render cycle.

* **xxxFetch()** where xxx is the property name (rolesGet in the above example) fetches the data.  Generally it is only used automatically with xxxGet.  In Javascript you would use <object>.get() which is asynchronous

Example:
     
     <b:iterate on="customer.rolesGet()" with="role">
        <div b:bind="role.account.number" b:showif="role.accountGet()">
     </b:iterate>

### Concurrency model

#### Optimistic Locking
Persistor uses a form of optimistic locking as follows:
 
 * Every collection has a sequence number \__version\__
 
 * The first time a record is saved \__version\__ is zero
 
 * Every update attempts to increment \__version\__ by providing a new value that is one more than the old one
  
 * Every update is subject to query that tests to see that _sequence has the same value as when the data was originally retrieved.  If this has changed an 'Update Conflict' exception is thrown.
  
 * The update conflict is caught for online applications (not daemons) and handled as follows:
 
    * the data in the session is restored to it's value at the start of the server call
    
    * the preServer method of the controller is called if present.  This code should refresh all persistent data
    
    * the changes from the browser are then applied
    
    * then with everything up-to-date the original method is called and absent any other intervening updates form another session should succeed.
    This is retried 5 times.
  
 Note that if you don't provide a call back and don't update sub-documents then the version number is not checked and only incremented.

#### Non-transactional databases
MongoDB does not have a concept of transactions and it's atomicity is limited to a single collection.  This puts the burden on the developer to not be sensitive to changes that must be made across collections.  In addition isolation is only available through the findAndModify pattern which unfortunately does not support all cases of where you might want to update sub-documents in an atomic fashion.  Persistor makes heavy use of sub-documents and so this is not a practical way of updating documents.  

 #### Transactional databases
 SQL databases have transaction semantics and can ensure all updates are successful or else rolled back.  In addition, the same optimistic locking model is used to detect update conflicts. 
 
 Because amorphic strives to limit the length of a transaction all updates are done in one single operation. 
 
 To begin a named transaction 
 
    var transaction = objectTemplate.beginTransaction();
 
 To begin a default transaction
    
    objectTemplate.beginDefaultTransaction();
    
To commit the transaction

    objectTemplate.commit(transaction).then({post commit code});