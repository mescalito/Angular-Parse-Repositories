# ParseRepositories - An Angular service to create repositories for Parse objects.

This package provides a quick and easy way to create repository patterened API objects to [Parse.com](https://www.parse.com) classes inside an AngularJS app. The main features include:

- Customizable CRUD methods for interacting with a class.
- Column name to POJO (Plain Old JavaScript Objects) property translation.
- Various hooks where code can be injected into the execution of requests to Parse's APIs.

Parse's JS SDK works well for contacting and interacting with the server, but it's not very good for interacting with Angular. Parse promises don't resolve the Angular digest cycle and often require several approaches that aren't considered good practices. This library fixes that by giving a standardized API for any services you create to work with Parse, and abstracts away http requesting.



[TOC]



## Installation

You can either download code directly from this repository or pull the service into your probject using bower.

    bower install angular-parse-repositories

To use the repository service in your angular project, make sure to include the file and require project "parse" as a dependency.

```js
angular.module('app', ['parse']);
```

Next, you will need to include the Parse SDK, which is not an Angular library, somewhere before you initialize your Angular app. To create repositories, you first need to initialize the Parse SDK authentication. To do that, call the **parseRepositoriesProvider.init** method in your app configuration.
 
```js
var app = angular.module('app', ['parse'])
.config(['parseRepositoriesProvider', function(provider) {
   provider.init(Parse, 'applicationKey', 'javascriptKey', 'optionalUserToken');
}]);
```

There are three required arguments:
- The global Parse object. It's required here in order to be minification safe.
- The Parse provided application authentication key.
- The Parse provided javascript authentication key.
- (optional) A user's token to [become](https://parse.com/docs/js_guide#users-become), if needed.

## Configuration

To use the service, create a service that requires **parseRepositories** as a dependency. Two methods are made available to you; *CreateRepository()* and *GettersAndSetters*.

```js
app.factory('someService', ['parseRepositories', function(repos) {
   var Class = repos.CreateRepository('className', {});
   
   repos.GettersAndSetters(Class, [
        {angular:'column1', parse:'column1'},
        {angular:'column2', parse:'column2'}
        // Any others
   ]);
   
   return Class;
}]);
```
 
## CreateRepository

```js
repos.CreateRepository('className', {});
```

**CreateRepository()** attaches an object to a Parse class. In most cases, you can think of classes on Parse as a database table on which queries can be made. Two arguments are required; the name of the class on the Parse account to use, and an options object.
 
By default, **CreateRepository()** returns an object with the following methods available on the class specified.

Method Name       | Arguments         | Returns Promise? | Resolves With
----------------- | ----------------- | ---------------- | --------------------------
`all()`           | none              | yes              | array of objects
`count()`         | none              | yes              | inter
`get(objectId)`   | id of object      | yes              | object
`create()`        | none              | no               | new object of class
`save(object)`    | object to save    | yes              | pointer to object provided
`delete(object)`  | object to delete  | yes              | pointer to object provided

Once a repository is created, any method can be added to or taken away from it.

```js
var Employees = repos.CreateRepository('Employees', {});

// Remove a method
delete Employees.delete;

// Add a method
Employees.someNewFunction = function() {
    // code here
};
```

### Customizing Queries

If you would like to replace the queries that are used on any of the methods provided, you can pass an array of query arguments as strings to the options object, on the method name property. The query object is always named 'query'. See Parse's [querying documentation](https://parse.com/docs/js_guide#queries) for details.

```js
var Class = repos.CreateRepository('className', {
    'all':{
        'queries':[
            'query.exists("activationDate");',
            'query.limit(1000);',
            'query.exists("role");'
        ]
    },
    'count':{
        'queries':[
            'query.exists("activationDate");',
            'query.limit(1000);'
        ]
    }
});
```

Customized queries are supported on `all()`, `get()`, and `count()`.

### Custom Methods

Be aware that all methods that use Parse promises wrap resolution with Angular's **$q** library. If you want to make a call to Parse, you'll want to copy this format.

```js
// Creating a new method that calls to Parse
app.factory('someService', ['parseRepositories', '$q', function(repos, $q) {
    // ... Set up code
    
    Employees.someNewFunction = function() {
        var defer = $q.defer();
        
        // Any querying
        var query = new Parse.Query('className');
        query.ascending('objectId');
        
        query.find({
            success:function(result) {
                defer.resolve(result); // resolve the Angular promise
            },
            error:function(e) {
                defer.reject(e); // reject the Angular promise
            }
        });
        
        return defer.promise;
    };
    
    // ...
    
    return Employees;
}]);
```

### Hooks

**CreateRepository()** allows the use of four different hooks to either inject logic or accomplish other goals. The following hook methods are available out of the box.

Hook Name        | Used In    | Accepts    | Is Passed
---------------- | ---------- | ---------- | ------------------------------
`beforeSave()`   | `save()`   | function   | The object to be saved
`afterSave()`    | `save()`   | function   | The object saved
`beforeDelete()` | `delete()` | function   | The object to be deleted
`afterDelete()`  | `delete()` | function   | The pointer to the deleted obj

These methods are very similar to the [cloud code methods](https://parse.com/docs/cloud_code_guide#functions-onsave) Parse provides. However, we understand there may be occasions where you don't want these specific hooks to be run globally on every object that hits the server. One popular reason for using these hooks is to tie into a user activity log or object relationship handling.

```js
var Class = repos.CreateRepository('className', {
    'save': {
        'beforeSave': function(obj) {
           // Format phone numbers to (###) ###-####
           var s = obj.phone.replace(/\D/g, '');
           obj.phone = '(' + s.substring(0, 3) + ') ' + s.substring(3, 3) + '-' + s.substring(6);
        }
    }
});
```

### Soft Deleting

Repositories support 'soft deleting' entities. When an object is soft deleted, the record is not removed from the table on Parse, but instead a designated column - for example, a deletedAt column - is given the current date to show the object has been "deleted". This allows queries to ignore any records that have been soft deleted, but the record can also be referenced later.

To use soft deleting, include the boolean true on the delete object and specify the 'softDelete' column.

```js
var Class = repos.CreateRepository('className', {
    'delete':{
        'softDelete':true,
        'softDeleteColumn':'deletedAt'
    }
});
```

Specifying an entity as soft deleted also causes the `all()` and `count()` methods to ignore any records with a value in the specified column.

## GettersAndSetters

Parse objects by default use explicit getters and setters while Angular uses implicit getters and setters.

```js
// Parse getters and setters
obj.set('name', 'JohnDoe');
alert(obj.get('name'));

// Angular getters and setters
obj.name = 'JohnDoe';
alert(obj.name);
```

This can be a real problem when you're working in Angular specific code, such as view templates, or if you just want to write abstracted code. To overcome this, you can pass an array of translation objects to the **GettersAndSetters** method. Each object should have an `angular` property and a `parse` property. The `angular` property defines the property that will be used on objects returned or created from the repository while the `parse` property defines the column that will be used on Parse's server. Notice that they do not have to be named the same thing. This also makes a really nice separation of concerns, so that if the names of properties on either the front end or back end, it does not have to affect the other.

```js
var Class = repos.CreateRepository('ClassName', {});

repos.GettersAndSetters(Class, [
    {angular:'name', parse:'name'},
    {angular:'email', parse:'email'},
    {angular:'startDate', parse:'dateOfHire'}
]);
```

Objects returned from a repo with **GettersAndSetters** used can be treated exactly like POJOs (Plain Old JavaScript Objects).

```js
app.controller('someCtrl', ['$scope', 'someService', function($scope, service) {
    $scope.object = [];
    
    // Initialization
    service.get(someId).then(
        function(result) {
            $scope.object = result;
        }
    );
    
    $scope.someFunctionCalledFromView = function() {
        $scope.object.startDate = new Date(); // Implicit setting and getting.
        
        service.save($scope.object).then(
            function(result) {
                // handle success
            },
            function(e) {
                // handle error
            }
        );
    };
}]);
```

## Example

Check out the index.html file in this repository to see an example usage.