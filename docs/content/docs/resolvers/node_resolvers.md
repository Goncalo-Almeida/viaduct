---
title: Node Resolvers
description: Writing resolvers for nodes in Viaduct
weight: 1
---

## Schema

Nodes are types that are resolvable by ID and implement the `Node` interface. Every object type that implements the `Node` interface has a corresponding node resolver.

```graphql
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  firstName: String
  lastName: String
  displayName: String @resolver
}
```

## Generated base class

Viaduct generates an abstract base class for all object types that implement Node. For the `User` example above, Viaduct generates the following code:

```kotlin
object NodeResolvers {
  abstract class User {
    open suspend fun resolve(ctx: Context): viaduct.api.grts.User =
      throw NotImplementedError()

    open suspend fun batchResolve(contexts: List<Context>): List<FieldValue<viaduct.api.grts.User>> =
      throw NotImplementedError()

    class Context: NodeExecutionContext<viaduct.api.grts.User>
  }

  // If there were more nodes, their base classes would be generated here
}
```

The nested `Context` class is described in more detail [below](#context).

## Implementation

Implement a node resolver by subclassing the generated base class, and overriding exactly one of either `resolve` or `batchResolve`. Learn more about batch resolution [here](/docs/resolvers/batch_resolution/).

Here's an example of a non-batching resolver for `User` that calls a user service to get data for a single user:

```kotlin
class UserNodeResolver @Inject constructor(
  val userService: UserServiceClient
): NodeResolvers.User() {
  override suspend fun resolve(ctx: Context): User {
    val data = userService.fetch(ctx.id.internalId)
    return User.builder(ctx)
      .firstName(data.firstName)
      .lastName(data.lastName)
      .build()
  }
}
```

Alternatively, if the user service provides a batch endpoint, you can implement a batch node resolver:

```kotlin
class UserNodeResolver @Inject constructor(
  val userService: UserServiceClient
): NodeResolvers.User() {
  override suspend fun batchResolve(contexts: List<Context>): List<FieldValue<User>> {
    val ids = contexts.map { it.id.internalId }
    val responses = userService.fetch(ids)
    return responses.map { response ->
      FieldValue.ofValue(
        User.builder(ctx)
          .firstName(response.firstName)
          .lastName(response.lastName)
          .build()
      )
    }
  }
}
```

Points illustrated by this example:

* Dependency injection can be used to provide access to values beyond what’s in the execution context.
* As mentioned previously, when building the [GRT](/docs/generated_code/) for a GraphQL object type, you do *not* have to provide values for all fields, and indeed should not provide values for fields outside of the resolver's responsibility set (e.g. the `displayName` field in our example).

## Context

Both `resolve` and `batchResolve` take `Context` objects as input. This class is an instance of `NodeExecutionContext`:

```kotlin
interface NodeExecutionContext<T: NodeObject>: ResolverExecutionContext {
  val id: GlobalID<T>
  fun selections(): SelectionSet<T>
}
```
For the example `User` type, the `T` type would be the user [GRT](/docs/generated_code/).

`NodeExecutionContext` includes the ID of the node to be resolved, and the selection set for the node being requested by the query. Most node resolvers are not "selective," i.e., they ignore this selection set and thus don’t call this function. In this case, as discussed above, it’s important that the node resolver returns its entire responsibility set.

**Advanced Users:** If the `selections` function is *not* called by an invocation of a resolver, then the engine will assume that invocation will return the full responsibility set of the resolver and may take actions based on that assumption.  If a resolver is going to be selective, then it **must** call this function to get its selection set rather than obtain it through some other means.

Since `NodeExecutionContext` implements `ResolverExecutionContext`, it also includes the utilities provided there, which allow you to:
* Execute [subqueries](/docs/resolvers/subqueries)
* Construct [node references](/docs/resolvers/node_references)
* Construct [GlobalIDs](/docs/globalids)

## Responsibility set

The node resolver is responsible for resolving all fields, including nested fields, without its own resolver. These are typically core fields that are stored together and can be efficiently retrieved together.

In the example above, the node resolver for `User` is responsible for returning the `firstName` and `lastName` fields, but not the `displayName` field, which has its own resolver. Note that node resolvers are *not* responsible for the `id` field, since the ID is an input to the node resolver.

Node resolvers are also responsible for determining whether the node exists. If a node resolver returns an error value, the entire node in the GraphQL response will be null, not just the fields in the node resolver's responsibility set.
