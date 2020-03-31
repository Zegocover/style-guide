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
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
```
