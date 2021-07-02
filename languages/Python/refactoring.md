# Refactoring

## Safer Refactoring

As with any form of software engineering, refactoring comes with its own set of risks. This section of our style guide focusses on strategies we use at Zego to reduce that risk.

### Experiments

When refactoring existing code with complex behaviour, consider using the [laboratory library](https://github.com/joealcorn/laboratory) to ensure that the output of your new code (the candidate function) matches the output of your old code (the control function).

Laboratory runs your old code and your new code sequentially, then compares the output. In either case, the output of the control function is returned as the result of the experiment. By defining your own `Experiment` subclass, you can [override `publish`](https://github.com/joealcorn/laboratory#publishing-results) to get more granular metrics. Within the monolith, you can subclass the `ZegoExperiment` class to automatically get some basic metrics (`match`es and `mismatch`es) published to Datadog. You can also [override the `compare` function](https://github.com/joealcorn/laboratory#controlling-comparison) if you expect your candidate function to return something slightly different from your control function.

As Laboratory runs both sets of code, it is important to take performance and idempotent behaviour into consideration. Laboratory does *not* ensure that the same side effects occur in both functions. The ideal candidate for an experiment is a pair of functions that perform well under standard conditions, do minimal reads from and no writes to databases, and do not call external services. Within our codebase these ideal candidates are often business logic within domain services. Laboratory should not be used for code that makes any updates to the database or calls third-party services.

An example experiment class:

```
class CakeExperiment(ZegoExperiment):
    def compare(self, control: Observation, candidate: Observation) -> bool:
        """
        Contrived cake-based experiment. We expect eggs, flour, sugar, and milk to be the same,
        but started_baking_at may be different.
        """
        for key in ["eggs", "flour", "milk", "sugar"]:
            if control.value[key] != candidate.value[key]:
                return self._handle_comparison_mismatch(control, candidate)
        return True

    def publish(self, result: Result) -> None:
        if result.match:
            statsd.increment("experiments.cake.success")
        else:
            statsd.increment("experiments.cake.failure")
```

An example execution:

```
class Bakery:
    def bake_cake(self, recipe: Recipe) -> Cake:
        experiment = CakeExperiment("Cake-based experiment")
        experiment.control(old_cupboard.get_ingredients(recipe))
        experiment.candidate(new_cupboard.get_ingredients(recipe))
        ingredients = experiment.conduct()

        return bake(ingredients)
```


### Feature Flags - not just for features

Feature flags allow partial rollout of all kinds of code, not just new features. Refactored code is particularly well-placed for partial rollout as there is minimal user impact expected (so switching a flag on or off for a particular cohort should result in no surprises for the user) and in most cases side effects such as writes to the database remain the same. We use [LaunchDarkly](https://launchdarkly.com) to manage flags, which ensures that a client will not switch cohorts unexpectedly.

Example usage:
```
if launchdarkly.variation("some-flag", {"key": "global"}, default=False):
    do_old_thing()
else:
    do_new_thing()
```

**Best Practices**

*Keys*

The first argument for `.variation()` is the key, or the feature flag's name. For each feature flag we can define individual cohorts through LaunchDarkly's UI. This means that we should never have a 1:1 mapping between keys and clients. "turn-on-new-feature" is good, "turn-on-new-feature-for-user-x" is bad.


*Monitoring*

Feature flags are particularly powerful when appropriate monitoring is added. When you add monitoring, consider:
- How can I identify which cohort this user is from (ie. whether they are interacting with the new code or the old code)?
- How can I identify if there has been a performance impact?
- How can I identify the impact of my refactor on individual business metrics?

*Set default using a keyword argument*

The LaunchDarkly SDK allows you to provide a default return value, [which may be anything you would like](https://github.com/launchdarkly/python-server-sdk/blob/d3a827774b0b1695cfb8ba991538a6faa2dd98da/ldclient/client.py#L250). While the meaning of the first to arguments can be easily inferred from context, the third one (default) is fairly ambiguous and should be called out using the kwarg syntax.
