# ParseRepositories - An Angular service to create repositories for Parse objects.

This package provides a quick and easy way to create repository patterened API objects to [Parse.com](https://www.parse.com) classes inside an AngularJS app. The main features include:

- Customizable CRUD methods for interacting with a class.
- Column name to POJO (Plain Old JavaScript Objects) property translation.
- Various hooks where code can be injected into the execution of requests to Parse's APIs.

## Installation

You can either download code directly from this repository or pull the service into your probject using bower.

    bower install angular-parse-repositories

To use the repository service in your angular project, make sure to include the file and require project "parse" as a dependency.

```js
angular.module('app', ['parse']);
```

Next, you will need to include the Parse SDK, which is not an Angular library, somewhere before you initialize your Angular app. To create repositories, you first need to initialize the Parse SDK authentication. To do that, call the **parseRepositoriesProvider.init** method in your app configuration.
 
```js
angular.module('app', ['parse'])
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
 
### CreateRepository