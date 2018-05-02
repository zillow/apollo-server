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

Since clients only get exactly what they ask for from GraphQL queries, adding new fields, queries, or mutations won't introduce any new breaking changes. You don't need to do anything!

## Field Rollover

Field rollover is a term given to an API change that's an evolution of a field, such as a rename or a change in arguments. Some of these changes can be really small, so versioning for any such change could create many versions quickly, and make your API hard to manage. We'll go over these two kinds of field rollover separately and show how to make these changes safely.

### Renaming a field

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

Sometimes we want to keep a field, but change how clients use it by adjusting its variables and/or return type. For example, if we had a `getUserData` query we used to fetch a single user's data, but wanted to change the arguments to _only_ support a list of `ids` instead of a single `id` and return a list of users.

```graphql
type Query {
  # what we have
  getUserInfo(id: ID!): User

  # what we want to end up with
  getUser(ids: [ID!]!): [User]!
}
```

Similar to how we incrementally moved a field to a new name in the previous section, we can take this process slowly and prevent any breaking changes.

The first thing we would do is add in the second argument side-by-side with the old one. Since _one_ of the two arguments is required, both have to be optional for now (we will enforce that exactly one of them exists in our resolver). Our return type would be a union of a single user or our list of users.

```js
const typeDefs = gql`
  union UserOrListOfUsers = User | [User]
  type Query {
    getUserInfo(id: ID, ids: [ID!]): UserOrListOfUsers
  }
`;

const resolvers = {
  Query: {
    getUserInfo: (root, args, context) => {
      if((!args.id && !args.ids) || (args.id && args.ids))
        throw new Error('You must provide EITHER id or a list of ids');

      const users = context.User.getByIds(args.id ? [args.id] : args.ids);

      // if they passed 'id' they're expecting a single user, not a list
      return args.id ? users[0] || null : users;
    }
  }
};
```
