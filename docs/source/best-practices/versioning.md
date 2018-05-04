---
title: Versioning
description: How to add and remove parts of your schema without breaking your clients
---

<!--
  QUESTIONS:
  - is it possible to migrate the return value of a field from item -> [item]
  - is it possibe to add errors manually to signify a deprecated field, but still return data
 -->

If you've worked with REST APIs in the past, you probably recognized a pattern or two that was used to version the API. This is common in the endpoint routes, like `/api/v1` or `/api/v2`. As a result, you may wonder if and how GraphQL endpoints should be versioned. Luckily, GraphQL endpoints don't usually need versioning.

## Why versioning isn't needed

Versioning is used to prevent breaking changes from being pushed to clients consuming an API. In Rest APIs, such breaking changes may be as small as renaming a field or as large as refactoring the whole data model. GraphQL, however, is very resilient to most changes and provides strategies that can usually avoid a separately versioned API.

## Additive Changes

Since clients only get exactly what they ask for from GraphQL queries, adding new fields, arguments, queries, or mutations won't introduce any new breaking changes. You don't need to do anything!

<h2 id="field-rollover">Field Rollover</h2>

Field rollover is a term given to an API change that's an evolution of a field, such as a rename or a change in arguments. Some of these changes can be really small, so versioning for any such change could create many versions quickly, and make your API hard to manage. We'll go over these two kinds of field rollover separately and show how to make these changes safely.

<h3 id="renaming-or-removing-a-field">Renaming or removing a field</h3>

Suppose we have query called `user` as defined below.

```graphql
type Query {
  user(id: ID!): User
}
```

We may want to rename it to `getUser` to be more descriptive of what the query is for like so:

```graphql
type Query {
  getUser(id: ID!): User
}
```

If that was all we did, this would be a breaking change for clients, as any expecting a `user` query would get an error. Instead of renaming the one query outright, we can simply add a new query for `getUser` and leave the old `user` query. To prevent code duplication, we can share resolver logic between the two fields like this:

```js
const getUserResolver = (root, args, context) => {
  context.User.getById(args.id);
};

const resolvers = {
  Query: {
    getUser: getUserResolver,
    user: getUserResolver
  }
}
```

**Deprecating a field**

But that's not all! If we just left both fields active, there's no way that clients would know that they were supposed to use the new query. Luckily, the graphql spec details a schema directive called `@deprecated`.

```
type Query {
  user(id: ID!): User @deprecated(reason: "renamed to 'getUser'")
  getUser(id: ID!): User
}
```

Adding this directive to a field does a couple things to help steer clients away from using it:
- lets clients know a field is deprecated by showing visual hints in tools like GraphQL Playground
- hides the field from recommendations in tools like GraphQL Playground


**Retiring the deprecated field**

Over time, usage will fall for the deprecated field and grow for the new field. Using tools like [Apollo Engine](https://www.apollographql.com/engine), you can make an educated decision about when to retire the field based on usage data.

### Changing arguments

**Non-breaking argument changes**

Sometimes we want to keep a field, but change how clients use it by adjusting its variables. For example, if we had a `getUsers` query that we used to fetch user data based off of a list of user ids, but wanted to change the arguments to support a `groupId` to look up users of a group or filter the users requested by the `ids` argument to only return users in the group:

```graphql
type Query {
  # what we have
  getUsers(ids: [ID!]!): [User]!

  # what we want to end up with
  getUsers(ids: [ID!], groupId: ID!): [User]!
}
```

Since this is an additive change, and doesn't actually change the default behavior of the `getUsers` query, this isn't even a breaking change.

**Breaking changes**

An example of a breaking change on an argument would be renaming (or deleting) an argument.

```graphql
type query = {
  # what we have
  getUsers(ids: [ID!], groupId: ID!): [User]!

  # what we want to end up with
  getUsers(ids: [ID!], groupIds: [ID!]): [User]!
}
```

There's no way to mark an argument as deprecated, but there are a couple options. If we wanted to leave the old `groupId` argument active, we wouldn't need to do anything; adding a new argument isn't a breaking change as long as existing functionality doesn't change.

Instead of supporting it, if we wanted to remove the old argument, the safest option would be to create a new field and deprecate the current `getUsers` field. Using a monitoring tool, like Apollo Engine, you can tell when usage of an old field has dropped to a reasonable level and remove it. The earlier [field rollover](#field-rollover) section gives more info on how to do that.

