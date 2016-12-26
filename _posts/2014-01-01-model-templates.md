---
layout: page
title: "Type System"
category: model
order: 0
---

## Introduction to Supertype

Supertype is the type system used by Amorphic.  It supports creating function templates that instantiate objects via the Javascript new function

	var Person = objectTemplate.create("Person", {
        name: {type: String, value: ""},
        email: {type: String, value: ""}
    });
    
...
    
    var me = new Person();
    me.name = "Sammy"
    me.email = "sammy@elsamman.com";

Supertype supports a number of features for creating rich object models
* Extending templates (inheritance)
* Composition (including templates within templates)
* Relationships (one-to-one and one-to-many relationships to other templates)
* Rich definition of properties that interact with other parts of amorphic
    * Controlling whether or not the properties are persisted
    * Controlling whether properties are synchronized between browser and server
    * Defining validation, parsing and formatting when used with Bindster
    * Values and descriptions for multi-value fields that automatically populate selects and radio buttons when used with Bindster.

## Creating Templates

    var <template-name> = <object-template>.create(<create-params>,  <propertiesAndMethods>)

* **\<template-name\>** - the name of your template. 

* **\<object-template\>** - a parameter in a constructor function provided by the amorphic function (see directories) that allows templates to be created specifically for a given session.

* **\<create-params\>** - an object defining the parameters for the template or the name of your template:

    * **name** - the name of your template

    * **toServer** - optional property that can be set to false if the template or objects created from the template are never to be sent to the server.  If it is a string it is a the name of a rule that must be present in order for transmission to occur to the server.  This allows, for example, template transmission for say an admin application but not a customer facing application.

    * **toClient** - same as **toServer** but controls transmission from the server to the browser.  Useful for secure templates that should never be exposed to the browser
    
    Examples:

            var MedicalData = objectTemplate.create( {
                    name: "MedicalData",
                    toClient: false,  // Ensure not exposed
                    toServer: 'admin' // Only admin can update
                },{    
                name: {type: String, value: ""},
                condition: {type: String, value: ""}
            });


* **propertiesAndMethods** - an object that describes each data property or method that will be created with an objected is instantiated from this template. Each data property has sub-properties that define the property attributes.  

## Template Data Properties

### Basic Types

The most basic type attributes are:

* **type** the type of the property:
        
* **String**, **Number**, **Date**, **Boolean** - primitive Javascript types

* **Array** - used to represent a collection of other types.  When **Array** is used, **of** must also be present to define the **type** of the elements of the collection.  The type may be a primitive type or another template (the variable returned from createTemplate).  **Warning:** arrays of primitive types are very inefficient because Amorphic cannot optimize their synchronization between the browser and the server.  It is usually better to define a template even if it has only a single type.  Exceptions are small collections.

* **value** - the initial value when object is instantiated

  Examples:

            var Child = objectTemplate.create("Child", {
                name: {type: String, value: ""},
            });
            var Parent = objectTemplate.create("Parent", {
                name: {type: String, value: ""},
                children: {type: Array, of: Child},
                spouse: {type: Parent}
            });
### Synchronization Property Attributes        
In addition there are attributes that relate to how the property will be transported and/or persisted:
        
* **isLocal** - when set to true the property is not automatically synchronized between the browser and the server and are never persisted.  Useful to optimize large data elements that are only needed only on one environment.
        
* **persist** - when set to false the property is never persisted. 
        
* **toServer** - when set to false the property will never be transmitted to the server.  This makes the property secure in that no one can tamper with the property on the browser and have it automatically synchronize and thus end up on the server. You only modify such properties in code that executes on the server.
        
* **toClient** - when set to false the property will never be transmitted to the browser.  This makes the property private in that it will never be accessible on the client.  An example would be a password which can be set on the browser but never be retransmitted to the browser.
        
* **notNull** - when set to true the property may not be saved while it still has a null or undefined value.  
        
* **comment** - when use with a database that supports comments (e.g. Postgres), adds the comment string to the schema.
        
* **logChanges** - debug level logging is done when any data is persisted and properties with **logChanges** set to true are logged.
          
* **fetch** - forces Persistor to fetch the related object (and further descendent objects).  See the Fetch Cascading section in Persist for more details.

### Data Binding Attributes
        
Finally these attributes are used when data binding objects to a view using Bindster. See the Bindster documentation for details of how they are used.
        
* **validate** - a validation expression applied to any form fields to which this property is bound as if it was applied as a b:validate attribute on the form field. 
        
* **format** - a formatting expression applied to any form fields to which this property is bound as if it was applied as a b:format attribute on the form field.
        
* **parse** - a parsing expression applied to any form fields to which this property is bound as if it was applied as a b:parse attribute on the form field.

* **values** - a set of valid values that will automatically fill selects and with the aid of Bindster mappers, radio buttons and checkboxes as well with a set of values.  In the case of selects it is equivalent to using b:fill
         
* **descriptions** - goes with **values** to provide descriptions when codes are used for **values**.  In a select it is equivalent to b:using
          
* **rule** - a set of rules that can be querried by the controller to apply validation, parsing or formatting

**Note:**  A future version of Amorphic will support validation and values when data is synchronized to the server to provide additional security against manipulation of applications via the Javascript console.

## Template Methods

Along with data properties, methods may be freely intermixed.  They are defined in one of two ways:
    
        <property-name>: function () {}
        
Used for code that can be executed either in the browser or on the server.  The function is executed in the same environment as the caller.
    
        <propert-name>: <function-properties>
        
Used when you want to force execution on what environment or the other.  Amorphic will automatically make the call across the wire to the other environment, first synchronizing all objects in the target environment with those in the calling environment.  **<function-properties>** is an object with properties that defines which environment the method will be executed in.  It has these properties:
        
* **on** - defines where the code will reside:
     
* **'client'** - The code lives in the browser.  This is useful if you need to call code in the browser from code that runs on the server.  In general the pattern of performing code that must be execute in the browser **after** calling a method that executes on the server is preferred.
         
* **'server'** - The code lives on the server and is only executed on the server.
         
* **body** - A function that will be executed

* **validate** - A function called in the case where code in the browser calls code in the server and the browser code wishes to validate the input before making the call and throw an error or otherwise use the data binding mechanism's error handling     

## Extending Templates

Supertype supports classical inheritance by using:

    var <extended-template-name> = <base-template-name>.>.create(<create-params>,  <propertiesAndMethods>)
    
* **extended-template-name** is the name of the new template to be created based on the **base-template-name**
* **create-params** is the same as for <object-template>.create and generally is just **extended-template-name**.
* **propertiesAndMethods** is the same as for <object-template.create> and is used to define additional properties or to override attributes of existing properties.  It can also provide additional methods or override existing methods. 

It should be noted that templates are standard Javascript functions created using the function prototype and as such support the **instanceof** operator such that:
 
    var Foo = objectTemplate.create("Foo", {});
    var ExtendedFoo = Foo.extend("ExtendedFoo", {});
    ExtendedFoo instanceof Foo // would be true

## Composing Templates

There are generally two ways to compose (e.g. combine templates):

* **mixin** - Include data properties and methods from another template.  This allows you to group properties by function and then assemble them into larger templates.  The consuming code need not know the origin of the properties.

* **references** - Include a reference to other templates in the type attribute of a data property.  In this case references to the properties of referenced templates always use the property name of the reference (e.g. customer.policy.policyNumber).  At present references are implemented by the persistence layer as database references though they use joins and/or sub-documents to maximize performance

You can mixin either entire templates:

    <template-name>.mixin(<template-name>)
    
or individual properties:
    
    <template-name>.mixin(<propertiesAndMethods>);
    
Example:
    
    var Name = objectTemplate.create("Name", {
        firstName: {type: String},
        middleName: {type: String}
        lastName: {type: String}});
        
    Employee.mixin(Name);
    Customer.mixin(Name);
## Projections
A future version of Amorphic will have support for creating sub-sets of existing templates.  This is useful when you want to have a very efficient way to query only part of table. 

    var PartialPolicy = Policy.project("PartialPolicy", 
        ['policyNumber', 'status', 'workflowState']);
          
The advantage of using project over just simply creating a duplicate template is that you don't have to supply all of the attributes for the properties.    

# Common Object Properties

All objects created from Supertype templates have these common methods:

* **.toJSONString()** - converts the objects and related objects into a JSON string resolving circular references.  It can be restored using <template-name>.fromJSONString(<string>).  This round trip should be used with caution because it generates non-standard object id's that cannot be synchronized between client and server.  There are however certain times when, for example, you want to export and import object data that this can be useful.

* **.\_\_values\_\_(\<property-name>)** - returns the set of values defined as attributes of the property when the template was created.

* **.\_\_descriptions\_\_(\<property-name>)** - returns the set of descriptions as an object with a property for each value as defined in the attributes of the property when the template was created.  For example:
 
       customer.__descriptions__('status')['R']  // Might return 'Registered'
       
* **.\_\_prop\_\_(\<property-name>)** - returns the attributes of the property as defined when it's templates was created.
       
* **.createCopy(\<callback\>) - creates a deep copy of the object giving you precise control of how referenced objects are handled by supplying a callback that is called as each referenced object is to be copied.  The callback determines how the object will be copied.  The callback has these arguments:
  * **1 - \<parent-object>** - the object containing the property to be copied.
  * **2 - \<property-name>** - the name of the property to be copied
  * **3 - \<template>** - the template to be cloned from the **of** or **type** attribute of the property.  
  
  The callback may return:
  * **undefined** - do not clone the referenced object and set the reference to null.
  * **\<object>** - returning a templated object means that that object will be used rather than cloning.  The properties of the returned object will continue to be copied.
  * **\[object]** - returning an array with one object in the first position indicates that the object will be used rather than cloning but that the properties of the object will not be traversed and copied.  This is useful when you want to connect the reference to an existing object.
  * **null** - clone the referenced object and copy into it the properties of the referenced object 
