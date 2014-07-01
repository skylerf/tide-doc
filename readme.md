

#Introduction

[Tide](https://github.com/tide-framework/tide) is a web framework that allows 
seemless communication between [Amber](http://amber-lang.net) and 
[Pharo](http://pharo-project.org)\. 

Tide exposes information using the `JSON` format\. The `JSON` is built from 
Pharo objects and sent through the network to Amber\. The `JSON` contains data 
exposed from objects in Pharo, but can also contain callback information 
to perform actions from Amber to Pharo objects\. Having both data and actions 
sent to Amber makes Tide a very good communication protocol between the two 
Smalltalks\.

To make communication as natural as possible, the Amber layer of Tide uses 
promises to keep the flow of callback calls sequential\.

This documentation aims to teach how to install and use Tide through examples, 
as well as its architecture\.



# Installing Tide



##1\.  Prerequisites

Tide requires several libraries\. It of course depends on Pharo and Amber\. Amber
itself requires `nodejs` and `bower` to install its own dependencies\. The 
Pharo\-side of Tide requires Zinc, which is part of the default image since 
Pharo 2\.0\. Tide however has only been tested with Pharo 3\.0\.



###1\.1\.  NodeJs

Go to [nodejs\.org](http://nodejs.org) and install `node` \(and `npm`\) for your
platform\.



###1\.2\.  Bower

Bower is a package manager for the web, and Amber uses Bower to manage 
dependencies\. The simplest way to use bower is to install it globally as 
follows:




    $ npm install -g bower





###1\.3\.  Pharo

Tide requires Pharo 3\.0\. The simplest way to install it is to evaluate the 
following:


&nbsp;



    $ curl get.pharo.org/30+vm | bash



To start the Pharo image, evaluate:


&nbsp;



    $ ./pharo-ui Pharo.image





####1\.3\.1\.  Preparing the Pharo image

Once you get the Pharo window open, you have to install the Tide backend part\. 
This means bringing the Pharo code you cloned from GitHub into the Pharo image\.


&nbsp;


-  Click on the background of the Pharo window
-  In the World menu that appears, click on `Workspace`
-  In that window, evaluate: \(you type the thing, select the text and then right 
  click and select "Do It" from the menu\)\.


&nbsp;



    Metacello new
      configuration: 'Tide';
      version: #development;
      repository: 'http://www.smalltalkhub.com/mc/Pharo/MetaRepoForPharo30/main';
      load.



When this is finished, evaluate:


&nbsp;



    TDDispatcher tideIndexPageUrl inspect



When first executed, you will get an error saying you must execute bower 
install in a particular directory\. Open a terminal, change to the right 
directory, and execute:


&nbsp;



    $ bower install



Back in the Pharo window, close the error message and evaluate the same instruction 
again:


&nbsp;



    TDDispatcher tideIndexPageUrl inspect



This should give you the URL at which your web browser should be pointed to\. 
Now copy this URL, open your web browser and paste it in the browser's address bar\.





###1\.4\.  Starting the server

The `TDServer` class provides a convenient way to start/stop a Tide server, using
Zinc behind the scenes:


&nbsp;



    TDServer startOn: 5000. "Start the server on port 5000"
    TDServer stop. "Stop any running server"





# A first example: the traditional counter

In order to get started with Tide, we will implement the traditional counter example\.

Tide already includes such an example in the `Tide-Examples` package that you can 
refer to\.



##2\.  The presenter

A counter application should contain two buttons, one to increase and the other one to
decrease a count\. It should also display the count value to the user\.

While this application might seem extremely simplistic, it already shows some of the 
core principles behing Tide: Presenters and Proxies\.

We start by creating the `MyCounter` class in Pharo by subclassing `TDPResenter`\.


&nbsp;



    TDPresenter subclass: #MyCounter
    	instanceVariableNames: 'count'
    	classVariableNames: ''
    	category: 'MyCounter'



Note that not all "exposed" objects have to be subclasses of `TDPresenter`\. As we will
see later, any object can be exposed to Amber using a `TDModelPresenter` instance
on the domain object\.

Our class has one instance variable `count`, that we initialize to `0`:


&nbsp;



    MyCounter >> initialize
        count := 0



To display the count value to the user, we will need to expose `count` using an accessor\.
We also add two methods to increase and decrease our counter:


&nbsp;



    MyCounter >> count
        ^ count
    
    MyCounter >> increase
        count := count + 1
    
    MyCounter >> decrease
        count := count - 1



The final step we need to add the our counter is pragmas\. Pragmas are 
metadata on methods\. Tide uses pragmas to expose data \(called state in Tide\) 
and callbacks \(called actions\) to Amber\. Here's our final version of the 
counter class:


&nbsp;



    MyCounter >> count
        <state>
        ^ count
    
    MyCounter >> increase
        <action>
        count := count + 1
    
    MyCounter >> increase
        <action>
        count := count - 1





##3\.  Registering applications with handlers

We now have to create an entry point with our counter presenter in the Tide server\.
To register the entry point, evaluate:


&nbsp;



    MyCounter registerAt: 'my-counter'.



We can deduce two points from the preceding evaluation:


&nbsp;


-  Presenter classes are registered as handlers, not instances\. Tide will create "per session" instances of the registered class meaning that presenters are not share between user sessions\.
-  The entry point will have a `handler` associated with a fixed entry point  url `'/my-counter'`\. When someone will query that registered url, the presenter will generate `JSON` data corresponding to its state and actions, and the handler to send it back in a response to the request\.

If we perform a request at `http://localhost:5000/my-counter`, we get the following 
`JSON` data back:


&nbsp;



    {
      "__id__":"bwv8m74bhgzmv0dgvzptuy4py",
      "actions":{
        "increase":"/my-counter?_callback=359446426",
        "decrease":"/my-counter?_callback=523483752"
      },
      "state":{
        "count":0
      }
    }





##4\.  The Amber application

The next step in our example is to create the Amber\-side of this counter application\.
We will use Amber to render an HTML view of our counter, and perform actions using proxies
back to the counter defined in Pharo\.



###4\.1\.  The client\-side API

On the client\-side, root presenters exposed as handler can be accessed by creating proxies:


&nbsp;



    myProxy := TDClientProxy on: '/my-counter'.



Message sent to proxies will be resolved using its **state** and **actions\+** as defined on 
the server\-side\.

Calls to state methods are resolved locally and synchronously, because the state is passed
over to Amber as we previously say in the JSON data\.

Calls to action methods perform requests that will result in performing the corresponding
method on the pharo object asynchronously\. Once the action is performed, the proxy will
be automatically updated with possible new state and actions\.

Since action calls are not synchronous, Tide proxies have a special method `then:` used
to perform actions only when and if the action is resolved and the proxy updated\.


&nbsp;



    "synchronous state call"
    myProxy count. "=> 0"
    
    "async action call"
    myProxy increase; then: [
        myProxy count "=> 1" ]





###4\.2\.  The widget class

In Amber's IDE, create a new class `MyCounterWidget`\. 


&nbsp;



    Widget subclass: #MyCounterWidget
    	instanceVariableNames: 'counter header'
    	package: 'Tide-Amber-Examples'



The widget class has two instance variables: `counter`, which is hold a proxy over the 
Pharo counter object, and `header` which will hold a reference on the header tag brush to
update the UI\.

To initialize our counter widget, we connect it to the Pharo counter presenter as follows:


&nbsp;



    initialize
        super initialize.
        counter := TDClientProxy on: '/my-counter'



Note that `'/my-counter'` is the path to the server\-side handler for our counter presenter\.

We can now create the rendering methods\.


&nbsp;



    render
        counter connect then: [
            self appendToJQuery: 'body' asJQuery ]
    
    
    renderOn: html
    	header := html h1 with: counter count asString.
    	html button 
    		with: '++';
    		onClick: [ self increase ].
    	html button 
    		with: '--';
    		onClick: [ self decrease ]
    
    update
    	header contents: [ :html |
    		html with: counter count asString ]



The `render` method waits for the counter to be connected, then appends the widget to the
`body` element of the page \(using the `renderOn:` method\)\.

`renderOn:` is a typical widget rendering method using the builtin Amber `HTMLCanvas`\.
The `count` message send to the `counter` proxy will be resolved as a state accessor as
defined on the server\-side\.

Finally instead of updating the entire HTML contents of the counter, `update` will only 
update the relevant part, the header\.

We still miss two methods to actually increase and decrease our counter:


&nbsp;



    increase
    	self counter increase.
    	self counter then: [ self update ]
    
    decrease
    	self counter decrease.
    	self counter then: [ self update ]



<a name=""></a>![](images/tide-counter.png "file://images/tide-counter.png")



# More on actions



# Managing sessions



# Handlers


&nbsp;
<p class="todo">should it be there already? It seems too early to talk about that, but I need to introduce the concept in order to talk about the file upload handler\.</p>


# Creating custom Presenters



# Managing file uploads

Managing file uploads in the context of a flat\-client application can be cumbersome\. 
The reason is that file uploads with the HTTP protocols were not made for asynchronous 
uploads\. Tide tries to solve this problem by abstracting away the implementation details 
of an AJAX\-friendly file upload with the `TDFileHandler` class\.



##5\.  Creating file upload entry points




# Handling exceptions



# A more advanced example: <<FIND SOMETHING>>

