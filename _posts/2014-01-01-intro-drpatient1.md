---
layout: page
title: "Tutorial"
category: intro
date: 2014-01-01 00:00:00
order: 1
---

## The Appointment Scheduling Application

This is simple application that demonstrates

* Simple data model with various relationships between objects such as
    * One-to-One
    * One-to-Many
* Seamless synchronization between the browser and the server
* Data binding from the model to the HTML view
* A simple controller to orchestrate it all  
  
## How to set it up

* Clone or checkout this project from github

        https://github.com/selsamman/amorphic-drpatient-demo.git
    
* Make sure you have nodejs installed (version 4 or later)
    
* Run NPM to get the required node modules

        npm update
        
* Start the application

        nodejs app.js --port 30001

* Go to the local web page in your browser

        http://localhost:30001
    
* And you should see    

![Model, View, Controller](../img/drpatient.png)
    
  
## Put it through it's paces

* Enter the name of a doctor and hit enter

* Enter the name of a second doctor and hit enter

* Enter the name of a patient and hit enter

* Enter the name of a second patient and hit enter

* Click on one of the doctors to select it

* Select the patient in the drop-down on the right

* Enter a date (dd/mm/yy) and hit enter

* You will now see an appointment in the list on the right

* Refresh the browser window (or close it and reopen the same URL).  You will see that all your data was saved.

## What just happened?

### index.html

Let's first have a look at **apps/drpatient/public/index.html**.  It contains:

* Standard scripts needed. Underscore and Q are needed by the Amorphic software.  Bootstrap is optional but is used in this simple demo.


        <!-- standard software scripts --->
        <script src="../../common/js/lib/underscore.js"></script>
        <script src="/modules/q/q.js?ver=0q.js?ver=0"></script>
        <script src="../../common/js/lib/jquery.js"></script>
        <script src="../../common/js/lib/bootstrap.min.js"></script>
        <link href="../../common/css/bootstrap.min.css" rel="stylesheet">

    
* Amorphic software, including the data binding layer - Bindster, the type system - supertype, the browser server synchronization layer and the amorphic client
    
        <!-- amorphic static scripts -->
        <script src="/bindster/index.js?ver=0"></script>
        <script src="/supertype/index.js?ver=0"></script>
        <script src="/semotus/index.js?ver=0"></script>
        <script src="/amorphic/client.js?ver=0"></script>
    
* The dynamically generated script which will set everything up, make sure things pick up where they left off if needed
    
    <!-- Dynamically generated script that will instatiate the controller -->
    <script src="/amorphic/init/drpatient.js?ver=0"></script>

* Then you have the body of the HTML which we will com back to

* And finally a script that gets the application started

        <!-- Bind view to amorphic -->
        <script src="/bindster/bindster-amorphic.js"></script>

So let's follow the trail .....

### Data Binding

* When you entered the name of the doctor this HTML was executedd

        <input b:bind="doctorName" class="form-control" placeholder="Name" b:onenter="addDoctor()"/>

    Notice the b: attributes on the input form.  The data binding layer, Bindster has a b: namespace for attributes that it knows about.  Notably:
    
        b:bind="doctorName" 
    
     Binds the input field to a property in the controller object called doctorName.  When you type into the field and lose focus, that property will be set to the value in the form field and conversely if the doctorName property is updated by Javascript, the form field will be populated.  Here try it by going to the Javascript console in the browser and typing
       
            > controller.doctorName = 'Foo';
            > controller.refresh()
        
     Now you will see Foo as the doctor name.  Type 'Bar' in the doctorName field and hit tab.  Then type this in the browser console:
            
            > controller.doctorName
              "Bar"
        
     You have just witnessed two-way data binding

### The Controller   

The input field also has an event declared

        b:onenter="addDoctor()" 
    
This causes the addDoctor() method in the controller to be executed in the controller which is contained in  **apps/drpatient/public/js/controller.js**. 
     
        module.exports.controller = function (objectTemplate, uses)
        {
            var BaseController = uses('./baseController.js', "BaseController");
            var Doctor = uses("model.js", "Doctor");
            var Patient = uses("model.js", "Patient");
            var Appointment = uses("model.js", "Appointment");
            Controller = BaseController.extend("Controller", {
                ....
            }
            
        }    

All amorphic application files are organized into templates.  Templates are defined in files that export an initialization function that Amorphic wil call to craete the templates. The property exported (controller in this case) **must be named the same as the file itself**.  Two parameters are passed in the intialization function:

* **objectTemplate** used to create new templates (we will se that in action later)
    
* **uses** a function to declare usage of other templates.  **uses** is passed the file name of the template and the template name itself
    
    
In this case we declare our usage of a stock base controller and the various templates that consitute the principal entities for our demo which is known as the "model".  Finally we create our **Controller** template by extending the base controller.  This gives our controller all of the properties and methods of the base controller and let's us add our own properties and methods.  A few of these we just excercised in our demo:
    
   doctorName:     {type: String},
     
Defines the doctorName property as bein a **String** type
     
    doctors:        {type: Array, of: Doctor, value: []},
      
Defines an **Array** of **Doctor** objects as defined in the **Doctor** template.  Importantly we initialized the value to that of an empty array **[]**

### Controller Events     
     
Within the controller there is the code that was executed from the **b:onenter**  
     
    addDoctor:  {on: "server", body: function () {
         var doctor = new Doctor(this.doctorName)
         this.doctors.push(doctor)
         this.doctorName = "";
     }},

 The **addDoctor** is defined to be a function that lives on the **server**.  This means that when you hit enter the following sequence of events happened:
     
* All of the data in the browser (notably the value in the doctorName field you just updated is synchronized with the server

* Your addDoctor function is execute on the server.  It creates a new Doctor object and and adds it to the doctors array.

* All of the changes to data on the server (notably the new element in the doctors array and the clearing of the doctorName field) is now synchronized with the browser as you return from the addDoctorFunction

* Amorphic automatically renders the screen (like when you typed controller.refresh()) and the new doctor name appears in the list above the input field
     
That is a lot to happen from this one little function definition but this is the essence of what Amorphic does for you - it let's you execute code on the server within the same file that contains code executed in the browser.  It keeps everything in sync to allow this all to happen in a seamless fashion.  Note that in this case we didn't have to declare our function as living on the server.  It could just as well have been on the browser.  However - if we did not make a call to the server, the data would never have been saved on the server and when we hit refresh it would have cleared everything.  

### The Model    
    
Let's have a look at the templates for the principal entitie, Doctor, Patient and Appointment whih constitute the model:
    
    module.exports.model = function (objectTemplate, uses)
    {
        var Doctor = uses("model.js", "Doctor");
        var Patient = uses("model.js", "Patient");
        var Appointment = uses("model.js", "Appointment");
    
        objectTemplate.create("Doctor", {
            name:           {type: String},
            appointments:   {type: Array, of: Appointment, value: []},
            init: function (name) {this.name = name}
        });
    
        objectTemplate.create("Patient", {
            name:           {type: String},
            appointments:   {type: Array, of: Appointment, value: []},
            init: function (name) {this.name = name}
        });
    
        objectTemplate.create("Appointment", {
            doctor:         {type: Doctor},
            patient:        {type: Patient},
            when:           {type: Date},
            init: function(when, doctor, patient) {
                this.when = when;
                this.doctor = doctor;
                this.patient = patient;
                doctor.appointments.push(this)
                patient.appointments.push(this);
            }
        });
    }

As usual the template file starts with the intializer function, this time exporting model which corresponds to the name of the model file, model.js.  It declares it's usage of the three entities.  By declaring them at the top of the file, these templates were immediately available for use.  This way in the Patient template when a reference is made to the Appointment template
    
    appointments:   {type: Array, of: Appointment, value: []},

The Appointment template has already been declared.  Otherwise it would be unresolved.    So we see here how the objectTemplate parameter is used.  It is used when you want to create a brand new template.  We also see that the templates bascially define a standard Javascript function that can be used to create an object using the **new** operator as we saw in the addDoctor function:
 
     var doctor = new Doctor(this.doctorName)
  
Amorphic uses this form of "classical inheritence" rather than prototypal inheritance (the createObject pattern).  

### Relationships between Objects

Also note the **init** function in the tmeplate definitions.  When you execute the **new** operator on a template the **init** function is executed and you can put any code needed to setup the property values there.  Let's look at the **init** function for appointment:
  
     init: function(when, doctor, patient) {
          this.when = when;
          this.doctor = doctor;
          this.patient = patient;
          doctor.appointments.push(this)
          patient.appointments.push(this);
      }
                  
This **init** function takes three parameters which are all needed to seup an new appointment.  **when** is a time which is just assigned to the when property but **doctor** and **patient** are objects that are assigned to their respective properties.  Note that to setup the appointment, the new appointment is pushed into the appointment arrays of both the doctor and the patient.  Here is the code in the controller that is executed when you added the appointment:
   
       addAppointment:  {on: "server", body: function () {
           (new Appointment(this.appTime, this.appDoctor, this.appPatient));
           this.appTime = null;
       }}

This code (which lives on the server), simply creates a new **Appointment** object.  It doesn't need to do anything with it once it is created since the **init** function, as we just saw, added the appointement to both the **Doctor** and the **Patient**.  But how did we get **this.appDoctor** and **this.appPatient**?  Of course they are properties of the controller
  
      // New appointment properties
      appDoctor:      {type: Doctor},
      appPatient:     {type: Patient},
      appTime:        {type: Date},

But how did their values get populated?
    
* **appTime** is simply bound using a **formatter** and **parser** defined in the baseController that deal with dates and times.  Formatters are methods in the controller that are called to format data before transferring it to the form control and parsers are methods in the controller that process data from the control before it is moved to the data property.
  
      <input b:bind="appTime" class="form-control" b:onenter="addAppointment()"
             placeholder="Appointment Time" b:format="formatDateTime()" b:parse="parseDate()"/>

* **appDoctor** represents the currently selected doctor (the one you click on in the list).  Here is the HTML that generates the list of doctors and selects the current one 
    
        <b:iterate on="doctors" with="doctor">
                <p class="bg-success"><a href="#" b:bind="doctor.name" b:onclick="appDoctor = doctor"></a></p>
        </b:iterate>
 
    The **\<b:iterate\>** tag will cause the enclosed code to be generated a number times based on the **on** attribute which in this case refers to the array of doctors.  On each iteration the **with** property causes a property to be created in the controller that is populated with the value of the specific instance of the doctor array for that iteration.  What that means is that when the **b:onclick** code is executed **doctor** set set to that specific instance before executing the code so in this case it will set the **appdoctor** property in the controller to the currently selected **doctor**
     
* **appPatient** is selected using a drop-down
     
        <select b:bind="appPatient" class="form-control" b:fill="patients"
                 b:fill-key="value._\\_id_\\_" b:fill-value="value.name"></select>
     
     The data binding layer has special handling for selects:
     
   * It will "fill" the values based on the **b:fill** attribute.  In this cases it points to the array of patients in the controller.
   * Since this is an array of objects you need to specify the particular property of that object which will be used as the select option keys  (value attribute of the option tag).  This is done with the **b:fill-key** attribute which in this case is set to the **__id__**.  The **\\** characters are used to escape handling of the **__xxx__** pattern which would otheriwse have special meaning.  Every object has a property **__id__** which uniquely identifies it so the list of options will have unique values.
    * Similarly you also need to specify the text of the option tag with the **fill-value** attribute.  In this case it is the **name** property of the patient object.
    * Normally when you select a value the data binding layer will simply populate the bound field (in this case **appPatient** with the selected key (value attribute of the option tag).  However when the value is **__id__** the data binding layer wills substitute the actual object.  So selecting the patient in the drop-down actually populates **appDoctor** with the object that represents that doctor.
           