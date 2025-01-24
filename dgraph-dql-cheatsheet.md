# Dgraph DQL Cheat Sheet

## Table of Contents

| | |
|---|---|
| [1. Schema Basics](#schema-basics) | [2. Query Structure](#query-structure) |
| [3. Filters & Functions](#filters--functions) | [4. Aggregations](#aggregations) |
| [5. Math on Value Variables](#math-on-value-variables) | [6. Mutations](#mutations) |
| [7. Upserts](#upserts) | [8. Deleting Data](#deleting-data) |

## Schema Basics

### Predicate Types

| Type | Go Type | Description | Example |
|------|---------|-------------|---------|
| `default` | string | Default type if none specified | `"text"` |
| `int` | int64 | Integer values | `42` |
| `float` | float | Decimal values | `3.14` |
| `string` | string | Text values | `"John Doe"` |
| `bool` | bool | Boolean values | `true` |
| `dateTime` | time.Time | RFC3339 format | `"2024-01-23T15:04:05.999Z"` |
| `geo` | go-geom | Geolocation data | `"{'type':'Point','coordinates':[1.0,2.0]}"` |
| `password` | string | Encrypted strings | `"hash3dP@ssw0rd"` |
| `float32vector` | []float32 | Vector of float32 numbers | Used for embeddings |
| `uid` | uint64 | Node reference | Used for relationships |

### Predicate Definitions
```graphql
name: string @index(term) .
age: int @index(int) .
friend: [uid] @reverse .
description_vector: float32vector @index(hnsw(metric:"cosine")) .
```

### Special Directives

| Directive | Purpose | Example |
|-----------|---------|---------|
| `@index` | Enables searching/filtering | `name: string @index(exact)` |
| `@reverse` | Creates implicit reverse edges | `friend: [uid] @reverse` |
| `@unique` | Ensures unique values | `email: string @unique @index(exact)` |
| `@upsert` | Enables upsert operations | `email: string @index(exact) @upsert` |
| `@noconflict` | Disables conflict detection | `count: int @noconflict` |
| `@lang` | Enables language-specific text | `title: string @lang` |

### Node Types
```graphql
# Predicate declarations
name: string @index(term) .
release_date: datetime @index(year) .
revenue: float .
starring: [uid] .
director: [uid] .

# Type definitions
type Film {
    name
    release_date
    revenue
    starring
    director
}

type Person {
    name
    ~starring    # Reverse edge reference
    ~director
}
```

### Internationalization Support
```graphql
# International predicate names
<职业>: string @index(exact) .
<年龄>: int @index(int) .

# Language-specific values
title: string @index(fulltext) @lang .

# Usage in mutations
_:movie <title> "The Matrix"@en .
_:movie <title> "黑客帝国"@zh .
```

### Password Type Usage
```graphql
# Schema
pass: password .

# Mutation
{
  set {
    uid(user) <pass> "SecurePass123" .
  }
}

# Query (checking password)
{
  check(func: uid(user)) {
    checkpwd(pass, "SecurePass123")
  }
}
```

### Reverse Edges

The `@reverse` directive creates an implicit reverse edge in the graph, allowing you to query relationships in both directions without explicitly storing both edges. 

Here's a simple explanation:

If you have:
```graphql
<friend>: [uid] @reverse .
```

When you create an edge:
```graphql
A --friend--> B
```

Dgraph automatically maintains a *virtual* reverse edge:
```graphql
B <--~friend-- A
```

This means you can query either:
- From A to find their friends
- From B to find who has them as a friend (using `~friend`)

Without `@reverse`, you'd need to manually create and maintain both edges.


### Dgraph Type System

Dgraph also supports an optional type system that lets you define named **Types** and specify which predicates belong to each. Types help organize data more effectively, ensure certain predicates exist for nodes of that type, and allow you to write type-based queries like `type(User)`.

#### Defining a Type

```graphql
type User {
  name
  age
  friend
}
```

This block means:
- There's a **Type** named `User`.
- Nodes labeled as `User` will typically have `name`, `age`, and `friend` predicates.

#### Assigning a Node a Type

If you insert data for a `User`, you can specify the type in your mutation:

**RDF Example**:
```graphql
mutation {
  set {
    _:alice <dgraph.type> "User" .
    _:alice <name> "Alice" .
    _:alice <age> "28" .
  }
}
```

**JSON Example**:
```graphql
mutation {
  set {
    "uid": "_:alice",
    "dgraph.type": "User",
    "name": "Alice",
    "age": 28
  }
}
```

- The `<dgraph.type>` predicate is a special internal predicate used to assign a node to one or more types.

## Query Structure

### Basic Query
```graphql
{
  users(func: type(Person)) {
    name
    age
    friend {
      name
    }
  }
}
```

### Query with Variables
```graphql
query withVar($name: string) {
  users(func: eq(name, $name)) {
    uid
    name
  }
}
```

### Nested Queries
```graphql
{
  users(func: type(Person)) {
    name
    friend @filter(has(age)) {
      name
      age
    }
  }
}
```

### Variable Blocks
```graphql
{
  # Store UIDs in variables
  var(func: type(Person)) {
    name
    friends as friend {
      age as age  # Store scalar value
    }
    average as avg(val(age))  # Compute aggregate
    sum_friends as count(friend)  # Count edges
  }

  # Use variables in query
  result(func: uid(friends)) @filter(gt(val(age), 25)) {
    name
    age
    friend_count: val(sum_friends)
    avg_friend_age: val(average)
  }
}
```

Examples of Variable Usage:
```graphql
# Storing and using UIDs
var(func: type(Person)) @filter(has(name)) {
  a as uid  # Store current node UIDs
}
query(func: uid(a)) { name }

# Storing and using scalar values
var(func: type(Person)) {
  person_age as age
  person_name as name
}
query(func: uid(person_age)) {
  name
  age: val(person_age)
  username: val(person_name)
}

# Computing with variables
var(func: type(Person)) {
  age as age
  bonus as math(age * 100)  # Mathematical operations
}
query(func: uid(age)) {
  name
  age
  calculated_bonus: val(bonus)
}
```

## Filters & Functions

### Common Filter Functions
```graphql
eq(predicate, value)    # Equal to
gt(predicate, value)    # Greater than
lt(predicate, value)    # Less than
le(predicate, value)    # Less than or equal
ge(predicate, value)    # Greater than or equal
between(predicate, val1, val2)  # Value in range
```

### String Functions
```graphql
allofterms(predicate, "terms")  # All terms present
anyofterms(predicate, "terms")  # Any term present
regexp(predicate, /pattern/)    # Regular expression match
```

### Logical Operators
```graphql
@filter(AND(has(name), gt(age, 18)))
@filter(OR(eq(name, "John"), eq(name, "Jane")))
@filter(NOT(has(friend)))
```

## Aggregations

### Basic Aggregations
```graphql
{
  stats(func: type(Person)) {
    count(uid)        # Count nodes
    avg(age)          # Average of age
    sum(salary)       # Sum of salary
    min(age)          # Minimum age
    max(age)          # Maximum age
  }
}
```

### Grouping
```graphql
{
  groups(func: type(Person)) @groupby(age) {
    count(uid)
  }
}
```

## Math on Value Variables

### Basic Math Functions

| Operation | Syntax | Description | Example |
|-----------|--------|-------------|---------|
| `math()` | `math(val1 + val2)` | Basic arithmetic | `bonus as math(salary * 1.1)` |
| `floor()` | `floor(val(x))` | Round down to nearest integer | `floor_price as floor(val(price))` |
| `ceil()` | `ceil(val(x))` | Round up to nearest integer | `ceil_amount as ceil(val(amount))` |
| `ln()` | `ln(val(x))` | Natural logarithm | `log_value as ln(val(growth))` |
| `exp()` | `exp(val(x))` | Exponential function | `exp_growth as exp(val(rate))` |
| `since()` | `since(val(x))` | Time since datetime | `days_active as since(val(join_date))` |

### Advanced Math Operations

| Operation | Syntax | Description | Example |
|-----------|--------|-------------|---------|
| `pow()` | `pow(val(x), y)` | Power function | `squared as pow(val(base), 2)` |
| `sqrt()` | `sqrt(val(x))` | Square root | `root_value as sqrt(val(area))` |
| `abs()` | `abs(val(x))` | Absolute value | `magnitude as abs(val(difference))` |
| `min()`/`max()` | `min(val(x), val(y))` | Minimum/Maximum | `lower_bound as min(val(price1), val(price2))` |
| `logbase()` | `logbase(val(x), base)` | Logarithm with custom base | `log10 as logbase(val(num), 10)` |
| `cond()` | `cond(val(x) > 0, val(y), val(z))` | Conditional value | `result as cond(val(score) > 90, "A", "B")` |

### Practical Examples

```graphql
# Calculate compound interest
{
  var(func: type(Investment)) {
    principal as initial_amount
    rate as interest_rate
    years as duration
    
    # Compound Interest Formula: P(1 + r)^t
    interest_multiplier as math(pow(1 + val(rate), val(years)))
    final_amount as math(val(principal) * val(interest_multiplier))
  }
  
  investments(func: uid(principal)) {
    investment_id
    initial_amount
    calculated_return: val(final_amount)
    profit: math(val(final_amount) - val(principal))
  }
}

# Calculate age and categorize users
{
  var(func: type(Person)) {
    birth_date as dob
    current_age as since(val(birth_date))/60/60/24/365
    
    age_category as cond(
      val(current_age) < 18, "minor",
      cond(val(current_age) < 65, "adult", "senior")
    )
  }
  
  people(func: uid(birth_date)) {
    name
    dob
    age: val(current_age)
    category: val(age_category)
  }
}

# Statistical calculations
{
  var(func: type(Product)) {
    price as price
    
    # Calculate various statistical measures
    log_price as ln(val(price))
    price_squared as pow(val(price), 2)
    price_root as sqrt(val(price))
    
    # Price adjustments
    discounted_price as math(val(price) * 0.9)
    ceiling_price as ceil(val(discounted_price))
    floor_price as floor(val(discounted_price))
  }
  
  statistics(func: uid(price)) {
    product_name
    original_price: val(price)
    price_metrics {
      log: val(log_price)
      squared: val(price_squared)
      square_root: val(price_root)
      discounted {
        exact: val(discounted_price)
        ceiling: val(ceiling_price)
        floor: val(floor_price)
      }
    }
  }
}
```

## Mutations

### Add Data
```graphql
mutation {
  set {
    _:person <name> "John Doe" .
    _:person <age> "30" .
    _:person <friend> uid(0x123) .
  }
}
```

### Update Data
```graphql
mutation {
  set {
    uid(0x123) <age> "31" .
  }
}
```

## Upserts

### Basic Upsert
```graphql
mutation {
  query {
    user as var(func: eq(email, "john@example.com"))
  }
  
  condition: uid(user) {
    # Update if exists
    uid(user) <name> "John Doe" .
  }
  
  default: not(uid(user)) {
    # Create if not exists
    _:user <email> "john@example.com" .
    _:user <name> "John Doe" .
  }
}
```

### Conditional Upsert
```graphql
mutation {
  query {
    u as var(func: eq(email, "john@example.com"))
    v as var(func: eq(status, "ACTIVE"))
  }
  
  condition: uid(u) AND uid(v) {
    uid(u) <status> "INACTIVE" .
  }
}
```

## Deleting Data

### Delete Specific Predicates
```graphql
mutation {
  delete {
    uid(0x123) <name> * .
  }
}
```

### Delete Node and All Edges
```graphql
mutation {
  delete {
    uid(0x123) * * .
  }
}
```

### Delete with Conditions
```graphql
mutation {
  delete {
    uid(0x123) <friend> uid(0x456) .
  }
}
```

