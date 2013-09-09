<h1>auto-crud</h1>

<h2>Introduction</h2>
auto-crud is an abstraction layer for generating REST routes to perform basic CRUD operations.  It glues together
express, mongo and json-schema validation.  The end goal is to define CRUD APIs in a declarative and uniform way
instead of writing route handlers directly.

<b>Also available through NPM: </b> `npm install auto-crud`

<h2>Defining an API</h2>
The auto-crud module exports a single function which takes a configuration object as its only input.

```javascript
autocrud({
    app: app,                   // The express application object.
    collection: mongo.hoosit,   // The collection object generated by the mongo driver.
    name: 'hoosit',             // The name of the object (this will be appended to the end of path).
    path: '/api',               // The root URL path that this API should be generated at.
    schema: {                   // The json schema that should be used for validation
        type: 'object',
        properties: {
            name: {type: 'string', required: true},
            description: {type: 'string'},
            rating: {type: 'integer'},
            comments: {type: 'array', items: {type: 'string'}}
        },
        additionalProperties: false
    }
});
```

<h3>Configuration object parameters</h3>
* `app` The express application object.  This is used to generate REST routes for GET, POST, PUT and DELETE.
* `collection` The mongo collection object.  This is where the API will store its data.  NOTE: this tool will not ensure indexes.
* `name` The name of this API's objects, this will also be appended to the end of the path param when generating routes.
* `path` The root path for the routes generated by this API.
* `schema` The json-schema that will be used to validate objects given to the API by POST and PUT.

<h3>Optional parameters</h3>
* `getCreate` If true a get and get by id method will be registered with express.
* `postCreate` If true a post method will be registered with express.
* `putCreate` If true a put by id method will be registered with express.
* `deleteCreate` If true a delete by id method will be registered with express.

NOTE: The Autocrud object will have all the http methods callable from other objects in the server, regardless of being
registered with express.

<h3>Input Transformation</h3>
It is often that input data from a client application differs from how you want to store data in your database.  So,
auto-crud allows you to specify transformation functions, which will be called after the input object has been
validated, but before a database insert or update.

```javascript
autocrud({
    ... // Default options
    postTransform: function (body) {
        if (!body.rating) body.rating = 1;
    }
});
```

Transform functions take a single parameter, which is the message body after it has been validated.

* `defaultTransform` If specified, this transform function will be applied to both POST and PUT operations.
* `postTransform` If specified, this function will be used for POST operations, overriding the defaultTransform.
* `putTransform` If specified, this function will be used for PUT operations, overriding the defaultTransform.

<h3>Route Authentication</h3>
No API would be complete without having an authentication mechanism.  This is supported by adding an express middleware
function to the configuration object.  This allows auto-crud to be agnostic about how users are authenticated.  I tend
to prefer passport, so that is what is shown in the example.

```javascript
autocrud({
    ... // Default options
    defaultAuthentication: function (req, res, next) {
        if (req.isAuthenticated() && _.contains(req.user.roles, 'administrator')) next();
        else res.send(401, 'Unauthenticated');
    }
});
```

* `defaultAuthentication` If specified, this middleware will by applied to all HTTP methods on the auto-curd route.
* `getAuthentication` If specified, this middleware will be applied to the GET HTTP method.  Overrides default.
* `postAuthentication` If specified, this middleware will be applied to the POST HTTP method.  Overrides default.
* `putAuthentication` If specified, this middleware will be applied to the PUT HTTP method.  Overrides default.
* `deleteAuthentication` If specified, this middleware will be applied to the DELETE HTTP method.  Overrides default.

<h3>Object Ownership</h3>
Often, the API doesn't need to authenticate users based on any sort of roles, but upon what objects are actually owned
by the user.  auto-crud provides a facility to automatically tag mongo documents with a value representing the owner.
Typically this is the ObjectID of the user in question.

```javascript
autocrud({
    ... // Default options
    ownerIdFromReq: function (req) {
        return new ObjectID(req.user._id);
    },
    ownerField: 'owner'
});
```
It is also possible to tell autocrud that objects own themselves.  This is useful for the case in which you have a user
object, as a user should be able to edit itself and only itself.  When an object owns itself, the POST method does not
require authentication to use.  This allows new users registering with your site to create an account, then login to
modify it.  NOTE: You cannot specify ownerSelf and ownerField at the same time. When an object owns itself, its
ownerField is always its own "_id" field.

```javascript
autocrud({
	... // Default options
	ownerIdFromReq: function (req) {
		return new ObjectID(req.user._id);
	}
	ownerSelf: true
});
```

NOTE: To enable object ownership the `ownerIdFromReq` field and either the `ownerField` or `ownerSelf` fields must be
provided.  You cannot use `ownerField` and `ownerSelf` at the same time.
* `ownerIdFromReq` A function that extracts the user id value from the request object.  This is passed as the first param.
* `ownerField` The name of the field in each mongo document that holds the owner id.
* `ownerSelf` If true the object uses its own "_id" field as its ownerField.

<h2>Events</h2>
There are a handful of events that will be fired with http calls to the Autocrud object.

* `get` Called when get by search http route is accessed.
    *  `documents` The documents that were returned as a result of the search call.  Only the current page if limit and
skip are provided.
    *  `count` The total number of documents in the database that matched the search criteria
* `getId` Called when get by id http route is accessed.
    *  `id` The id of the resource accessed.
    *  `document` The document that was returned to the caller.
* `post`
    *  `document` The document that was inserted to the database.
* `putId`
    *  `id` The id of the resource accessed.
    *  `document` The changes that were made to the database document.
    *  `modCount` The number of documents that were updated.
* `deleteId`
    *  `id` The id of the resource deleted.
    *  `modCount` The number of documents that were deleted.
