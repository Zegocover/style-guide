# Python architecture

- [General](#general)
- [Code Organisation](#code-organisation)
  - [Terminology](#terminology)
  - [Layering](#layering)
  - [Guidelines](#guidelines)
- [Django](#django)
  - [Use UUID instead of sequential IDs](#use-uuid-instead-of-sequential-ids)

## General

(TODO)


## Code Organisation

### Terminology

To be able to talk about code organisation, it's worth defining some common terminology. These definitions are taken from domain-driven design which, although it isn't possible to fully apply in Django, provides some valuable techniques we can use.

The reason DDD can't be fully applied the key pattern in DDD of separating your model from persistence so when the model changes your database doesn't necessarily have to, and vice versa. You'll often see this called the repository pattern. This has quite a number of benefits such as exposing behaviour rather than data, allowing you to optimise storage independently of your model, or even change where data is stored (perhaps into a separate service).

However, Django idiomatically merges the model and persistence layers with its ORM. It would be possible to treat the classes derived from `django.models.Model` as being repositories and only ever using them to save or retrieve data, then expose plain Python classes internally. However, that would negate many of the reasons to use Django such as the auto-generated admin forms and suchlike.

This has implications around architecture because it means you're essentially restricted to a layered architecture rather than the more modern (and arguably superior) 'onion' architecture (aka ports and adapters, or hexagonal architecture).

#### Entity

An entity is an object whose identity is defined by an identifier, rather than by its data. This identifier may be the database primary key, or a surrogate identifier such as a policy number. Entities are often mutable, although this is not a requirement.

Many of the most common entities we deal with such as `Policy`, `Product` and `User` are entities because their identifier is what defines them; a `User` does not become a different user if they change their email address, for example.

#### Value Object

A value object is an object whose identity is defined solely by its data. A common example is a `string` as although there may be multiple instances of the string `"hello world"` in memory we consider them to be the same for all intents and purposes. Value objects are always immutable, as any change to the data is effectively a different value object.

Unfortunately, efficient use of relational databases requires identifiers, so there are a couple of patterns that have evolved for storing value object.

The `User` entity shows a common approach when there is a 1:1 relationship between an entity and a value object it contains: The user's address is stored inline in the user table as individual fields, so an `Address` property can be synthesised, for example:

```python
@dataclass
class Address:
    line1: str
    # ...
    postcode: str

class User(Model):
    @cached_property
    def address(self) -> Address:
        return Address(self.address_line1, ..., self.address_postcode)
```

The `FinancialTransaction` entity shows the pattern to use where there is a 1:many relationship between the entity and its `Entry` value objects: The value objects are stored in a separate table and then reference their parent object. At first glance the `Entry` objects might look like entities because they have an identifier, but it's only their data that matters so they are really value objects; their identifier exists only to help with database structure.

#### Aggregate

An aggregate is one or more entities that logically form a single unit. In any aggregate there is a 'top level' entity, known as the aggregate root, which is responsible for enforcing any invariants in the entire aggregate.

Most entities will also be aggregate roots; larger aggregates composed of multiple entities are comparatively rare.

### Layering

To make code manageable and maintainable, most large codebases are separated into different 'layers' which each have their own responsibility. Within a Django application, the layers we would typically have are, from bottom to top:

- Infrastructure
- Model & Persistence
- Domain Services
- Application Services
- Presentation

#### Infrastructure

Infrastructure, in the application sense, is the set of basic facilities that other code can build on such as signals, queues, logging, metrics, and so on. This is arguably less of a layer than supporting code that sits alongside all other layers.

#### Model & Persistence

This layer has the responsibility of ensuring that all aggregates are valid by enforcing their invariants, and saving/retrieving data from the database. Any domain logic that is related to a _single_ aggregate should live in this layer.

#### Domain Services

Domain services are used as a container for domain logic that relates to _multiple_ aggregates, because the logic does not belong in any particular aggregate. Domain services, along with the model and persistence, form the domain model.

#### Application Services

Application services are not part of the domain, but are used to orchestrate the domain into higher level processes. An example of this might be a renewals service which would orchestrate actions such as taking payment, renewing the policy, and sending notifications.

#### Presentation

The presentation layer includes anything that exposes the application to the outside world, such as a user interface or API. The presentation layer should typically only use application services, although in some cases may use the domain model directly.

### Guidelines

#### Encapsulate domain logic in methods

Even domain logic that appears trivial is worth encapsulating in a method.

This is best illustrated with an example. The `Policy` model has a `renewals_switch_to` property which is an optional `Product` that should be used on renewals. To determine the product to use for renewals seemed as easy as the following, which was used in a number of places:

```python
renewal_product = policy.renewals_switch_to or policy.product
```

However, the logic changed and this was subsequently updated in a number of places to also check things like whether new users were permitted, or whether the new product allowed renewals. Unfortunately it wasn't changed consistently everywhere, which meant that some places had different behaviour.

If this had been encapsulated in a method, then there would be a single source of truth for which product should be renewed to, and only one place would need to be changed:

```python
class Policy(Model):
    def renewal_product(self) -> Optional[Product]:
        product = self.product.renewals_switch_to or self.product
        return product if product.allow_new_users else None
```

You'll notice that this method uses two different aggregates, `Policy` and `Product`, so the method should arguably be in a domain service because aggregates should only have methods related to themselves. An alternative implementation might look something like this:

```python
class PolicyRenewals:
    def __init__(policy: Policy) -> None
        self.policy = policy

    def renewal_product(self) -> Optional[Product]:
        product = self.policy.product.renewals_switch_to or self.policy.product
        return product if product.allow_new_users else None
```

In this case either approach seems reasonable, because no properties of the `Product` aggregate are used, and too many tiny domain services can hurt discoverability. However, if the method started getting more complex and using details of the product, or there were a set of related renewals methods, then it would make sense to extract them into a cohesive domain service.

## Django

### Use UUID instead of sequential IDs
There are a fair number of articles that explain why sequential ids might be an issue but, to summarise:

###### Why:
- Avoids information disclosure
- Stops entity enumeration -> [IDOR](https://medium.com/@woj_ciech/explaining-idor-in-almost-real-life-scenario-in-bug-bounty-program-c214008f8378)
- Less prone to mistakes while querying DB

Don't:
```python
class Entity(models.Model):
    # Django adds a sequential ID by default
    ...
```

```python
class Entity(models.Model):
    id = models.AutoField(primary_key=True)
```

Do:
```python
class Entity(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
```

### Use of signals should be considered an anti-pattern

The use of signals can make it very difficult to reason about the code especially when they produce side-effects. Signals should only be used in very limited cases and most times shouldn't be used at all. Signals don't offer any advantages since they're executed synchronously. 

Keep the business logic out of the models and instead use service classes with well defined API's which encapsulate functionality. [Refer to Encapsulate domain logic in methods](#encapsulate-domain-logic-in-methods)

#### Don't
Use signals decouple code or have code run as a side-effects. They can fail and you may never know. They're also hard to test for and have no consistent order.

#### Do
Use they for very limited purposes. Better to not use them at all.

### Large migrations
At times we'll need to do updates to large tables and if we try to them all in one go we can bring down the whole site. In order to do this safely the migration should be broken down into multiple steps.
* Add new field
* Set default to value through code

