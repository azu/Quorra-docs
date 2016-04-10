# Models & ORM

 - [Introduction](#introduction)
 - [Basic usage](#basic-usage)
 - [Insert, Update, Delete](#insert-update-delete)
 - [Associations](#associations)
 - [Validations](#validations)

## Introduction

Quorra makes connecting with databases and running queries extremely simple. Quorra comes installed with a powerful
ORM/ODM called [Waterline](https://github.com/balderdashy/waterline), a datastore-agnostic tool that dramatically
simplifies interaction with one or more databases. It provides an abstraction layer on top of the underlying
database, allowing you to easily query and manipulate your data without writing vendor-specific integration code.

Before getting started, be sure to configure a database connection in `app/config/database.js`.

## Basic usage

To get started, create a waterline model. Models live in the `app/models` directory of your application.

### Defining An Waterline Model

```javascript
// path: app/models/User.js
var User = {
    attributes: {
       ...
    }
};

module.exports = User;
```
#### tableName

Note that we did not tell Waterline which table to use for our User model. The lower-case, plural name of the model
file will be used as the table name unless another name is explicitly specified. So, in this case, Waterline will
assume the User model stores records in the user table. You may specify a custom table by defining a tableName property
on your model:

```javascript
// path: app/models/User.js
var User = {
    tableName: 'my_users',

    attributes: {
       ...
    }
};

module.exports = User;
```

#### schema

```javascript
schema: true
```

A flag to toggle schemaless or schema mode in databases that support schemaless data structures. If turned off, this
will allow you to store arbitrary data in a record. If turned on, only attributes defined in the model's attributes
object will be stored.

For adapters that don't require a schema, such as Mongo or Redis, the default setting is `schema:false`.

#### connection

```javascript
connection: 'my-local-postgresql'
```
You may define a `connection` property to override the name of the database connection that should be used when
utilizing the model. Defaults to model.connection attribute specified in the configuration file `app/config/database.js`

#### autoPk

```javascript
autoPK: true
```

Waterline will define a primary key for each table by default.  The details of this default PK vary between
adapters (e.g. MySQL uses an auto-incrementing integer primary key, whereas MongoDB uses a randomized string UUID).In
any case, the primary keys generated by Waterline will be unique. You may define a `primaryKey` property to override
this convention or set attribute `autoPK` to false in your model.

#### autoUpdatedAt & autoCreatedAt

You will need to place updatedAt and createdAt columns on your table by default. If you do not wish to have these
columns automatically maintained, set the `autoUpdatedAt` and `autoCreatedAt` property on your model file to false.

#### identity

```javascript
identity: 'user'
```

The lowercase unique key for this model, e.g. user. By default, a model's identity is inferred automatically by
lowercasing its filename. All waterline models are attached to `app.models` object by identity as attribute. So you
can access a model like following:

```javascript
User = App.models.user // where user is the model identity
```

#### globalId

```javascript
globalId: 'User'
```

This flag changes the global name by which you can access your model (if the globalization of models is enabled). If
you do not specify a `globalId` attribute on your model model `identity` will be used as `globalId`

#### attributes

```javascript
attributes: {
  name: { type: 'string' },
  email: { type: 'email' },
  age: { type: 'integer' }
}
```

See [Model Attributes](https://github.com/balderdashy/waterline-docs/blob/master/models/data-types-attributes.md).

### migrate

This flag sets the schema to automatically alter the schema, drop the schema or make no changes (safe). Default: `alter`

 - safe - never auto-migrate my database(s). I will do it myself (by hand).
 - alter - auto-migrate, but attempt to keep my existing data (experimental).
 - drop - wipe/drop ALL my data and rebuild models every time I run Quorra server.

In a production environment (`NODE_ENV==="production"`) Quorra always uses `migrate: "safe"` to protect inadvertent
deletion
 of your data.

### Retrieving All Models

```javascript
Model.find(function(err, users){
    //
});
```

### Retrieving A Record By Primary Key

```javascript
Model.findOne(1, function(err, user){
    console.log(user.name);
});
```

### Querying Using Waterline Models

```javascript
Model.find({ where: { name: 'foo' }, skip: 20, limit: 10, sort: 'name DESC' })
    .exec(function(err, users){
        //
    });
```

Find out more about querying in the [waterline docs](https://github.com/balderdashy/waterline-docs).

### Waterline Aggregates

```javascript
Model.count({ age: { '<': 30 }}, function(err, count) {
//
})

// Find the highest grossing movie by genre.
Movie.find()
    .groupBy('genre')
    .max('revenue')
    .then(function(results) {
        // Max revenue for the first genre.
        results[0].revenue;
    });
```

Some adapters, such as [sails-mysql](https://github.com/balderdashy/sails-mysql) and [sails-postgresql](https://github.com/balderdashy/sails-postgresql), support the query function which will run the provided RAW
query against the database. This can sometimes be useful if you want to run complex queries and performance is very
important.

```javascript
var title = "The King's Speech";
Movie.query('SELECT * FROM movie WHERE title = $1', [title], function(err, results) {
  // using sails-postgresql
  console.log('Found the following movie: ', results.rows[0]);

  // using sails-mysql
  console.log('Found the following movie: ', results[0]);
});
```

Find out more about aggregates in the [waterline docs](https://github.com/balderdashy/waterline-docs).

## Insert, Update, Delete

To create a new record in the database from a model, simply pass the arguments object to the model create method.

### Saving A New Record

```javascript
User.create({
  name: 'Walter Jr'
})
.exec(function(err, user) {});
```

After saving or creating a new record that uses auto-incrementing IDs, you may retrieve the ID by accessing the
object's id attribute:

```javascript
var insertedId = user.id;

// Retrieve the user by the attributes, or create it if it doesn't exist...
User.findOrCreate({ name: 'Walter Jr' })
.exec(function(err, users) {
//either user(s) with the name 'Walter Jr' get returned or
//a single user gets created with the name 'Walter Jr' and returned
});
```

### Updating A Record

To update records use the update method:

```javascript
//by criteria
User.update({name:'Walter Jr'},{name:'Flynn'}).exec(function afterwards(err, updated){

  if (err) {
    // handle error here- e.g. `res.serverError(err);`
    return;
  }

  console.log('Updated user to have name ' + updated[0].name);
});

//by primary key
User.update(78234,{name:'Flynn'}).exec(function afterwards(err, updated){

  if (err) {
    // handle error here- e.g. `res.serverError(err);`
    return;
  }

  console.log('Updated user to have name ' + updated[0].name);
});
```

### Deleting An Existing Record

To delete a record, simply call the destroy method on the model with criteria or primary key:

```javascript
User.destroy({ name: 'Flynn' })
.exec(function(err) {});

User.destroy(234)
.exec(function(err) {});
```

## Associations

Of course, your database tables are probably related to one another. For example, a blog post may have many comments,
or an order could be associated to the user who placed it. Waterline makes managing and working with these associations
easy. Waterline supports many types of associations:

 - [One To One](https://github.com/balderdashy/waterline-docs/blob/master/models/associations/one-to-one.md)
 - [One To One](https://github.com/balderdashy/waterline-docs/blob/master/models/associations/one-to-one.md)
 - [One To Many](https://github.com/balderdashy/waterline-docs/blob/master/models/associations/one-to-many.md)
 - [Many To Many](https://github.com/balderdashy/waterline-docs/blob/master/models/associations/many-to-many.md)
 - [Dominance](https://github.com/balderdashy/waterline-docs/blob/master/models/associations/dominance.md)

## Querying Associations

`.populate( foreignKey, [query] )` a chainable method is used between `.find()/.update()` and `.exec()` in order to
retrieve records associated with the model being queried. You must supply the Foreign Key specified in your model
config.

```javascript
User.find({name:'Mike'}).exec(function(e,r){
  console.log(r[0].toJSON())
})

/*
{ name: 'Mike',
  age: 16,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 7 }
*/

User.find({name:'Mike'}).populate('pets').exec(function(e,r){
  console.log(r[0].toJSON())
});

/*
{ pets:
   [ { name: 'Pinkie Pie',
       color: 'pink',
       id: 7,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Rainbow Dash',
       color: 'blue',
       id: 8,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Applejack',
       color: 'orange',
       id: 9,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) } ],
  name: 'Mike',
  age: 16,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 7 }
*/

User.find({name:'Mike'}).populate('pets',{color:'pink'}).exec(function(e,r){
  console.log(r[0].toJSON())
});

/*
{ pets:
   [ { name: 'Pinkie Pie',
       color: 'pink',
       id: 7,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) }],
  name: 'Mike',
  age: 16,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 7 }
*/
```
## Validations

Validations are handled by [Anchor](https://github.com/balderdashy/anchor) which is based off of
[Node Validate](https://github.com/chriso/validator.js) and supports most of the properties in
node-validate. For a full list of validations see: [Anchor Validations](https://github.com/balderdashy/anchor/blob/master/lib/match/rules.js).

Validations are defined directly in your Collection attributes. In addition you may set the attribute
type to any supported Anchor type and Waterline will build a validation and set the schema type as
a string for that attribute.

Validation rules may be defined as simple values or functions (both sync and async) that return the
value to test against.

Available validations are:

```javascript
attributes: {
  foo: {
    empty: true,
    required: true,
    notEmpty: true,
    undefined: true,
    string:
    alpha: true,
    numeric: true,
    alphanumeric: true,
    email: true,
    url: true,
    urlish: true,
    ip: true,
    ipv4: true,
    ipv6: true,
    creditcard: true,
    uuid: true,
    uuidv3: true,
    uuidv4: true,
    int: true,
    integer: true,
    number: true,
    finite: true,
    decimal: true,
    float: true,
    falsey: true,
    truthy: true,
    null: true,
    notNull: true,
    boolean: true,
    array: true,
    date: true,
    hexadecimal: true,
    hexColor: true,
    lowercase: true,
    uppercase: true,
    after: '12/12/2001',
    before: '12/12/2001',
    is: /ab+c/,
    regex: /ab+c/,
    not: /ab+c/,
    notRegex: /ab+c/,
    equals: 45,
    contains: 'foobar',
    notContains: 'foobar',
    len: 35,
    in: ['foo', 'bar'],
    notIn: ['foo', 'bar'],
    max: 24,
    min: 4,
    minLength: 4,
    maxLength: 24
  }
}
```

### Custom validations

Validations can also be defined as functions, either sync or async.

```javascript
attributes: {
  website: {
    type: 'string',
    contains: function(cb) {
      setTimeout(function() {
        cb('http://');
      }, 1);
    }
  }
}
```

Validations can also be used against other attributes using the `this` context.

```javascript
attributes: {
  startDate: {
    type: 'date',
    before: function() {
      return this.endDate;
    }
  },

  endDate: {
    type: 'date',
    after: function() {
      return this.startDate;
    }
  }
}
```

### Validation rules

| Name of validator | What does it check? | Notes on usage |
|-------------------|---------------------|----------------|
|after| check if `string` date in this record is after the specified `Date` | must be valid javascript `Date` |
|alpha| check if `string` in this record contains only letters (a-zA-Z) | |
|alphadashed|| does this `string` contain only numbers and/or dashes? |
|alphanumeric| check if `string` in this record contains only letters and numbers. | |
|alphanumericdashed| does this `string` contain only numbers and/or letters and/or dashes? | |
|array| is this a valid javascript `array` object? | strings formatted as arrays won't pass |
|before| check if `string` in this record is a date that's before the specified date | |
|binary| is this binary data? | If it's a string, it will always pass |
|boolean| is this a valid javascript `boolean` ? | `string`s will fail |
|contains| check if `string` in this record contains the seed | |
|creditcard| check if `string` in this record is a credit card | |
|date| check if `string` in this record is a date | takes both strings and javascript |
|datetime| check if `string` in this record looks like a javascript `datetime`| |
|decimal| | contains a decimal or is less than 1?|
|email| check if `string` in this record looks like an email address | |
|empty| Arrays, strings, or arguments objects with a length of 0 and objects with no own enumerable properties are considered "empty" | lo-dash _.isEmpty() |
|equals| check if `string` in this record is equal to the specified value | `===` ! They must match in both value and type |
|falsey| Would a Javascript engine register a value of `false` on this? | |
|finite| Checks if given value is, or can be coerced to, a finite number | This is not the same as native isFinite which will return true for booleans and empty strings |
|float| check if `string` in this record is of the number type float | |
|hexadecimal| check if `string` in this record is a hexadecimal number | |
|hexColor| check if `string` in this record is a hexadecimal color | |
|in| check if `string` in this record is in the specified array of allowed `string` values | |
|int|check if `string` in this record is an integer | |
|integer| same as above | Im not sure why there are two of these. |
|ip| check if `string` in this record is a valid IP (v4 or v6) | |
|ipv4| check if `string` in this record is a valid IP v4 | |
|ipv6| check if `string` in this record is aa valid IP v6 | |
|is| | something to do with REGEX|
|json| does a try&catch to check for valid JSON. | |
|len| is `integer` > param1 && < param2 | Where are params defined? |
|lowercase| is this string in all lowercase? | |
|max| | |
|maxLength| is `integer` > 0 && < param2 |  |
|min| | |
|minLength| | |
|not| | Something about regexes |
|notContains| | |
|notEmpty| | WTF |
|notIn| does the value of this model attribute exist inside of the defined validator value (of the same type) | Takes strings and arrays |
|notNull| does this not have a value of `null` ? | |
|notRegex| | |
|null| check if `string` in this record is null | |
|number| is this a number? | NaN is considered a number |
|numeric| checks if `string` in this record contains only numbers | |
|object| checks if this attribute is the language type of Object | Passes for arrays, functions, objects, regexes, new Number(0), and new String('') ! |
|regex| | |
|required| Must this model attribute contain valid data before a new record can be created? | |
|string| is this a `string` ?| |
|text| okay, well is <i>this</i> a `string` ?| |
|truthy| Would a Javascript engine register a value of `false` on this? | |
|undefined| Would a javascript engine register this thing as have the value 'undefined' ? | |
|uppercase| checks if `string` in this record is uppercase | |
|url| checks if `string` in this record is a URL | |
|urlish| Does the `string` in this record contain something that looks like a route, ending with a file extension? | /^\s([^\/]+\.)+.+\s*$/g |
|uuid| checks if `string` in this record is a UUID (v3, v4, or v5) | |
|uuidv3| checks if `string` in this record is a UUID (v3) | |
|uuidv4| checks if `string` in this record is a UUID (v4) | |

### Custom Validation Types

You can define your own types and their validation with the `types` object. It's possible to access
and compare values to other attributes. This allows you to move validation business logic into your
models and out of your controller logic.

```javascript
var User = {
  types: {
    point: function(latlng){
      return latlng.x && latlng.y
    },

    password: function(password) {
      return password === this.passwordConfirmation;
    });
  },

  attributes: {
    firstName: {
      type: 'string',
      required: true,
      minLength: 5,
      maxLength: 15
    },

    location: {
      type: 'json',
      point: true
    },

    password: {
      type: 'string',
      password: true
    },

    passwordConfirmation: {
      type: 'string'
    }
  }
};

module.exports = User;
```