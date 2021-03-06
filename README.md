# Fawn
## Promise based Library for atomic-ish operations in MongoDB

Fawn provides the ability to carry out edits on a mongoDB database as a series of steps. If an error occurs on any of the steps, the database is returned to it's initial state (it's state before the transaction started).

**View on [GitHub](https://github.com/e-oj/Fawn/)**

- [Getting Started](#getting_started)
- [Usage](#usage)
- [API](#api)
- [Test](#test)

## <a name="getting_started"></a>Getting Started:

Install [node.js](https://nodejs.org) and [mongoDB](https://www.mongodb.com/download-center)

Start mongoDB in a terminal: ```mongod```

Then:
```npm install fawn```

## <a name="usage"></a>Usage:

```javascript
var Fawn = require("fawn");
```

### <a name="examples"></a>Examples
Say you have two bank accounts, one belongs to John Smith and the other belongs to Broke Ass. You would like to transfer $20 from John Smith to Broke Ass. Assuming all first name and last name pairs are unique, this might look like:

```javascript
var task = Fawn.Task()

//assuming "Accounts" is the Accounts collection
task.update("Accounts", {firstName: "John", lastName: "Smith"}, {$inc: {balance: -20}})
  .update("Accounts", {firstName: "Broke", lastName: "Ass"}, {$inc: {balance: 20}})
  .run()
  .then(function(){
    //update is complete
  })
  .catch(function(err){
    // Everything has been rolled back.
    
    //log the error which caused the failure
    console.log(err);
  });
```

if you prefer not to chain function calls, you don't have to:

```javascript
task.update("Accounts", {firstName: "Broke", lastName: "Ass"}, {$inc: {balance: -20}})
task.update("Accounts", {firstName: "Coke", lastName: "Dealer"}, {$inc: {balance: 20}})
task.run()
  .then(function(){
    //update is complete
  })
  .catch(function(err){
    // Everything has been rolled back.
    
    //log the error which caused the failure
    console.log(err);
  });
```

The server could crash before a task is complete, You can use the Roller to rollback all incomplete transactions before starting your server.

```javascript
// assuming Fawn has been initialized. See Fawn.init below
var roller = Fawn.Roller();

roller.roll()
  .then(function(){
    // start server
  });
```

## <a name="api"></a>API

- [Fawn.init](#fawn_init)
- [Fawn.Task](#fawn_task)
- [task.initmodel](#task_initmodel)
- [task.save](#task_save)
- [task.update](#task_update)
- [task.options](#task_options)
- [task.remove](#task_remove)
- [task.run](#task_run)
- [Fawn.Roller](#fawn_roller)
- [Roller.roll](#roller_roll)

### <a name="fawn_init"></a>Fawn.init(db, _collection, options): Initialize Fawn

> db (required): [mongoose](https://github.com/Automattic/mongoose) instance or [connection string](https://docs.mongodb.com/manual/reference/connection-string/)

> _collection (optional): Name of collection used internally by Fawn to store transactions

> options (optional. lol): Connection options. Same as [mongoose connection options](http://mongoosejs.com/docs/connections.html#options)

<br> **Note: if you're running multiple apps connected to the same db, provide a string value for _collection that's unique to each app. Do this to avoid a situation where one app rolls back the unfinished transaction(s) of another app.**

If you're using mongoose in your project initialize Fawn with mongoose:

```javascript
var mongoose = require("mongoose");

mongoose.connect("mongodb://127.0.0.1:27017/testDB");

// remember, _collection is optional
Fawn.init(mongoose, "Fawn_collection_name_if_you_want_to_specify");
```

Without mongoose, Initialze Fawn like so:

```javascript
// options object (http://mongoosejs.com/docs/connections.html#options)
var options = {
  user: "teh_huose_kat",
  pass: "teh_Kitti_passwrod"
}

var collection = "Fawn_collection_name_if_you_want_to_specify";

// remember, _collection and options are optional
Fawn.init("mongodb://127.0.0.1:27017/testDB", collection || null, options || null);
```
<br>

### <a name="fawn_task"></a>Fawn.Task(): Create a Fawn task
  
  > returns: A new task

After intitializing Fawn, create a task like so:

```javascript
var task = Fawn.Task();
```
<br>

### <a name="task_initmodel"></a>task.initModel(modelName, schema): To initialize a model with a Schema.

  > modelName (required): Name of the collection associated with this model
  
  > schema (optional): Same as object passed to [mongoose Schema](http://mongoosejs.com/docs/guide.html#definition). Also see [validation](http://mongoosejs.com/docs/validation.html)
  
  <br>If you're using mongoose, define your models with mongoose wherever possible. If the model has been defined by mongoose before this function is called, mongoose will throw an OverwriteModelError and if it was defined by Fawn, Fawn will throw an Error. Models can be defined only once.
  
  ```javascript
  var schema = {
    name: {type: String, required: true}
    , specials: [{title: String, year: Number}]
  }
  
  task.initModel("comedians", schema);
  ```
  
  Save operations to the "comedians" model will validate against the schema;

<br>

### <a name="task_save"></a>task.save(model, doc): To save a document</b>

  > model (required): Name of the collection we're saving to or a mongoose model or a mongoose document

  > doc (optional): Object to save or a mongoose document
	
  <br>these are all valid:
  
  ```javascript
  var Cars = mongoose.model("cars", new Schema({make: String, year: Number}));
  var toyota = new Cars({make: "Toyota", year: 2015});

  task.save("cars", {make: "Toyota", year: 2015});
  task.save(Cars, {make: "Toyota", year: 2015});
  task.save("cars", toyota);
  task.save(Cars, toyota);
  task.save(toyota);
  ```
  
  *Note: No changes will be made to to your database until you call task.run()*
  
<br> 

[mongoose]: <http://mongoosejs.com/docs/api.html#model_Model.update> 

[mongodb]: <https://docs.mongodb.com/manual/core/document/#document-query-filter>

### <a name="task_update"></a>task.update(model, condition, data): To update a document

  > model (required): Name of the collection we're updating or a mongoose model or a mongoose document

  > condition (required): Same as in [mongoose][] and [mongodb][]
  
  > data (optional): Data to update with same as in [mongoose][] and [mongodb](https://docs.mongodb.com/manual/reference/method/db.collection.update/#update-parameter)
  
  <br> These are all valid
 
  ```javascript
  var Cars = mongoose.model("cars", new Schema({make: String, year: Number}));
 
  task.update("cars", {make: "Toyota"}, {year: 2016});
  task.update(Cars, {make: "Toyota"}, {year: 2016});
  
  Cars.findOne({make: "Toyota"}, function(toyota){
    task.update(toyota, {year: 2016});
  });
 ```
 
  *Note: No changes will be made to to your database until you call task.run()*
  
  <br>
  
### <a name="task_options"></a>task.options(options): Add options to an update task.

  > options (required): Update options - same as in [mongoose][]
  
  <br> Attach to update call as shown
  
  ```javascript
  task.update("cars", {make: "Toyota"}, {year: 2016})
    .options({multi: true});
  
  // Also valid
  
  task.update("cars", {make: "Ford"}, {year: 2016});
  task.options({multi: true});
  ```
  
  *Note: No changes will be made to to your database until you call task.run()*

  <br>
  
### <a name="task_remove"></a>task.remove(model, condition): Remove document(s) from a collection

  > model (required): Name of the collection we're deleting from or a mongoose model or a mongoose document
  
  > condition (optional): Same as in [mongoose](http://mongoosejs.com/docs/api.html#query_Query-remove)
  
  <br> These are all valid
  
  ```javascript
  var Cars = mongoose.model("cars", new Schema({make: String, year: Number}));
  
  // removes all cars with year === 2015
  task.remove("cars", {year: 2015});
  task.remove(Cars, {year: 2015});
  
  Cars.findOne({year: 2015}, function(car){
    // remove just this car
    task.remove(car);
  })
  ```

  *Note: No changes will be made to to your database until you call task.run()*

  <br> 
  
### <a name="task_run"></a>task.run(): Run a task.
  
  > returns: Promise

  For the database changes to occur, you must call task.run(). This function returns a promise. If an error occurs, the promise is rejected with the error that caused the failure.
  
  ```javascript
  task.update("Accounts", {firstName: "John", lastName: "Smith"}, {$inc: {balance: -20}})
    .update("Accounts", {firstName: "Broke", lastName: "Ass"}, {$inc: {balance: 20}})
    .run()
    .then(function(){
      //update is complete
    })
    .catch(function(err){
      // Everything has been rolled back.
      
      //log the error which caused the failure
      console.log(err);
    });
  ```
  <br>
  
### <a name="fawn_roller"></a>Fawn.Roller(): Get the Roller object.
  
  > returns: The Roller object
  
  After initializing  Fawn, get the Roller like so:
  
  ```javascript
  var Roller = Fawn.Roller();
  ```
 <br>
 
### <a name="roller_roll"></a>Roller.roll(): Roll back all incomplete transcations
  
  Returns all the documents affected by incomplete transactions to their original state. Should only be used when no tasks are in progress, usually on server startup.
  
  ```javascript
  var roller = Fawn.Roller();
  
  roller.roll()
    .then(function(){
      // start server
    });
  ```
  
## <a name="test"></a>Test

  To test this module, start mongodb in a terminal
  
  ```
  mongod
  ```
  
  Then cd to the project directory and run 
  
  ```
  npm test
  ```
