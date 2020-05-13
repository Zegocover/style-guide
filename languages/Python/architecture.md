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

To be able to talk about code organisation, it's worth defining some common terminology. These definitions are taken from domain-driven design which, although it isn't possible to fully apply it in Django, provides some valuable techniques which can be used.

#### Value Object

A value object is an object whose identity is defined solely by its data. A common example is a `string` as although there may be multiple instances of the string `"hello world"` in memory we consider them to be the same for all intents and purposes. Value objects are always immutable, as any change to the data is effectively a different value object.

The `User` entity shows a common pattern with value objects of storing their data inline in an entity. Here the user's address is stored as individual fields in the database record, which could be synthesised into an `Address` property.

#### Entity

An entity is an object whose identity is defined by an identifier, rather than by its data. This identifier may be the database primary key, or a surrogate identifier such as a policy number. Entities are often mutable, although this is not a requirement.

#### Aggregate

An aggregate is one or more entities that logically form a single unit. In any aggregate there is a 'top level' entity, known as the aggregate root, which is responsible for enforcing any invariants in the entire aggregate.

The `FinancialTransaction` aggregate is an interesting one to look at, because it has multiple `Entry` children which need to obey the invariant that their value always sums to zero. At first glance the `Entry` objects might look like entities, but it's only their data that matters so they are really value objects; their identifier exists only to allow hierarchical data to be stored within the constraints of a relational database.

### Layering

To make code manageable and maintainable, most large codebases are separated into different 'layers' which each have their own responsibility. Within a Django application, the layers we would typically have are, from bottom to top:

- Model & Persistence
- Domain Services
- Application Services
- Presentation

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

If this had been encapsulated in a domain service, then there would be a single source of truth for which product should be renewed to, and only one place would need to be changed:

```python
class PolicyRenewals:
    def __init__(policy: Policy) -> None
        self.policy = policy

    def renewal_product(self) -> Optional[Product]:
        product = self.policy.product.renewals_switch_to or self.policy.product
        return product if product.allow_new_users else None
```

Note that a domain service is used here partly because it is dealing with two different aggregates, `Policy` and `Product`, and partly because putting all the domain logic related to renewals together helps to improve cohesion and make it easier to reason about. In other cases it may be more appropriate to put the logic in a method on the aggregate root itself.





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



