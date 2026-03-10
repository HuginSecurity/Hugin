# GraphQL API

Hugin exposes a GraphQL API primarily used by the desktop UI. It provides richer querying capabilities than the REST API for flow data -- multi-field filtering, sorting, pagination, and real-time subscriptions are all available in a single interface.

## Endpoint

```
POST /graphql     Execute queries and mutations
GET  /graphql     GraphQL Playground (interactive browser IDE)
```

The GraphQL Playground is accessible at `http://127.0.0.1:8081/graphql` in a browser when the API server is running.

## Schema

The schema is built with [async-graphql](https://github.com/async-graphql/async-graphql) and provides queries, mutations, and subscriptions.

### Queries

**flows** -- List flows with filtering, sorting, and pagination.

```graphql
query {
  flows(filter: {
    host: "example.com"
    method: "POST"
    statusCode: 200
    flagged: true
    limit: 50
    offset: 0
    sortBy: "created_at"
    sortOrder: "desc"
    createdAfter: "2026-03-01T00:00:00Z"
    minStatus: 200
    maxStatus: 299
    hasParams: true
    contentType: "application/json"
    tag: "auth"
  }) {
    flows {
      id
      method
      url
      host
      status
      contentType
      size
      latency
      flagged
      createdAt
    }
    total
    limit
    offset
    hasMore
  }
}
```

Filter fields:

- `method` -- Filter by HTTP method
- `host` -- Filter by hostname
- `urlContains` -- Substring match on URL
- `statusCode` -- Exact status code match
- `flagged` -- Boolean filter for flagged flows
- `projectId` -- Filter by project UUID
- `limit` -- Results per page (default: 100)
- `offset` -- Pagination offset (default: 0)
- `sortBy` -- Sort column: `created_at`, `method`, `host`, `status`, `latency`, `size`
- `sortOrder` -- `asc` or `desc` (default: `desc`)
- `createdAfter` -- ISO 8601 timestamp lower bound
- `createdBefore` -- ISO 8601 timestamp upper bound
- `minStatus` / `maxStatus` -- Status code range
- `minLatencyMs` -- Minimum response latency in milliseconds
- `hasParams` -- Only flows whose URL contains `?`
- `contentType` -- Content-Type LIKE match
- `tag` -- Flow must have this specific tag

**flow** -- Get a single flow by ID.

```graphql
query {
  flow(id: "a3f2e1d0-...") {
    id
    method
    url
    host
    status
    requestHeaders
    responseHeaders
    requestBody
    responseBody
    latency
    flagged
    annotations
    createdAt
  }
}
```

**stats** -- Proxy statistics.

```graphql
query {
  stats {
    totalFlows
    uniqueHosts
    totalSize
  }
}
```

### Mutations

**annotateFlow** -- Flag, unflag, comment, or delete a flow.

```graphql
mutation {
  annotateFlow(id: "a3f2...", action: FLAG, comment: "Interesting endpoint") {
    id
    flagged
  }
}
```

**deleteFlow** -- Delete a flow by ID.

```graphql
mutation {
  deleteFlow(id: "a3f2...") {
    success
  }
}
```

### Subscriptions

**flowUpdates** -- Real-time stream of new flows as they are captured by the proxy.

```graphql
subscription {
  flowUpdates {
    id
    method
    url
    host
    status
    latency
    createdAt
  }
}
```

This subscription uses WebSocket transport (graphql-ws protocol). Connect to `ws://127.0.0.1:8081/graphql` with the `graphql-transport-ws` subprotocol.

## Usage with curl

```bash
# Query flows
curl -X POST http://127.0.0.1:8081/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ flows(filter: { host: \"example.com\", limit: 10 }) { flows { id method url status } total } }"}'

# Get stats
curl -X POST http://127.0.0.1:8081/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ stats { totalFlows uniqueHosts } }"}'
```

## Usage from JavaScript

```javascript
const response = await fetch('http://127.0.0.1:8081/graphql', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    query: `
      query($host: String) {
        flows(filter: { host: $host, limit: 20 }) {
          flows { id method url status latency }
          total hasMore
        }
      }
    `,
    variables: { host: 'api.example.com' }
  })
});
const { data } = await response.json();
```

## Authentication

When API authentication is enabled, include credentials in the request:

```bash
# Bearer token
curl -X POST http://host:8081/graphql \
  -H "Authorization: Bearer your-token" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ stats { totalFlows } }"}'

# Basic Auth
curl -X POST http://host:8081/graphql \
  -u "admin:password" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ stats { totalFlows } }"}'
```

## REST vs GraphQL

The REST API and GraphQL API share the same underlying data store and service layer. Choose based on your use case:

- **REST:** Simple CRUD, automation scripts, CI/CD integration, shell scripting
- **GraphQL:** Complex queries with multiple filters, real-time subscriptions, UI integration, when you need precise control over returned fields
