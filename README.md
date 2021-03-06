Collection2 [![Build Status](https://travis-ci.org/aldeed/meteor-collection2.png?branch=master)](https://travis-ci.org/aldeed/meteor-collection2)
=========================

A smart package for Meteor that extends Meteor.Collection to provide support
for specifying a schema and then validating against that schema
when inserting and updating.

This package requires and automatically installs the 
[simple-schema](https://github.com/aldeed/meteor-simple-schema) package,
which provides the `SimpleSchema` object type for defining and validating
against schemas.

## Installation

Install using Meteorite. When in a Meteorite-managed app directory, enter:

```
$ mrt add collection2
```

## Why Use Collection2?

* While adding allow/deny rules ensures that only authorized users can edit a
document from the client, adding a schema ensures that only acceptable properties
and values can be set within that document from the client. Thus, client side
inserts and updates can be allowed without compromising security or data integrity.
* Schema validation for all inserts and updates is reactive, allowing you to
easily display customizable validation error messages to the user without any
event handling.
* Schema validation for all inserts and updates is automatic on both the client
and the server, providing both speed and security.
* The [autoform](https://github.com/aldeed/meteor-autoform) package can
take your collection's schema and automatically create HTML5 forms based on it.
AutoForm provides automatic database operations, method calls, validation, and
user interface reactivity. You have to write very little markup and no event
handling. Refer to the [autoform](https://github.com/aldeed/meteor-autoform)
documentation for more information.

## Example

Define the schema for your collection by setting the `schema` option to
a `SimpleSchema` instance.

```js
Books = new Meteor.Collection("books", {
    schema: new SimpleSchema({
        title: {
            type: String,
            label: "Title",
            max: 200
        },
        author: {
            type: String,
            label: "Author"
        },
        copies: {
            type: Number,
            label: "Number of copies",
            min: 0
        },
        lastCheckedOut: {
            type: Date,
            label: "Last date this book was checked out",
            optional: true
        },
        summary: {
            type: String,
            label: "Brief summary",
            optional: true,
            max: 1000
        }
    })
});
```

Do an insert:

```js
Books.insert({title: "Ulysses", author: "James Joyce"}, function(error, result) {
  //The insert will fail, error will be set,
  //and result will be undefined or false because "copies" is required.
  //
  //The list of errors is available by calling Books.simpleSchema().namedContext().invalidKeys()
});
```

Or do an update:

```js
Books.update(book._id, {$unset: {copies: 1}}, function(error, result) {
  //The update will fail, error will be set,
  //and result will be undefined or false because "copies" is required.
  //
  //The list of errors is available by calling Books.simpleSchema().namedContext().invalidKeys()
});
```

## Attaching a Schema to a Collection

As you saw in the example, the typical way of attaching a SimpleSchema to a collection is to
provide it as the `schema` option for the constructor. Another way to attach a schema is to
call `myCollection.attachSchema(mySimpleSchemaInstance)`. This is particularly useful for
collections created by other packages, such as the `Meteor.users` collection.

Obviously, when you attach a schema, you must know what the schema should be. For `Meteor.users`,
something like the following should work:

```js
{
  username: {
    type: String,
    optional: true
  },
  emails: {
    type: [Object],
    optional: true,
    blackbox: true
  },
  services: {
    type: Object,
    optional: true,
    blackbox: true
  },
  createdAt: {
    type: Date,
    optional: true
  },
  profile: {
    type: Object,
    optional: true,
    blackbox: true
  }
}
```

This schema has not been thoroughly vetted to ensure
that it accounts for all possible properties the accounts packages might try to set. Furthermore,
any other packages you add might also try to set additional properties. If you see warnings in the
console about keys being removed, that's a good indication that you should add those keys to the
schema.

Note also that this schema uses the `blackbox: true` option for simplicity. You might choose instead
to figure out a more specific schema.

(If you figure out a more accurate `Meteor.users` schema, documentation pull requests are welcome.)

## Schema Format

Refer to the
[simple-schema](https://github.com/aldeed/meteor-simple-schema) package
documentation for a list of all the available schema rules and validation
methods.

Use the `MyCollection2.simpleSchema()` method to access the bound `SimpleSchema`
instance for a Meteor.Collection instance. For example:

```js
check(doc, MyCollection2.simpleSchema());
```

## Validation Contexts

In the examples above, note that we called `namedContext()` with no arguments
to access the SimpleSchema reactive validation methods. Contexts let you keep
multiple separate lists of invalid keys for a single collection2.
In practice you might be able to get away with always using the default context.
It depends on what you're doing. If you're using the context's reactive methods
to update UI elements, you might find the need to use multiple contexts. For example,
you might want one context for inserts and one for updates, or you might want
a different context for each form on a page.

To use a specific named validation context, use the `validationContext` option
when calling `insert` or `update`:

```js
Books.insert({title: "Ulysses", author: "James Joyce"}, { validationContext: "insertForm" }, function(error, result) {
  //The list of errors is available by calling Books.simpleSchema().namedContext("insertForm").invalidKeys()
});

Books.update(book._id, {$unset: {copies: 1}}, { validationContext: "updateForm" }, function(error, result) {
  //The list of errors is available by calling Books.simpleSchema().namedContext("updateForm").invalidKeys()
});
```

## Validating Without Inserting or Updating

It's also possible to validate a document without performing the actual insert or update:

```js
Books.simpleSchema().namedContext().validate({title: "Ulysses", author: "James Joyce"}, {modifier: false});
```

Set the modifier option to true if the document is a mongo modifier object.

You can also validate just one key in the document:

```js
Books.simpleSchema().namedContext().validateOne({title: "Ulysses", author: "James Joyce"}, "title", {modifier: false});
```

Or you can specify a certain validation context when calling either method:

```js
Books.simpleSchema().namedContext("insertForm").validate({title: "Ulysses", author: "James Joyce"}, {modifier: false});
Books.simpleSchema().namedContext("insertForm").validateOne({title: "Ulysses", author: "James Joyce"}, "title", {modifier: false});
```

## Inserting or Updating Without Validating

To skip validation, use the `validate: false` option when calling `insert` or
`update`. On the client (untrusted code), this will skip only client-side
validation. On the server (trusted code), it will skip all validation.

## Additional SimpleSchema Options

In addition to all the other schema validation options documented in the 
[simple-schema](https://github.com/aldeed/meteor-simple-schema) package, the
collection2 package adds additional options explained in this section.

### unique

Set `unique: true` in your schema to ensure that non-unique values will never
be set for the key. You may want to ensure a unique mongo index on the server
as well. Refer to the documentation for the `index` option.

The error message for this is very generic. It's best to define your own using
`MyCollection.simpleSchema().messages()`. The error type string is "notUnique".

### denyInsert and denyUpdate

If you set `denyUpdate: true`, any collection update that modifies the field
will fail. For instance:

```js
Posts = new Meteor.Collection('posts', {
  schema: new SimpleSchema({
    title: {
      type: String
    },
    content: {
      type: String
    },
    createdAt: {
      type: Date,
      denyUpdate: true
    }
  })
});

var postId = Posts.insert({title: 'Hello', content: 'World', createdAt: new Date});
```

The `denyInsert` option works the same way, but for inserts. If you set
`denyInsert` to true, you will need to set `optional: true` as well. 

### autoValue

The `autoValue` option is provided by the SimpleSchema package and is documented
there. Collection2 adds the following properties to `this` for any `autoValue`
function that is called as part of a C2 database operation:

* isInsert: True if it's an insert operation
* isUpdate: True if it's an update operation
* isUpsert: True if it's an upsert operation (either `upsert()` or `upsert: true`)
* userId: The ID of the currently logged in user. (Always `null` for server-initiated actions.)
* isFromTrustedCode: True if the insert, update, or upsert was initiated from trusted (server) code

Note that autoValue functions are run on the client only for validation purposes,
but the actual value saved will always be generated on the server, regardless of
whether the insert/update is initiated from the client or from the server.

There are many possible use cases for `autoValue`. It's probably easiest to
explain by way of several examples:

```js
{
  // Force value to be current date (on server) upon insert
  // and prevent updates thereafter.
  createdAt: {
    type: Date,
      autoValue: function() {
        if (this.isInsert) {
          return new Date;
        } else if (this.isUpsert) {
          return {$setOnInsert: new Date};
        } else {
          this.unset();
        }
      },
      denyUpdate: true
  },
  // Force value to be current date (on server) upon update
  // and don't allow it to be set upon insert.
  updatedAt: {
    type: Date,
    autoValue: function() {
      if (this.isUpdate) {
        return new Date();
      }
    },
    denyInsert: true,
    optional: true
  },
  // Whenever the "content" field is updated, automatically set
  // the first word of the content into firstWord field.
  firstWord: {
    type: String,
    optional: true,
    autoValue: function() {
      var content = this.field("content");
      if (content.isSet) {
        return content.value.split(' ')[0];
      } else {
        this.unset(); // Prevent user from supplying her own value
      }
    }
  },
  // Whenever the "content" field is updated, automatically
  // update a history array.
  updatesHistory: {
    type: [Object],
    optional: true,
    autoValue: function() {
      var content = this.field("content");
      if (content.isSet) {
        if (this.isInsert) {
          return [{
              date: new Date,
              content: content.value
            }];
        } else {
          return {
            $push: {
              date: new Date,
              content: content.value
            }
          };
        }
      } else {
        this.unset();
      }
    }
  },
  'updatesHistory.$.date': {
    type: Date,
    optional: true
  },
  'updatesHistory.$.content': {
    type: String,
    optional: true
  },
  // Automatically set HTML content based on markdown content
  // whenever the markdown content is set.
  htmlContent: {
    type: String,
    optional: true,
    autoValue: function(doc) {
      var markdownContent = this.field("markdownContent");
      if (Meteor.isServer && markdownContent.isSet) {
        return MarkdownToHTML(markdownContent.value);
      }
    }
  }
}
```

### index

Use the `index` option to ensure a MongoDB index for a specific field:

```js
{
  title: {
    type: String,
    index: 1
  }
}
```

Set to `1` or `true` for an ascending index. Set to `-1` for a descending index.
Or you may set this to another type of specific MongoDB index, such as `"2d"`.
Indexes works on embedded sub-documents as well.

If you have created an index for a field by mistake and you want to remove it,
set `index` to `false`:

```js
{
  "address.street": {
    type: String,
    index: false
  }
}
```

If a field has the `unique` option set to `true`, the MongoDB index will be a
unique index as well. Then on the server, Collection2 will rely on MongoDB
to check uniqueness of your field, which is more efficient than our
custom checking.

```js
{
  "pseudo": {
    type: String,
    index: true,
    unique: true
  }
}
``` 

Indexes are built in the background so indexing does *not* block other database
queries.

### custom

The `custom` option is provided by the SimpleSchema package and is documented
there. Collection2 adds the following properties to `this` for any `custom`
function that is called as part of a C2 database operation:

* isInsert: True if it's an insert operation
* isUpdate: True if it's an update operation
* isUpsert: True if it's an upsert operation (either `upsert()` or `upsert: true`)
* userId: The ID of the currently logged in user. (Always `null` for server-initiated actions.)
* isFromTrustedCode: True if the insert, update, or upsert was initiated from trusted (server) code

## What Happens When The Document Is Invalid?

The callback you specify as the last argument of your `insert()` or `update()` call
will have the first argument (`error`) set to a generic error. But generally speaking,
you would probably use the reactive methods provided by the SimpleSchema
validation context to display the specific error messages to the user somewhere.
The [autoform](https://github.com/aldeed/meteor-autoform) package provides
some handlebars helpers for this purpose.

## More Details

For the curious, this is exactly what Collection2 does before every insert or update:

1. Removes properties from your document or mongo modifier object if they are
not explicitly listed in the schema.
2. Automatically converts some properties to match what the schema expects, if possible.
3. Adds automatic (forced or default) values based on your schema.
4. Validates your document or mongo modifier object.
5. Performs the insert or update like normal, only if it was valid.

Collection2 is simply calling SimpleSchema methods to do these things.

This check happens on both the client and the server for client-initiated
actions, giving you the speed of client-side validation along with the security
of server-side validation.

## Virtual Fields

**NOTE: Virtual fields support may eventually be retired. I recommend using
the [collection-helpers](https://github.com/dburles/meteor-collection-helpers)
package instead since it achieves a similar result but does it more efficiently.**

You can also implement easy virtual fields. Here's an example of that:

```js
Persons = new Meteor.Collection("persons", {
    schema: new SimpleSchema({
        firstName: {
            type: String,
            label: "First name",
            max: 30
        },
        lastName: {
            type: String,
            label: "Last name",
            max: 30
        }
    }),
    virtualFields: {
        fullName: function(person) {
            return person.firstName + " " + person.lastName;
        }
    }
});
```

This adds the virtual field to documents retrieved with `find()`, etc., which means you could
now do `{{fullName}}` in your HTML as if fullName were actually stored in the MongoDB collection.
However, you cannot query on a virtual field.

## Contributing

Anyone is welcome to contribute. Fork, make and test your changes (`mrt test-packages ./`),
and then submit a pull request.

### Major Contributors

@mquandalle

[![Support via Gittip](https://rawgithub.com/twolfson/gittip-badge/0.2.0/dist/gittip.png)](https://www.gittip.com/aldeed/)
