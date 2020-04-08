# Python architecture

- [General](#general)
- [Django](#django)
  - [Use UUID instead of sequential IDs](#use-uuid-instead-of-sequential-ids)


## General
[work in progress]

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
