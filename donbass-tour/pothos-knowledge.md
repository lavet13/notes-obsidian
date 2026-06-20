---
id: knowledge
aliases:
  - knowledge
tags: []
---

**Pothos GraphQL**

---

# prisma plugin

`t.prismaField` is used within root object types(`Query` and `Mutation`) and has
additional first argument query which is used for pre-loading data needed to
resolve nested parts of the current query.

`builder.prismaObject` is used for getting the type info without using object refs or
needing imports from prisma client.

`builder.prismaObjectField(s)` methods can be used to extend prisma objects,
and is preferrable than `builder.objectField(s)` which don't support using
**selections**, or **exposing fields not in the default selection.**

`t.relation` is used within `builder.prismaObject` or `builder.prismaNode`, it
defines a field that can be pre-loaded by a parent resolver. This will create
something like
```typescript
{
    include: {
        author: true
    }
}
```
that will be passed as part of the query argument of a `t.prismaField` or
`builder.prismaNode` resolver. If the parent is another `relation` field, the
includes become nested, and the full relation chain will be passed to the
`t.prismaField` that started the chain.

For example the query:
```typescript
query {
  me {
    posts {
      author {
        id
      }
    }
  }
}
```

the `me` `t.prismaField` would receive something like the following as its query
parameter:

```typescript
{
  include: {
    posts: {
      include: {
        author: true;
      }
    }
  }
}
```

**Filters, Sorting, and arguments**

So far we have been describing very simple queries without any arguments,
filtering, or sorting. For `t.prismaField` definitions, you can add arguments to
your field like normal, and pass them directly into your Prisma query inside the
resolver. The resolver receives a `query` parameter that contains Pothos's pre-load
instructions for nested relations, which you must spread into your Prisma call to
keep nested fields working efficiently:

```typescript
posts: t.prismaField({
  type: ['Post'],
  args: { published: t.arg.boolean() },
  resolve: (query, root, args) => {
    return prisma.post.findMany({
      ...query, // Required to preserve nested preloads
      where: { published: args.published },
      orderBy: { createdAt: 'desc' }
    })
  }
})
```
For `t.relation`, the flow is different because you aren't writing the Prisma
query yourself. Instead, you add a query option to your field that returns Prisma
modifiers based on the arguments:
```typescript
posts: t.relation('posts', {
  args: { published: t.arg.boolean() },
  query: (args) => ({
    where: { published: args.published },
    orderBy: { createdAt: 'desc' },
    take: 10
  })
})
```
In short: `t.prismaField` gives you full control over the database call, so you
apply filters and sorting directly in the resolver while always preserving the
`...query` spread. `t.relation` delegates the query to Pothos, so you supply filters
via the `query` option, which returns only the modifiers (`where`, `orderBy`, `skip`,
`take`) that Pothos automatically merges into the parent's fetch chain. You never
need to manage `include` or `select` manually in a relation, as Pothos builds them
based on the GraphQL request.

`t.relationCount`
Prisma supports querying for relation counts, which lets you include counts for
related records alongside your normal data fetches. Pothos exposes this via
`t.relationCount`, which works similarly to `t.relation` but returns a number
instead of the related objects.

```typescript
builder.prismaObject('User', {
  fields: (t) => ({
    id: t.exposeID('id'),
    postCount: t.relationCount('posts', {
      where: { published: true }
    }),
  }),
})
```

This generates a query like
```typescript
{
    include: {
        _count: {
            select: {
                posts: {
                    where: {
                        published: true
                    }
                }
            }
        }
    }
}
```
that gets merged into the parent `t.prismaField's` query parameter. The count is
resolved automatically without requiring a separate database call, keeping your
queries efficient.

**Important notes:**

---

- `t.relationCount` only returns a **number** — you cannot select fields from the
related objects, only count them.
- The `where` option works like Prisma's normal `where`, but remember it filters what
gets counted, not what gets returned.
- If you need both the related objects and a filtered count of them, you can use
`t.relation` and `t.relationCount` side-by-side on the same relation, each with
their own query/where options.
```typescript
// Get both published posts and the total count of drafts
fields: (t) => ({
  publishedPosts: t.relation('posts', {
    query: { where: { published: true } }
  }),
  draftCount: t.relationCount('posts', {
    where: { published: false }
  })
})
```
**In short:** use `t.relationCount` when you need a numeric count of related records
without fetching the records themselves. It integrates seamlessly with Pothos's
pre-loading system, supports filtering via where


**Optimized queries without `t.prismaField` with a `queryFromInfo` helper**

## Select mode for types
**By default prisma plugin use include when including relations, use select to
select as few columns as possible**
To do this, you can add a select instead of an include to your prismaObject:
```typescript
builder.prismaObject('User', {
  select: {
    id: true,
  },
  fields: (t) => ({
    id: t.exposeID('id'),
    email: t.exposeString('email'),
  }),
});
```

The `t.expose*` and `t.relation` methods will all automatically add selections for
the exposed fields WHEN THEY ARE QUERIED, ensuring that only the requested
columns will be loaded from the database.

In addition to the `t.expose` and `t.relation`, you can also add custom selections
to other fields:
```typescript
builder.prismaObject('User', {
  select: {
    id: true,
  },
  fields: (t) => ({
    id: t.exposeID('id'),
    email: t.exposeString('email'),
    bio: t.string({
      // This will select user.profile.bio when the `bio` field is queried
      select: {
        profile: {
          select: {
            bio: true,
          },
        },
      },
      resolve: (user) => user.profile.bio,
    }),
  }),
});
```

## Connections
`t.prismaConnection`

