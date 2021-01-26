---
title: Graphql defer example part I
date: "2021-01-22T22:12:03.284Z"
description: "Demonstrating @defer directive in graphql with NodeJS"
---

Graphql comes with handy capabilities when developing end user facing applications and in this blog post I'm going to talk about ```@defer``` directive and demonstrate it with a working example.


If you want to follow along with me checkout my example github repo [graphql-defer-example](https://github.com/dilansri/graphql-defer-example)


Both ```@defer``` and ```@stream``` directives are still under proposal stage as of this writing and with [Apollo](https://www.apollographql.com/blog/introducing-defer-in-apollo-server-f6797c4e9d6e/) its available as a experimental feature.


With standard graphql request/reponse model, if you are requesting large amount of data from your graphql query, the client has to wait until all the fields are resolved in graphql server to get a response.

With the ```@defer``` directive you can de-prioritize part of your data in your query so that they'll be available to the client at a later stage.

To achieve this we can use standard ```multipart``` response from our express server.

In this example, I'll be using 
- [`graphql-helix`](https://github.com/contrawork/graphql-helix) to process and execute the graphql server request. [Checkout this awesome post about graphql-helix](https://dev.to/danielrearden/building-a-graphql-server-with-graphql-helix-2k44)
  
- [`nexus`](https://github.com/graphql-nexus/nexus) to create the typed schema

- [`graphql@experimental-stream-defer`](https://www.npmjs.com/package/graphql/v/15.4.0-experimental-stream-defer.1) graphql experimental published package.

Refer to [graphql-defer-example](https://github.com/dilansri/graphql-defer-example) repo if you want to examine the complete code.

#### 1. Creating an express server with ```/graphql``` endpoint

```javascript
import express from 'express'
const app = express()
const PORT = 8000

app.use(express.json())

app.use('/graphql', async (req, res) => {

  return res.send('ok')
  
})

app.listen(PORT, () => {
  console.log(`⚡️[server]: Server is running at https://localhost:${PORT}`)
})
```

#### 2. Setup the typed graphql schema with nexus

```javascript
import { makeSchema, queryType, objectType, list } from 'nexus'
import * as path from 'path'
import { products } from './resolvers'
...

const Query = queryType({
  definition(t) {
    t.field('product', {
      type: Product,
      resolve: () => ({id: 'x', name: 'y', images: []}) // return static since we will focus more on products query
    })
    t.field('products', {
      type: list(Product),
      resolve: products
    })
  },
})

const Image = objectType({
  name: 'Image',
  definition(t) {
    t.string('url')
  }
})

const Product = objectType({
  name: 'Product',
  description: 'Defines a single product',
  definition(t) {
    t.nonNull.string('id')
    t.string('name')
    t.field('images', {
      type: list(Image)
    })
    t.field('relatedProducts', {
      type: list(Product),
    })
  }
})

export const schema = makeSchema({
  types: [Query, Product, Image],
  outputs: {
    schema: path.join(__dirname, '../generated/schema.graphql'),
    typegen: path.join(__dirname, '../generated/typings.ts'),
  },
})

```

Here we have the ```products``` resolver that will be resolved to a static data included in the repository.

#### 3. Mimick the slow  resolver so we can experiment @defer directive

```javascript
...
const Product = objectType({
  name: 'Product',
  description: 'Defines a single product',
  definition(t) {
    t.nonNull.string('id')
    t.string('name')
    t.field('images', {
      type: list(Image)
    })
    t.field('relatedProducts', {
      type: list(Product),
    })
    t.field('comments', {
      type: list('String'),
      resolve: async ({ id }) => new Promise((resolve) => {
        const productComments = comments(id) || null
        setTimeout(() => resolve(productComments), 1000) // settimeout for testing deferring on comments
      })
    })
  }
})
...

```

Notice the ```comments``` field that will be resolved after 1 second when we request it with our products.


#### 4. Add graphql-helix to process the incoming graphql queries

```javascript
import express from 'express'
import { 
  getGraphQLParameters, 
  processRequest, 
  shouldRenderGraphiQL, 
  renderGraphiQL 
} from 'graphql-helix'
import { schema } from './graphql/schema'
....

app.use('/graphql', async (req, res) => {

  const {
    query,
    variables,
    operationName
  } = getGraphQLParameters(req)

  const result = await processRequest({
    schema,
    query,
    variables,
    operationName,
    request: req,
  })

})

...

```

The result you'll get from ```graphql-helix``` has 3 types.
- RESPONSE - when you have a query without ```@defer, @stream``` directives
- PUSH - when you have a graphql subscription
- MULTIPART_RESPONSE - when your query includes ```@defer or @stream``` directive

Since we are interested in **MULTIPART_RESPONSE** lets process it.


#### 5. Process and send a response with a multipart response

```javascript
...
if(result.type === 'MULTIPART_RESPONSE') {
    
  req.on('close', () => {
    result.unsubscribe()
  })

  const response = new MultipartResponse(res, 'mixed', 'gcgfr')

  await result.subscribe((res) => {
    response.add(res)
  })

  return response.end()
}
...

```

Since there are no proper multipart response builder available, for this example I created a basic [MultipartResponse class](https://github.com/dilansri/graphql-defer-example/blob/main/server/src/MultipartResponse.ts) for express.

The important things to notice are that we use express ```response.writeHead``` initially and then ```response.write``` methods for subsequent responses that we will be sent to client as patches. And finally, ```response.end``` at the end to indicate to close the connection.



#### 6. Lets add graphiql test out our defer example

```graphql-helix``` comes with nice and easy function to mount **graphiql** based the browser request.

```javascript
...
app.use('/graphql', async (req, res) => {

  ...

  if(shouldRenderGraphiQL(req)) {
    return res.send(renderGraphiQL({defaultQuery: exampleQuery}))
  }

  ...

})
...

```

defaultQuery is the one that get populated on your graphiql interface as the default.


#### 7. Try out a defer query in graphiql interface

```graphql

fragment Comments on Product{
  comments
}

fragment RelatedProducts on Product {
  relatedProducts {
    id
    name
  }
}

query {
  products{
    id
    ...Comments @defer
    ...RelatedProducts @defer
  }
}

```

Here we define two fragments ```Comments, RelatedProducts``` so we can apply ```@defer``` directive to them.

#### 8. Lets examine the response we get 

```
--gcgfr
Content-Type: application/json; charset=utf-8
Content-Length: 71

{"data":{"products":[{"id":"1"},{"id":"2"},{"id":"3"}]},"hasNext":true}

--gcgfr
Content-Type: application/json; charset=utf-8
Content-Length: 127

{"data":{"relatedProducts":[{"id":"2","name":"product-2"},{"id":"3","name":"product-3"}]},"path":["products",0],"hasNext":true}

--gcgfr
Content-Type: application/json; charset=utf-8
Content-Length: 97

{"data":{"relatedProducts":[{"id":"1","name":"product-1"}]},"path":["products",1],"hasNext":true}

--gcgfr
Content-Type: application/json; charset=utf-8
Content-Length: 97

{"data":{"relatedProducts":[{"id":"1","name":"product-1"}]},"path":["products",2],"hasNext":true}

--gcgfr
Content-Type: application/json; charset=utf-8
Content-Length: 91

{"data":{"comments":["good review 1","bad review 1"]},"path":["products",0],"hasNext":true}

--gcgfr
Content-Type: application/json; charset=utf-8
Content-Length: 90

{"data":{"comments":["average review 1","review 1"]},"path":["products",1],"hasNext":true}

--gcgfr
Content-Type: application/json; charset=utf-8
Content-Length: 80

{"data":{"comments":["average review 2"]},"path":["products",2],"hasNext":false}
```

We see 7 parts in our reponse each seperated with the boundary ```--gcgfr ``` each which arrived as a chunk to the client seperately.

Also in the graphiql interface its visible thats the comments are appearing after few seconds.

![Defer Result](./graphl_defer.gif)


Thank you for reading. In the part II, I'll demonstrate how to consume the ```@defer``` functionality in a graphql client.

Feel free to reach out.