# iOS Best Practices

This guide outlines the best practices I've collected working on various teams/projects. 

## Introduction

## Table of Contents
* [Style](#style)
* [General Principles](#general-principles)
* [Object Collaboration](#object-collaboration)
* [Model View Controller](#model-view-controller)

## Style
Our [Style Guide](https://github.com/FlyingJalapeno/Objective-C-Style-Guide) contains information on formatting code and other concrete coding techniques. Because Objective-C is a dynamic language and is intimately intertwined with Cocoa and its conventions, the line between style and best practices tends to be a bit blurry.

## General Principles
Below are some general principles to guide you when architecting your code. 

### Always use the highest abstraction possible
When choosing a technology for implementing behavior, always choose the highest level abstraction that meets your needs. This keeps you from introducing unneeded complexity too soon in your application (YAGNI / Pre-optimization).

Some examples:
AVPlayer vc RemoteIO
Autolayout vs Frame calculation code
NSNotifications vs KVO
NSOperations vc GCD

#### GCD
Although GCD is a lower level abstraction, the API is rather straightforward and quick to implement. This makes GCD / dispatch blocks extremely good for quickly trampolining to and from main thread or as a replacement for "performSelector:afterDelay".  However, if your code begins to require state or order of operation is important, you should move to something more structured like NSOperations.

### Immutability
When possible favor immutability in your objects. In practical terms, this means pass all needed information in during initialization. This reduces the complexity by reducing the number of possible states of your object.  This makes the objects implementations more straightforward to write since you do not have to worry about existence of properties and indeterminate states. 

### Dependency Injection
With immutability comes dependency injection. In it's most basic form, this simply means passing all required parameters on initialization. This has a few effects - one is adding immutability, the other is declaring dependencies in a formal way that can be enforced at build time instead of at run time.  

###  Subclass only if needed
Subclassing is useful for organizing code, but can quickly become rigid and hard to deal within Objective-C codebases. Cocoa encourages composition over subclassing, and expresses this through it's APIs. (Composition over inheritance)

Before subclassing, try the following:
1. Can I use a category instead?
2. Can I separate this into another object and use the delegate pattern to coordinate behavior?

#### View Controllers
Specifically, you should generally avoid creating an base View Controller class that is used throughout your app. Adding behavior in this way can make difficult to make visual changes to views later on. Additionally, view controllers from 3rd party libraries will be unable to inherit your custom behavior, and so you may end up duplicating code anyways. 

Note: This is not proposing that you never create an abstract view controller class, instead keep such view controllers scoped to a specific set of related view controllers. 

##### Use Categories to share UI code.
Building on the category principal, a great way to share code between your view controllers is to encapsulate flows and behaviors in category methods that can be accessed from all view controllers.

By adding a category to hold your logic, view controllers can now "opt in" to the behavior you defined

### What should be public
Only expose properties and methods needed by other classes. 
 
#### Minimal
 You should always strive for publicly exposing the fewest number of properties and methods. This reduces the number entry points into the class which reduces the complexity of how the class can be used.
 
#### Expose all initialization parameters
Any parameters passed in during initialization should be exposed as a public, and if possible, a readonly property. This allows users of your class to differentiate between instances using the arguments passed in when creating them.

### Structure and complex behavior
NSOperations (or something like BFTask in [Bolts](https://github.com/BoltsFramework/Bolts-iOS)) are good abstractions for asynchronous, dependent tasks. (Command Pattern)

These types of classes provide structure to your business rules and workflows and encapsulate that behavior into a concrete object while providing a standard API for progress, cancelation, and errors.

### Global Access
In general global access should be avoided when possible. Sometimes it appropriate - one example is a user session which may be needed by many classes in an app. It can be overkill to pass this as a initializer parameter in every class, so this is a good candidate for global access.

#### Singletons
The most common way to provide global access. Singletons should be used sparingly. Refrain from having any view controller or view singletons - most of the time you can scope the behavior to specific class or a category (add a UIViewController category). Many times a singleton sees like a good idea, but eventually you may need "2" of that object - and many assumptions your code made will be wrong. Singletons also make unit testing much harder.

#### App Delegate
It is common for people to use the App Delegate as a singleton - which in turn leads to the app delegate containing and externalizing objects that are not its responsibility. Many times people do this to get a reference to the entry point of the view(controller) hierarchy - try creating a purpose built object for this instead.

#### NSObject Categories
Adding categories to NSObject usually is a bad idea - since almost everything is an NSObject. Try to better scope your code.

## Object Collaboration
### Types of communication
Depending upon the situation, use an appropriate communication pattern. The standard patterns are below listed in order of increasing complexity.

#### Message
Simply passing a message to an object and invoking a method. This is the most basic form of communication in Cocoa/Objective-C.

#### Target/Action
A form of message passing bound to specific control states. This is only used for views communicating to other views or view controllers. 

#### Blocks 
Invoking a block of code before/after a task. Useful for asynchronous communication. A great way to decouple the message sender from the actual code that is executed. Can also send back "responses" via the block return value. Usually defined in line, so intent can be more clear than using delegation. 

#### Delegation
A more specific form of message sending implemented through the use of protocols and is thus more loosely coupled as the sender does not know the receivers type.  Useful for Informing another object before/after a task, or asking "should" this action take place. "Ok" for asynchronous communication, but with a little more overhead than a block.  that also allows for "responses" via the method return value.

#### Notification Center 
Posting a notification before/after task, useful for notifying multiple objects. Although it is more overhead, it is a more formal way to specify specific events, 

#### KVO 
Fine grained notifications at a property level. This is the most verbose and error prone of the techniques, and in general should be avoided if possible. If it must be used, look into a more resilient 3rd party wrapper like [MAKVONotificationCenter](https://github.com/mikeash/MAKVONotificationCenter) or [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa).

### Communication Situations
Based on the types of objects involved, below is a list of appropriate ways to communicate: 

#### Views
##### View -> View
- Message
- Delegation
- Target/Action

##### View -> View Controller
- Blocks
- Target/Action
- Delegation

##### View -> Model Controller
- NEVER

##### View -> Model
- NEVER

### View Controllers
##### View Controller -> View
- Message

##### View Controller -> View Controller
- Use the Model Layer to communicate changes
- Delegation
- Blocks

##### View Controller -> Model Controller
- Message

##### View Controller -> Model
- Message

### Model Controllers
##### Model Controller -> View
- NEVER

##### Model Controller -> View Controller
- Block
- Notification
- KVO
- Delegation (not preferred for long lived objects that can change delegates)

##### Model Controller -> Model Controller
- Block
- Notification
- KVO
- Delegation (not preferred for long lived objects that can change delegates)

##### Model Controller -> Model
- Message

### Models
#### Model -> View
- NEVER

##### Model -> View Controller
- Notification
- KVO

##### Model -> Model Controller
- Notification
- KVO

##### Model -> Model
- Message
- KVO - maybe, more limey this should be in a Model Controller

## Model View Controller
In general you should follow Apple guidelines and design your apps using MVC principals. Practically that breaks down into the following:

### Models
Make your models "dumb".  Do not place methods on the models that control behavior. Models should reflect state.

They should not contain methods that control behavior. These methods should be contained in controller objects.

Methods on model objects should largely be confined to lazy and stateless data transformation, like returning a date as a string, or a sorted array of its children.

#### Concrete Models vs Generic Collections
When possible you should always use custom subclasses and protocols model objects to represent your data. Using NSDictionaries and NSArrays to pass around data leads to more verbose code (ex. accessing values by key) and ambiguity of data types. Using the language features is leads to more understandable and expressive code.

### Controllers
Controllers come in two broad flavors.

#### View Controllers
The primary purpose for view controllers is:

1. Control the navigation of the application
2. Provide a bridge models and views. They pull in data and push it out to the views in the form they need.

Beyond this, view controllers tend to be a dumping ground for all code without a home. Look for ways to remove code by using collaborating objects. Below are some common behaviors that are included in View Controller implementations that can easily be moved into other files.

##### Datasources
A great way to reduce the amount of code in view controllers is to remove the boilerplate code for table views and collection views. This is a technique covered extensively on [objc.io](http://www.objc.io/issue-1/lighter-view-controllers.html).

##### Layout
Layout code should be contained in Nibs, Storyboards, or custom view classes (or in the case of collection views, layout classes). 

##### Business Logic
Business logic, domain knowledge should be placed where it can be shared among view controllers - use Model Controllers and Models.

#### Model Controllers / Services
Model Controllers should perform model manipulations that are UI independent (like network and processing) exception: user input . Use these to encapsulate business logic, algorithms, and rules. Anything that doesn't require a UI class should be put in these objects so they can be shared in different parts of the code base. This also perpetuates composition over subclassing, i.e. placing behavior in a shared component instead of View Controller abstract superclass.
 
### Views
Views should encapsulate layout code. 

#### Plan for updates and animations
Place code that updates your views based on the model in a separate method in your view controller, like:

```objc
- (void)updateViewAnimated:(BOOL)animated;
```

Call this method whenever you need to update the view. The animated flag gives you the opportunity to animate changes as well. This also plays nicely with lifecycle methods:

```objc
- (void)viewDidAppear:(BOOL)animated{
  [super viewDidAppear:animated];
  [self updateViewAnimated:animated];
}
```

#### Auto Layout
Auto Layout was designed to allow rapid UI development when working with multiple screen sizes and localized content. It also provides a high level abstraction (constraints) to describe layouts. 

Auto Layout should be favored over old style springs and struts autoresizing behavior and frame calculations in code which can be difficult to maintain and understand.

#### Nibs vs Storyboards vs Code
In general use the highest level abstraction you can. In order:

##### Storyboards
Storyboards are great for static workflows. For complex applications, one storyboard is onerous, if not impossible. In that case you can use multiple storyboards, each confined to a specific workflow. Login/Signup flows are great candidates for this. 

If your workflows are more dynamic, then storyboards provide little advantage over a traditional nib (Except: Only Storyboards have access to certain new features like static table views and prototype cells.

##### Nibs
Nibs are great for table view cells and collection view cells, as well as views that do not play nicely with storyboards (You need several top level objects to construct your view).

#### Code
Writing layout code is a necessity when performing animations. Remember that many times you can do your initial layout in a Nib/Storyboard, and modify the views in code at runtime.

If using Nibs/Storyboards with Auto Layout, you will need to expose your constraints as IBOutlets to modify them in code.  

### Minimum Supported OS
If possible, support only 1 OS prior to the current OS, with an eye on dropping support each summer as major OS betas are released.

