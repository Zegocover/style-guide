# Python architecture

- [Python architecture](#python-architecture)
  - [General](#general)
  - [Django](#django)
    - [Use UUID instead of sequential IDs](#use-uuid-instead-of-sequential-ids)
          - [Why:](#why)
    - [Queries inside managers](#queries-inside-managers)
          - [Why:](#why-1)


## General
[work in progress]

## Django
### Use UUID instead of sequential IDs
There are a fair number of articles that explain why sequential ids might be an issue but, to summarise:

###### Why:
- Avoids information disclosure
- Stops entity enumeration
- Less prone to mistakes while querying DB

Don't:
```python
class Entity(models.Model):
    id = models.AutoField(primary_key=True)
```

Do:
```python
class Entity(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
```

### Queries inside managers
Django concept of a `Manager` is:
> The interface through which database query operations are provided to Django models

Even though the term `Manager` is as broad as it can be, we should follow the Django concept and define all of our queries inside the Manager set for a particular Model.

###### Why:
- Avoids duplication of queries
- Easier to refactor code by not having to find all the dispersed queries if a field is changed or removed
- Encapsulates Model logic close to the model
- Easier to read

Don't:
```python
# don't write random queries in the code
Policy.objects.filter(
  end_date__gte=tz_now(tz), product__enabled=True)
```

Do:
```python
# define the manager
class PolicyManager(models.Manager):
  def get_active_or_upcoming():
    return self.get_queryset().filter(
      end_date__gte=tz_now(tz), product__enabled=True
    )

# set the manager in the model
class Policy(models.Model):
  objects = PolicyManager()

# and then use this wherever you want to
Policy.objects.get_active_or_upcoming() 
```

Name of a manager query method should be descriptive of what the query is actually achieving, not how it is achieving it. e.g:

Don't:
```python
def get_product_enabled_and_end_date_gte_today():
  return self.get_queryset().filter(
    end_date__gte=tz_now(tz), product__enabled=True
  )
```

Do:
```python
def get_active_or_upcoming():
  return self.get_queryset().filter(
    end_date__gte=tz_now(tz), product__enabled=True
  )
```

There is also no need to duplicate the name of the entity in the method name, so `get_active_or_upcoming_policy` can be just `get_active_or_upcoming`, remember that this should be called like so:

```python
Policy.objects.get_active_or_upcoming()
```
