# Awesome GraphQL Client

GraphQL Client with file upload support for NodeJS and browser

![CI/CD](https://github.com/lynxtaa/awesome-graphql-client/workflows/CI/CD/badge.svg) [![npm version](https://badge.fury.io/js/awesome-graphql-client.svg)](https://badge.fury.io/js/awesome-graphql-client) [![Codecov](https://img.shields.io/codecov/c/github/lynxtaa/awesome-graphql-client)](https://codecov.io/gh/lynxtaa/awesome-graphql-client)

## Features

- [GraphQL File Upload](https://github.com/jaydenseric/graphql-multipart-request-spec) support
- Works in browsers and NodeJS
- Small size (around 1.5Kb gzipped)
- Full Typescript support
- Supports queries generated by [graphql-tag](https://www.npmjs.com/package/graphql-tag)
- Supports GraphQL GET requests
- Perfect for React apps in combination with [react-query](https://www.npmjs.com/package/react-query). See [Next.js example](https://github.com/lynxtaa/awesome-graphql-client/tree/master/examples/next.js)

## Install

```sh
npm install awesome-graphql-client
```

## Quick Start

### Browser

```js
import { AwesomeGraphQLClient } from 'awesome-graphql-client'

const client = new AwesomeGraphQLClient({ endpoint: '/graphql' })

// Also query can be an output from graphql-tag (see examples below)
const GetUsers = `
  query getUsers {
    users {
      id
    }
  }
`

const UploadUserAvatar = `
  mutation uploadUserAvatar($userId: Int!, $file: Upload!) {
    updateUser(id: $userId, input: { avatar: $file }) {
      id
    }
  }
`

client
  .request(GetUsers)
  .then((data) =>
    client.request(UploadUserAvatar, {
      id: data.users[0].id,
      file: document.querySelector('input#avatar').files[0],
    }),
  )
  .then((data) => console.log(data.updateUser.id))
  .catch((error) => console.log(error))
```

### NodeJS

```js
const { AwesomeGraphQLClient } = require('awesome-graphql-client')
const FormData = require('form-data')
const { createReadStream } = require('fs')
const fetch = require('node-fetch')

const client = new AwesomeGraphQLClient({
  endpoint: 'http://localhost:8080/graphql',
  fetch,
  FormData, // Required only if you're using file upload
})

// Also query can be an output from graphql-tag (see examples below)
const UploadUserAvatar = `
  mutation uploadUserAvatar($userId: Int!, $file: Upload!) {
    updateUser(id: $userId, input: { avatar: $file }) {
      id
    }
  }
`

client
  .request(UploadUserAvatar, { file: createReadStream('./avatar.img'), userId: 10 })
  .then((data) => console.log(data.updateUser.id))
  .catch((error) => console.log(error))
```

## Table of Contents

- API
  - [AwesomeGraphQLClient](#awesomegraphqlclient)
  - [GraphQLRequestError](#graphqlrequesterror)
  - [gql](#approach-2-use-fake-graphql-tag)
- Examples
  - [Typescript](#typescript)
  - [Error Logging](#error-logging)
  - [GraphQL GET Requests](#graphql-get-requests)
  - [GraphQL Tag](#graphql-tag)
  - [Cookies in NodeJS](#cookies-in-nodejs)
  - [More Examples](#more-examples)

## API

## `AwesomeGraphQLClient`

**Usage**:

```js
import { AwesomeGraphQLClient } from 'awesome-graphql-client'
const client = new AwesomeGraphQLClient(config)
```

### `config` properties

- `endpoint`: _string_ - The URL to your GraphQL endpoint (required)
- `fetch`: _Function_ - Fetch polyfill (necessary in NodeJS, see [example](#nodejs))
- `fetchOptions`: _object_ - Overrides for fetch options
- `FormData`: _object_ - FormData polyfill (necessary in NodeJS if you are using file upload, see [example](#nodejs))
- `formatQuery`: _function(query: any): string_ - Custom query formatter (see [example](#graphql-tag))
- `onError`: _function(error: GraphQLRequestError | Error): void_ - Provided callback will be called before throwing an error (see [example](#error-logging))

### `client` methods

- `client.setFetchOptions(fetchOptions: FetchOptions)`: Sets fetch options. See examples below
- `client.getFetchOptions()`: Returns current fetch options
- `client.getEnpoint(): string`: Returns current GraphQL endpoint
- `client.request(query, variables?, fetchOptions?): Promise<data>`: Sends GraphQL Request and returns data or throws an error
- `client.requestSafe(query, variables?, fetchOptions?): Promise<{ data, response } | { error }>`: Sends GraphQL Request and returns object with 'data' and 'response' fields or with a single 'error' field. See examples below. _Notice: this function never throws_.

## `GraphQLRequestError`

### `instance` fields

- `message`: _string_ - Error message
- `query`: _string_ - GraphQL query
- `variables`: _string | undefined_ - GraphQL variables
- `response`: _Response_ - response returned from fetch

## Examples

## Typescript

```ts
interface getUser {
  user: { id: number; login: string } | null
}
interface getUserVariables {
  id: 10
}

const query = `
  query getUser($id: Int!) {
    user {
      id
      login
    }
  }
`

client
  .request<getUser, getUserVariables>(query, { id: 10 })
  .then((data) => console.log(data))
  .catch((error) => console.log(error))

client
  .requestSafe<getUser, getUserVariables>(query, { id: 10 })
  .then((result) => {
    if ('error' in result) {
      throw result.error
    }
    console.log(`Status ${result.response.status}`, `Data ${result.data.user}`)
  })
```

## Error Logging

```js
import { AwesomeGraphQLClient, GraphQLRequestError } from 'awesome-graphql-client'

const client = new AwesomeGraphQLClient({
  endpoint: '/graphql',
  onError(error) {
    if (error instanceof GraphQLRequestError) {
      console.error(error.message)
      console.groupCollapsed('Operation:')
      console.log({ query: error.query, variables: error.variables })
      console.groupEnd()
    } else {
      console.error(error)
    }
  },
})
```

## GraphQL GET Requests

Internally it uses [URL API](https://developer.mozilla.org/ru/docs/Web/API/URL/URL). Consider [polyfilling URL standard](https://github.com/zloirock/core-js#url-and-urlsearchparams) for this feature to work in IE

```js
client
  .request(query, variables, { method: 'GET' })
  .then((data) => console.log(data))
  .catch((err) => console.log(err))
```

## GraphQL Tag

### Approach #1: Use `formatQuery`

```js
import { AwesomeGraphQLClient } from 'awesome-graphql-client'
import { DocumentNode } from 'graphql/language/ast'
import { print } from 'graphql/language/printer'
import gql from 'graphql-tag'

const client = new AwesomeGraphQLClient({
  endpoint: '/graphql',
  formatQuery: (query: DocumentNode | string) =>
    typeof query === 'string' ? query : print(query),
})

const query = gql`
  query me {
    me {
      login
    }
  }
`

client
  .request(query)
  .then((data) => console.log(data))
  .catch((err) => console.log(err))
```

### Approach #2: Use fake `graphql-tag`

Recommended approach if you're using `graphql-tag` only for syntax highlighting and static analysis such as linting and types generation. It has less computational cost and makes overall smaller bundles. GraphQL fragments are supported too.

```js
import { AwesomeGraphQLClient, gql } from 'awesome-graphql-client'

const client = new AwesomeGraphQLClient({ endpoint: '/graphql' })

const query = gql`
  query me {
    me {
      login
    }
  }
`

client
  .request(query)
  .then((data) => console.log(data))
  .catch((err) => console.log(err))
```

## Cookies in NodeJS

```js
const { AwesomeGraphQLClient } = require('awesome-graphql-client')
const nodeFetch = require('node-fetch')

const client = new AwesomeGraphQLClient({
  endpoint: 'http://localhost:8080/graphql',
  fetch: require('fetch-cookie')(nodeFetch),
})
```

## More Examples

[https://github.com/lynxtaa/awesome-graphql-client/tree/master/examples](https://github.com/lynxtaa/awesome-graphql-client/tree/master/examples)
