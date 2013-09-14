.. link: 
.. description: 
.. tags: code, javascript, angularjs
.. date: 2013-09-14 09:31:00
.. slug: quick-and-dirty-parameter-passing-in-angular-revisited
.. title: Quick and dirty parameter passing in Angular 2; The Angling.
.. description: In which I discover the right way to pass information to controllers instantiated by repeaters in AngularJS.
.. comments: true

*This is a followup to an* |link|_

.. _link: ../08/quick-and-dirty-parameter-passing-in-angular.html

.. |link| replace:: *earlier post.*

After pushing that hack out to the rest of my team, I felt some modicum of pride - I had, through understanding little-used parts (at least, publicly) of a `poorly-documented framework <http://docs.angularjs.org/api/ng.$exceptionHandler>`_, managed to reason out a nice solution.

Days passed and the codebase expanded.  One day a teammate approached me and said, "Hey, you know that trick you used to pass the ID into the controller?" "Yes," I replied cautiously.  "You didn't need it."

.. figure:: http://i.imgur.com/5HbrwOw.jpg
 :alt: Insert "Duck typing" joke here.
 :target: http://imgur.com/5HbrwOw

 Wat.

As a refresher, here's what I had ended up implementing:
    
.. code:: html
  
    <div data-ng-controller="ParentController">
        <div data-ng-repeat="childGuid in childrenGuids" data-ng-controller="ChildController" data-id="{{childGuid}}"/>
    </div>
    
... which had some companion JavaScript in the repeated controller that read the value of :code:`data-id` when the view was updated.  There was no other way to pass data in to the :code:`ChildController`, right?  Wrong.  Let's step back for a second.

Scope in Angular is `special <https://www.google.com/search?q=snowflake&tbm=isch>`_. While the framework goes out of its way to make it seem like things are tightly bound to the encompassing controller and that the DOM merely exists for presentation purposes, nothing (especially in web development) is ever that cut and dry.

Quoth the Angular documentation:

    Scopes are arranged in hierarchical structure which mimic the DOM structure of the application.
    
and...

    The Model is scope properties; scopes are attached to DOM where scope properties are accessed through bindings.
    
As my coworker reminded me, the scope referenced within the :code:`ChildController` isn't bound to the controller, it's bound to the DOM. And, in the case of the repeater, the scope on the repeater node includes the :code:`childGuid` property.

What the code looks like now...

.. code:: html
  
    <div data-ng-controller="ParentController">
        <div data-ng-repeat="childGuid in childrenGuids" data-ng-controller="ChildController"/>
    </div>

.. code:: javascript

    var ChildController = function() {        
        $scope.id = $scope.childGuid;
        
        var init = function() {
            // do initialization
        };
        
        init();
    };

As you can see, on the second line of the controller, I'm directly referencing the repeated element.  You don't need to re-assign it like I'm doing in this example, by the way, calling it directly won't/shouldn't present any issues.

An important caveat: if you think the controller might get used elsewhere, it would be beneficial to guard against instantiation without the required arguments...

.. code:: javascript

    var ChildController = function($exceptionHandler) {        
        $scope.id = '';
        
        var init = function() {
            if (typeof $scope.childGuid === "undefined") {
                $exceptionHandler("The ChildController must be initialized with a childGuid in scope");
            }
            $scope.id = $scope.childGuid;
            // continue iniitialization
        };
        
        init();
    };
    
And with that my hack was dead. This was a good thing as hacks are, by nature, smells that should be minimized.

The moral of the story?  Write self-congratulatory blog posts about slightly-hacky solutions.  Someone will submit a patch.