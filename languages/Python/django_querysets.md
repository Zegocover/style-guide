### Use custom QuerySet methods outside the model
Prefer not to use the raw manager methods like `filter` or `order_by` outside the model itself, but instead create and use custom `QuerySet` methods that abstract over the details.

#### Why?
Raw ORM methods like `filter` and `order_by` expose implementation details of a model's storage and logic, which makes it more likely that they will be incorrect due to missing important details, and makes models much harder to refactor and reason about.
When writing custom ORM methods, prefer putting them on a customer `Queryset` so that they are chainable allowing more complex queries to be built up from reusable primitives, without exposing the implementation details of the model.
When the custom ORM method should return a single value, an aggregate, or similar prefer the `Manager` and explicitly annotate the return type


Prefer:
```python
# ------ models.py ------
class Policy(models.Model):
    objects = PolicyManager.from_queryset(PolicyQuerySet)()
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    class_of_use = models.CharField(max_length=20)
    start_date = models.DateTimeField()
    end_date = models.DateTimeField(null=True)

# ------ managers.py ------
class PolicyQuerySet(models.QuerySet):
    # these methods are available on both queryset and manager
    def sdp(self):
        return self.filter(class_of_use="SDP")
    def current(self):
        now = datetime.now(tz=timezone.utc)
        return self.filter(start_date__gte=now, end_date__lt=now)


class PolicyManager(models.Manager):
    # any manager-only methods would go here
    pass


class CreditChargeQuerySet(models.QuerySet):
    def balance(self, at=None, *, account=None) -> models.QuerySet:
        at = at or tz_now()
        qs = self.filter(date_created__lte=at)

        if account:
            qs = qs.filter(account=account)

        return qs


class CreditChargeManager(models.Manager):
    def aggregate_balance(self, at=None, account=None) -> Decimal:
        return (
            self.get_queryset().balance(
                at, account=account
            ).aggregate(models.Sum('amount'))['amount__sum']
            or Decimal('0')
        )


# ------ any other files ------
total_credits = account.credit_set.aggregate_balance()
current_sdp_policies = Policy.objects.current().sdp()
```

Avoid:
```python
now = datetime.now(tz=timezone.utc)

total_credits = account.credit_set.filter(
    date_created__lte=now,
    account=account
).aggregate(Sum('amount'))['amount__sum']


current_sdp_policies = Policy.objects.filter(
    start_date__gte=now, 
    end_date__lt=now,
    class_of_use="SDP"
)
```

