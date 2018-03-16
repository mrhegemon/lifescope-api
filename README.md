# graphql-compose-mongoose

[![travis build](https://img.shields.io/travis/graphql-compose/graphql-compose-mongoose.svg)](https://travis-ci.org/graphql-compose/graphql-compose-mongoose)
[![codecov coverage](https://img.shields.io/codecov/c/github/graphql-compose/graphql-compose-mongoose.svg)](https://codecov.io/github/graphql-compose/graphql-compose-mongoose)
[![](https://img.shields.io/npm/v/graphql-compose-mongoose.svg)](https://www.npmjs.com/package/graphql-compose-mongoose)
[![npm](https://img.shields.io/npm/dt/graphql-compose-mongoose.svg)](http://www.npmtrends.com/graphql-compose-mongoose)
[![Join the chat at https://gitter.im/graphql-compose/Lobby](https://badges.gitter.im/graphql-compose/graphql-compose.svg)](https://gitter.im/graphql-compose/Lobby)
[![Commitizen friendly](https://img.shields.io/badge/commitizen-friendly-brightgreen.svg)](http://commitizen.github.io/cz-cli/)
[![Greenkeeper badge](https://badges.greenkeeper.io/graphql-compose/graphql-compose-mongoose.svg)](https://greenkeeper.io/)


This is a plugin for [graphql-compose](https://github.com/graphql-compose/graphql-compose), which derives GraphQLType from your [mongoose model](https://github.com/Automattic/mongoose). Also derives bunch of internal GraphQL Types. Provide all CRUD resolvers, including `graphql connection`, also provided basic search via operators ($lt, $gt and so on).

Installation
============
```
npm install graphql graphql-compose mongoose graphql-compose-mongoose --save
```
Modules `graphql`, `graphql-compose`, `mongoose` are in `peerDependencies`, so should be installed explicitly in your app. They have global objects and should not have ability to be installed as submodule.

If you want to add additional resolvers [`connection`](https://github.com/graphql-compose/graphql-compose-connection) and/or [`pagination`](https://github.com/graphql-compose/graphql-compose-pagination) - just install following packages and `graphql-compose-mongoose` will add them automatically.
```
npm install graphql-compose-connection graphql-compose-pagination --save
```

#### Different builds
This library contains different builds for any purposes:
```js
// Default import for using under node v6 and above
import { composeWithMongoose } from 'graphql-compose-mongoose';

// For using node 8 and above (native async/await)
import { composeWithMongoose } from 'graphql-compose-mongoose/node8';

// Source code without Flowtype declarations
import { composeWithMongoose } from 'graphql-compose-mongoose/es';
```

Example
=======
Live demo: [https://graphql-compose.herokuapp.com/](https://graphql-compose.herokuapp.com/)

Source code: https://github.com/graphql-compose/graphql-compose-mongoose-example

```js
import mongoose from 'mongoose';
import { composeWithMongoose } from 'graphql-compose-mongoose';
import { schemaComposer } from 'graphql-compose';

// STEP 1: DEFINE MONGOOSE SCHEMA AND MODEL
const LanguagesSchema = new mongoose.Schema({
  language: String,
  skill: {
    type: String,
    enum: [ 'basic', 'fluent', 'native' ],
  },
});

const UserSchema = new mongoose.Schema({
  name: String, // standard types
  age: {
    type: Number,
    index: true,
  },
  languages: {
    type: [LanguagesSchema], // you may include other schemas (here included as array of embedded documents)
    default: [],
  },
  contacts: { // another mongoose way for providing embedded documents
    email: String,
    phones: [String], // array of strings
  },
  gender: { // enum field with values
    type: String,
    enum: ['male', 'female', 'ladyboy'],
  },
  someMixed: {
    type: mongoose.Schema.Types.Mixed,
    description: 'Can be any mixed type, that will be treated as JSON GraphQL Scalar Type',
  },
});
const UserModel = mongoose.model('UserModel', UserSchema);



// STEP 2: CONVERT MONGOOSE MODEL TO GraphQL PIECES
const customizationOptions = {}; // left it empty for simplicity, described below
const UserTC = composeWithMongoose(UserModel, customizationOptions);

// STEP 3: CREATE CRAZY GraphQL SCHEMA WITH ALL CRUD USER OPERATIONS
// via graphql-compose it will be much much easier, with less typing
schemaComposer.rootQuery().addFields({
  userById: UserTC.getResolver('findById'),
  userByIds: UserTC.getResolver('findByIds'),
  userOne: UserTC.getResolver('findOne'),
  userMany: UserTC.getResolver('findMany'),
  userCount: UserTC.getResolver('count'),
  userConnection: UserTC.getResolver('connection'),
  userPagination: UserTC.getResolver('pagination'),
});

schemaComposer.rootMutation().addFields({
  userCreate: UserTC.getResolver('createOne'),
  userUpdateById: UserTC.getResolver('updateById'),
  userUpdateOne: UserTC.getResolver('updateOne'),
  userUpdateMany: UserTC.getResolver('updateMany'),
  userRemoveById: UserTC.getResolver('removeById'),
  userRemoveOne: UserTC.getResolver('removeOne'),
  userRemoveMany: UserTC.getResolver('removeMany'),
});

const graphqlSchema = schemaComposer.buildSchema();
export default graphqlSchema;
```
That's all!
You think that is to much code?
I don't think so, because by default internally was created about 55 graphql types (for input, sorting, filtering). So you will need much much more lines of code to implement all these CRUD operations by hands.


FAQ
===
### Can I get generated vanilla GraphQL types?
```js
const UserTC = composeWithMongoose(UserModel);
UserTC.getType(); // returns GraphQLObjectType
UserTC.getInputType(); // returns GraphQLInputObjectType, eg. for args
UserTC.get('languages').getType(); // get GraphQLObjectType for nested field
UserTC.get('fieldWithNesting.subNesting').getType(); // get GraphQL type of deep nested field
```

### How to add custom fields?
```js
UserTC.addFields({
  lonLat: TypeComposer.create('type LonLat { lon: Float, lat: Float }'),
  notice: 'String', // shorthand definition
  noticeList: { // extended
    type: '[String]', // String, Int, Float, Boolean, ID, Json
    description: 'Array of notices',
    resolve: (source, args, context, info) => 'some value',
  },
  bio: {
    type: GraphQLString,
    description: 'Providing vanilla GraphQL type'
  }
})
```

### How to build nesting/relations?
Suppose you Model has `friendsIds` field with array of user ids. So let build some relations:
```js
UserTC.addRelation(
  'friends',
  () => ({
    resolver: UserTC.getResolver('findByIds'),
    args: { // resolver `findByIds` has `_ids` arg, let provide value to it
      _ids: (source) => source.friendsIds,
    },
    projection: { friendsIds: 1 }, // point fields in source object, which should be fetched from DB
  })
);
UserTC.addRelation(
  'adultFriendsWithSameGender',
  () => ({
    resolver: UserTC.get('$findMany'), // shorthand for `UserTC.getResolver('findMany')`
    args: { // resolver `findMany` has `filter` arg, we may provide mongoose query to it
      filter: (source) => ({
        _operators : { // Applying criteria on fields which have
                       // operators enabled for them (by default, indexed fields only)
          _id : { in: source.friendsIds },
          age: { gt: 21 }
        },
        gender: source.gender,
      }),
      limit: 10,
    },
    projection: { friendsIds: 1, gender: 1 }, // required fields from source object
  })
);
```
### Reusing the same mongoose Schema in embedded object fields
Suppose you have a common structure you use as embedded object in multiple Schemas.
Also suppose you want the structure to have the same GraphQL type across all parent types.
(For instance, to allow reuse of fragments for this type)
Here are Schemas to demonstrate:
```js
import { Schema } from 'mongoose';

const ImageDataStructure = Schema({
  url: String,
  dimensions : {
    width: Number,
    height: Number
  }
}, { _id: false });

const UserProfile = Schema({
  fullName: String,
  personalImage: ImageDataStructure
});

const Article = Schema({
  title: String,
  heroImage: ImageDataStructure
});
```
If you want the `ImageDataStructure` to use the same GraphQL type in both `Article` and `UserProfile` you will need create it as a mongoose schema (not a standard javascript object) and to explicitly tell `graphql-compose-mongoose` the name you want it to have. Otherwise, without the name, it would generate the name according to the first parent this type was embedded in.

Do the following:
```js
import { schemaComposer } from 'graphql-compose'; // get the default schemaComposer or your created schemaComposer
import { convertSchemaToGraphQL } from 'graphql-compose-mongoose';

convertSchemaToGraphQL(ImageDataStructure, 'EmbeddedImage', schemaComposer); // Force this type on this mongoose schema
```
Before continuing to convert your models to TypeComposers:
```js
import mongoose from 'mongoose';
import { composeWithMongoose } from 'graphql-compose-mongoose';

const UserProfileModel = mongoose.model('UserProfile', UserProfile);
const ArticleModel = mongoose.model('Article', Article);

const UserProfileTC = composeWithMongoose(UserProfileModel);
const ArticleTC = composeWithMongoose(ArticleModel);
```
Then, you can use queries like this:
```graphql
query {
  topUser {
    fullName
    personalImage {
      ...fullImageData
    }
  }
  topArticle {
    title
    heroImage {
      ...fullImageData
    }
  }
}
fragment fullImageData on EmbeddedImage {
  url
  dimensions {
    width height
  }
}
```

Customization options
=====================
When we convert model `const UserTC = composeWithMongoose(UserModel, customizationOptions);` you may tune every piece of future derived types and resolvers.

### Here is flow typed definition of this options:

The top level of customization options. Here you setup name and description for the main type, remove fields or leave only desired fields.
```js
export type typeConverterOpts = {
  name?: string,
  description?: string,
  fields?: {
    only?: string[],
    remove?: string[],
  },
  inputType?: typeConverterInputTypeOpts,
  resolvers?: false | typeConverterResolversOpts,
};
```

This is `opts.inputType` level of options for default InputTypeObject which will be provided to all resolvers for `filter` and `input` args.
```js
export type typeConverterInputTypeOpts = {
  name?: string,
  description?: string,
  fields?: {
    only?: string[],
    remove?: string[],
    required?: string[]
  },
};
```

This is `opts.resolvers` level of options.
If you set the option to `false` it will disable resolver or some of its input args.
Every resolver's arg has it own options. They described below.
```js
export type typeConverterResolversOpts = {
  findById?: false,
  findByIds?: false | {
    limit?: limitHelperArgsOpts | false,
    sort?: sortHelperArgsOpts | false,
  },
  findOne?: false | {
    filter?: filterHelperArgsOpts | false,
    sort?: sortHelperArgsOpts | false,
    skip?: false,
  },
  findMany?: false | {
    filter?: filterHelperArgsOpts | false,
    sort?: sortHelperArgsOpts | false,
    limit?: limitHelperArgsOpts | false,
    skip?: false,
  },
  updateById?: false | {
    record?: recordHelperArgsOpts | false,
  },
  updateOne?: false | {
    record?: recordHelperArgsOpts | false,
    filter?: filterHelperArgsOpts | false,
    sort?: sortHelperArgsOpts | false,
    skip?: false,
  },
  updateMany?: false | {
    record?: recordHelperArgsOpts | false,
    filter?: filterHelperArgsOpts | false,
    sort?: sortHelperArgsOpts | false,
    limit?: limitHelperArgsOpts | false,
    skip?: false,
  },
  removeById?: false,
  removeOne?: false | {
    filter?: filterHelperArgsOpts | false,
    sort?: sortHelperArgsOpts | false,
  },
  removeMany?: false | {
    filter?: filterHelperArgsOpts | false,
  },
  createOne?: false | {
    record?: recordHelperArgsOpts | false,
  },
  count?: false | {
    filter?: filterHelperArgsOpts | false,
  },
  connection?: false | {
    uniqueFields: string[],
    sortValue: mixed,
    directionFilter: (<T>(filterArg: T, cursorData: CursorDataType, isBefore: boolean) => T),
  },
  pagination?: false | {
    perPage?: number,
  },
};
```

This is `opts.resolvers.[resolverName].[filter|sort|record|limit]` level of options.
You may tune every resolver's args independently as you wish.
Here you may setup every argument and override some fields from the default input object type, described above in `opts.inputType`.
```js
export type filterHelperArgsOpts = {
  filterTypeName?: string, // type name for `filter`
  isRequired?: boolean, // set `filter` arg as required (wraps in GraphQLNonNull)
  onlyIndexed?: boolean, // leave only that fields, which is indexed in mongodb
  requiredFields?: string | string[], // provide fieldNames, that should be required
  operators?: filterOperatorsOpts | false, // provide filtering fields by operators, eg. $lt, $gt
                                           // if left empty - provides all operators on indexed fields
};

// supported operators names in filter `arg`
export type filterOperatorNames =  'gt' | 'gte' | 'lt' | 'lte' | 'ne' | 'in[]' | 'nin[]';
export type filterOperatorsOpts = { [fieldName: string]: filterOperatorNames[] | false };

export type sortHelperArgsOpts = {
  sortTypeName?: string, // type name for `sort`
};

export type recordHelperArgsOpts = {
  recordTypeName?: string, // type name for `record`
  isRequired?: boolean, // set `record` arg as required (wraps in GraphQLNonNull)
  removeFields?: string[], // provide fieldNames, that should be removed
  requiredFields?: string[], // provide fieldNames, that should be required
};

export type limitHelperArgsOpts = {
  defaultValue?: number, // set your default limit, if it not provided in query (default: 1000)
};
```

Used plugins
============
### [graphql-compose-connection](https://github.com/graphql-compose/graphql-compose-connection)
This plugin adds `connection` resolver. Build in mechanism allows sort by any unique indexes (not only by id). Also supported compound sorting (by several fields).

Besides standard connection arguments `first`, `last`, `before` and `after`, also added great arguments:
* `filter` arg - for filtering records
* `sort` arg - for sorting records

This plugin completely follows to [Relay Cursor Connections Specification](https://facebook.github.io/relay/graphql/connections.htm).

### [graphql-compose-pagination](https://github.com/graphql-compose/graphql-compose-pagination)
This plugin adds `pagination` resolver.

License
=======
[MIT](https://github.com/graphql-compose/graphql-compose-mongoose/blob/master/LICENSE.md)

## This is example app of `graphql-compose`

[![Travis](https://img.shields.io/travis/nodkz/graphql-compose-examples.svg?maxAge=2592000)](https://travis-ci.org/nodkz/graphql-compose-examples)
[![Greenkeeper badge](https://badges.greenkeeper.io/nodkz/graphql-compose-examples.svg)](https://greenkeeper.io/)

Live example on Heroku: [https://graphql-compose.herokuapp.com/](https://graphql-compose.herokuapp.com/)

```
npm install
npm run seed && npm run start:watch
open http://localhost:3000
```


## User: simple schema with one type
This [example](https://github.com/nodkz/graphql-compose-examples/tree/master/examples/user) has simple User mongoose model that supports bunch of CRUD operations.

```js
const UserSchema = new mongoose.Schema({
  name: String, // standard types
  age: {
    type: Number,
    index: true,
  },
  languages: {
    type: [LanguagesSchema], // you may include other schemas (here included as array of embedded documents)
    default: [],
  },
  contacts: { // another mongoose way for providing embedded documents
    email: String,
    phones: [String], // array of strings
  },
  gender: { // enum field with values
    type: String,
    enum: ['male', 'female', 'ladyboy'],
  },
});
```

<img width="982" alt="screen shot 2016-07-03 at 15 23 03" src="https://cloud.githubusercontent.com/assets/1946920/16544733/9ef9b146-4132-11e6-8a90-8702d2474cfd.png">

<img width="1330" alt="screen shot 2016-07-15 at 12 41 17" src="https://cloud.githubusercontent.com/assets/1946920/16865833/7ec028c8-4a89-11e6-980e-e17745e5085c.png">


## User for Relay: simple schema with one type

This [schema](https://github.com/nodkz/graphql-compose-examples/tree/master/examples/userForRelay) shows all available CRUD operations which are compatible with Relay. It uses `graphql-compose-mongose` and `graphql-compose-relay`:
- `composeWithRelay(RootQueryTC)` adds `node` field to the RootQuery. Via `RootQuery.node(id)` you may find objects by globally unique ID among all types.
- `composeWithRelay(UserTC)` - modify `UserTC` generated by `graphql-compose-mongoose`  
  - adds `id` field with Relay's globally unique ID
  - this type will be added to `NodeInterface` for resolving via `RootQuery.node`
  - for mutations will be added `clientMutationId` to input and output objects types
  - also all arguments in mutations will be moved into `input` arg

<img width="1362" alt="screen shot 2017-03-13 at 10 20 34" src="https://cloud.githubusercontent.com/assets/1946920/23841356/d3bd6f42-07d6-11e7-94f5-cc3618eaf45a.png">


## Northwind: complex schema with 8 models 🌶🌶🌶

This is a sample data of some trading company, which consists from 8 models. All models has cross-relations to each other. This schema used in the Relay example app: [Server schema code](https://github.com/nodkz/graphql-compose-examples/tree/master/examples/northwind), [Client app code](https://github.com/nodkz/relay-northwind-app), [Live demo of client](https://nodkz.github.io/relay-northwind/).

![relay-northwind-app](https://cloud.githubusercontent.com/assets/1946920/18013918/488e6830-6be2-11e6-84b6-884c8ab971ac.gif)


## Elasticsearch REST API wrapper

This [schema](https://github.com/nodkz/graphql-compose-examples/tree/master/examples/elasticsearch) uses [graphql-compose-elasticsearch](https://github.com/nodkz/graphql-compose-elasticsearch) module and  provides full API available in the official elasticsearch module.

<img width="1316" alt="screen shot 2017-03-07 at 22 26 17" src="https://cloud.githubusercontent.com/assets/1946920/23841396/2c123b3c-07d7-11e7-8c83-ff01c98090fb.png">