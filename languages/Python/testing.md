# Testing

## Unit Testing

In most cases, if you are adding code, changing anything that's already there, or want to reason about how something is currently working, you want a unit test. As a fast-growing startup we still have quite a lot of legacy code that sits untested - if you're changing it it's generally good practice (and very welcome and encouraged) to add a unit test. 

Where possible, unit tests should:
- Follow the **arrange-act-assert** pattern
- Use fixtures where you might want reusability
- Use factories rather than creating instances of models using .create() or using the model's constructor

In general, aim for smaller, more concise unit tests over longer tests that might cover a larger stretch of the codebase. Keeping tests smaller and with fewer assertions will make them easier to reason about, follow and fix when there are issues found, or refactor when the code under test has changed.

### Arrange, Act, Assert

The arrange, act, assert pattern allows you to split your test into three parts:

a) arrange
- this is where you set the system state under test
- this generally involves generating a bunch of objects using factories, and setting them up
- this might be a valid state if you want to test happy-path behaviour, or an invalid state if you want to test error handling

b) act
- this is where you carry out the logic under test
- this is often a function call

c) assert
- this is where you assert the behaviour you expected has been carried out
- generally we use Python's native `assert` statement, but if you're testing error handling you might want django.test.TestCase's `self.assertRaises()` or pytest's `raises()`.

Some common cases:

```python
class TestCar:
    def test_car_is_a_vehicle(self):
        # arrange
        car = CarFactory()

        # act
        car_is_vehicle = check_if_vehicle(car)

        # assert
        assert car_is_vehicle == True
```

```python
class TestBoat(django.test.TestCase):
    @pytest.fixture(autouse=True)
    def boat(self):
        return BoatFactory(name='Boaty McBoatface')

    def test_boat_cannot_fly(self, boat):
        # in this test the 'arrange' step is effectively done by the fixture

        # here the act and assert steps are bunched together
        with self.assertRaises(CannotFlyError) as e:
            boat.fly()
            assert e.message == 'Boat Boaty MyBoatface is unable to fly'
```

Or, with pytest's `raises`:

```python
class TestBoat:
    def test_boat_cannot_fly(self):
        # arrange
        boat = BoatFactory(name='Boaty McBoatface')

        # here the act and assert steps are bunched together
        with pytest.raises(CannotFlyError) as e:
            boat.fly()
            assert 'Boat Boaty MyBoatface is unable to fly' in str(e.value)
```

```python
class TestHovercraft:
    def test_hovercraft_travel_sends_destination_coordinates_to_api(self, mock_api):
        # arrange
        hovercraft = HovercraftFactory()

        # act
        hovercraft.go_to('porto')

        # assert
        mock_api.assert_called_with(vehicle_id=hovercraft.external_id, coordinates=('41.1579N', '8.6291W'))
```

### Using Fixtures

#### Why?

pytest's fixtures allow:
* easy reuse of standard objects
    - if you're writing a series of tests for a user living in London, you can define them once and test using a fresh copy of that object several times. This is great as it lowers the chance you'll make a mistake during setup, and if you need to change that object for some reason, you can change it once without having to change it in each test
* easy reuse of patched functions
    - if you're writing a series of tests for an integration with an external partner and you want to patch out the integration with a standard response you only need to do it once
* if you're expecting to use the fixture a number of times in your test class (or module), you can use the autouse=True flag to automatically make that fixture available to all tests in that class/module.


#### How?

```python
@pytest.fixture(autouse=True)
def product(self):
    return ProductFactory(is_flexible=False)

@pytest.fixture(autouse=True)
def policy(self, product):
    return PolicyFactory(product=product)

def test_something_about_policies(self, policy):
    result = function_under_test(policy)
    assert result.good == 'yes'
```


### Using Factories

#### Why?

factory_boy's factories allow us to:
* generate valid objects with sensible defaults
* generate sensible foreign key fields easily using subfactories
* set up related objects and configuration using post_generation steps

#### How?
- We define our factories separately by module - eg. there is a policy/factories.py, which carries all policy-related factories.
- You can create a new factory for a Django model by inheriting from factory.django.DjangoModelFactory, setting the model and defaults as so:

```python
class CatFactory(factory.django.DjangoModelFactory): 
    def Meta:
        model = Cat

    name = 'Felix'
    colour = 'black and white'
    pattern = 'stripy'
    legs = 4
```

You can then use this factory in a test:
```python
def test_plain_cat_can_climb():
    cat = CatFactory(pattern='plain')

    can_climb = cat.climb()

    assert can_climb == True
```

Further details on how to use factories can be found in FactoryBoy's [documentation](https://factoryboy.readthedocs.io/en/latest/index.html).


### On IDs
* Hardcoding an object ID in a test is bad practice, and should be avoided.

#### Why?
* As most of our IDs are automatically generated, there is no guarantee that you will pick an ID that will never be generated by the test suite. As IDs are primary keys, your test will fail if you attempt to put an ID into the test database table that already exists.


### Dates

* Ideally, your code under test should take a datetime as a parameter, rather than relying on the current time.
* However, if either the code under test or your test itself does need to generate datetimes dynamically (by using `datetime.datetime.now()` or similar), or relies on the current date fitting a certain set of criteria (such as being in the year 2020), you should freeze your test in time using the `@freeze_time` decorator from the freezegun library.

#### Why?
* Dates and times are, ultimately, difficult. We often see unexpected test failures from things like:
    - assuming that we will never reach a particular date - for example, a test that assumes a `future_shift` will start on 10/10/2020.
    - the year has changed, and your test that assumed we would always be in the year 2020 has failed.
    - daylight savings time! Terrible.
    - leap years! Total nightmare.
* The outcome of all of these is usually an unexpectedly broken master build.


#### How?
```python
from freezegun import freeze_time

class SomeTest:
    @freeze_time('2019-09-04T17:44:00Z')
    def look_its_definitely_august_2019():
        now = datetime.now()
```

If you want to freeze a datetime and (for example, in a fixture), you can also use `freeze_time` as a context manager:

```python
with freeze_time('2019-09-04T17:44:00Z'):
    now = datetime.now()
```

* Remember to define timezones in your tests.

#### Why?
* All of our instances of `datetime` are timezone-aware. Using non-timezone-aware datetimes in your tests will both give you a nasty warning and may cause unexpected behaviour when the timezone changes (particularly if you've forgotten to freeze your datetimes).

#### How?
* If you're manually defining a `datetime`, use the `tzinfo` keyword argument. You can use the `pytz` library to populate this, or `datetime`'s `timezone`.

```python
import datetime
import pytz

some_date = datetime(2020, 1, 2, tzinfo=pytz.UTC)
```


### Using parameterized tests

* In some cases you may find yourself writing a set of tests that have very similar inputs, very similar outputs, and you just want to assert that for an input `x`, we see an output `y`. For this, we use pytest's `@pytest.mark.parametrize` decorator.
* Further details on using pytest's `parametrize` can be found in the [pytest documentation](https://docs.pytest.org/en/stable/parametrize.html).

#### How?
```python
class TestAnimal:
    @pytest.mark.parametrize("animal,expected", [("bird", True), ("horse", False), ("cat", False)])
    def test_can_fly(self, animal, expected):
        assert can_fly(animal) == expected
```

#### Why?
* `parametrize` just effectively reruns the test for each input in turn, so it is mostly an exercise in avoiding repetition. In this case, avoiding repetition is good as it means we're not accidentally running a different test setup from one case to another, and any changes to the test will be neccessarily be run for all test cases.


### A note on mocking (and @patching)

Given the structure of our existing codebase, we prefer to use fixtures and factories over Mocks. In some cases, we use Mocks for instances of classes defined in libraries such as the Stripe Python library. If you choose to use the Mock class, consider instantiating it with the `spec` keyword argument. This will protect you against attempting to access attributes that don't exist on the class.

```python
class Plant(models.Model):
    type = models.CharField(max_length=100)


class TestPlant:
    def test_this_passes_fine(self):
        mock_plant = Mock()
        mock_plant.number_of_leaves    # this is fine, even though there is no number_of_leaves on Plant
        assert True

    def test_this_raises_an_error(self):
        mock_plant = Mock(spec=Plant)
        mock_plant.number_of_leaves   # this will raise an AttributeError
        assert True
```

We generally use @patch to patch out calls to external APIs. If you find that you are having to patch out lots of calls to functions within our Django monolith, this is generally a sign that the code under test mixes multiple concerns, and may have become more brittle as a result. (It is also often a strong indicator that the the code is too complex to reason about easily.) In either case, it is worth considering how you could refactor the code under test.
