Beyond accessing model attributes directly via their field names (e.g. `model.foobar`), models can be converted, dumped,
serialized, and exported in a number of ways:

## `model.model_dump(...)`

This is the primary way of converting a model to a dictionary. Sub-models will be recursively converted to dictionaries.

!!! note
    The one exception to sub-models being converted to dictionaries is that [`RootModel`](models.md#rootmodel-and-custom-root-types)
    and its subclasses will have the `root` field value dumped directly, without a wrapping dictionary. This is also
    done recursively.

See the [API docs for `model_dump`](../api/main.md#pydantic.main.BaseModel.model_dump) for more information.

Example:

```py
from typing import Any, List, Optional

from pydantic import BaseModel, Field, Json


class BarModel(BaseModel):
    whatever: int


class FooBarModel(BaseModel):
    banana: Optional[float] = 1.1
    foo: str = Field(serialization_alias='foo_alias')
    bar: BarModel


m = FooBarModel(banana=3.14, foo='hello', bar={'whatever': 123})

# returns a dictionary:
print(m.model_dump())
#> {'banana': 3.14, 'foo': 'hello', 'bar': {'whatever': 123}}
print(m.model_dump(include={'foo', 'bar'}))
#> {'foo': 'hello', 'bar': {'whatever': 123}}
print(m.model_dump(exclude={'foo', 'bar'}))
#> {'banana': 3.14}
print(m.model_dump(by_alias=True))
#> {'banana': 3.14, 'foo_alias': 'hello', 'bar': {'whatever': 123}}
print(FooBarModel(foo='hello', bar={'whatever': 123}).model_dump(exclude_unset=True))
#> {'foo': 'hello', 'bar': {'whatever': 123}}
print(
    FooBarModel(banana=1.1, foo='hello', bar={'whatever': 123}).model_dump(
        exclude_defaults=True
    )
)
#> {'foo': 'hello', 'bar': {'whatever': 123}}
print(FooBarModel(foo='hello', bar={'whatever': 123}).model_dump(exclude_defaults=True))
#> {'foo': 'hello', 'bar': {'whatever': 123}}
print(
    FooBarModel(banana=None, foo='hello', bar={'whatever': 123}).model_dump(
        exclude_none=True
    )
)
#> {'foo': 'hello', 'bar': {'whatever': 123}}


class Model(BaseModel):
    x: List[Json[Any]]


print(Model(x=['{"a": 1}', '[1, 2]']).model_dump())
#> {'x': [{'a': 1}, [1, 2]]}
print(Model(x=['{"a": 1}', '[1, 2]']).model_dump(round_trip=True))
#> {'x': ['{"a":1}', '[1,2]']}
```

## `model.model_dump_json(...)`

The `.model_dump_json()` method serializes a model directly to a JSON-encoded string
that is equivalent to the result produced by [`.model_dump()`](#modelmodeldump).

See [arguments](../api/main.md#pydantic.main.BaseModel.model_dump_json) for more information.

!!! note
    Pydantic can serialize many commonly used types to JSON that would otherwise be incompatible with a simple
    `json.dumps(foobar)` (e.g. `datetime`, `date` or `UUID`) .

```py
from datetime import datetime

from pydantic import BaseModel


class BarModel(BaseModel):
    whatever: int


class FooBarModel(BaseModel):
    foo: datetime
    bar: BarModel


m = FooBarModel(foo=datetime(2032, 6, 1, 12, 13, 14), bar={'whatever': 123})
print(m.model_dump_json())
#> {"foo":"2032-06-01T12:13:14","bar":{"whatever":123}}
print(m.model_dump_json(indent=2))
"""
{
  "foo": "2032-06-01T12:13:14",
  "bar": {
    "whatever": 123
  }
}
"""
```

## `dict(model)` and iteration

Pydantic models can also be converted to dictionaries using `dict(model)`, and you can also iterate over a model's
fields using `for field_name, field_value in model:`. With this approach the raw field values are returned, so
sub-models will not be converted to dictionaries.

Example:

```py
from pydantic import BaseModel


class BarModel(BaseModel):
    whatever: int


class FooBarModel(BaseModel):
    banana: float
    foo: str
    bar: BarModel


m = FooBarModel(banana=3.14, foo='hello', bar={'whatever': 123})

print(dict(m))
#> {'banana': 3.14, 'foo': 'hello', 'bar': BarModel(whatever=123)}
for name, value in m:
    print(f'{name}: {value}')
    #> banana: 3.14
    #> foo: hello
    #> bar: whatever=123
```

Note also that [`RootModel`](models.md#rootmodel-and-custom-root-types) _does_ get converted to a dictionary with the key `'root'`.

## Custom serializers

Serialization can be customised on a field using the `@field_serializer` decorator, and on a model using the
`@model_serializer` decorator.

Here is an example:

```py
from datetime import datetime, timedelta, timezone
from typing import Any, Dict

from pydantic import BaseModel, ConfigDict, field_serializer, model_serializer


class WithCustomEncoders(BaseModel):
    model_config = ConfigDict(ser_json_timedelta='iso8601')

    dt: datetime
    diff: timedelta

    @field_serializer('dt')
    def serialize_dt(self, dt: datetime, _info):
        return dt.timestamp()


m = WithCustomEncoders(
    dt=datetime(2032, 6, 1, tzinfo=timezone.utc), diff=timedelta(hours=100)
)
print(m.model_dump_json())
#> {"dt":1969660800.0,"diff":"P4DT14400S"}


class Model(BaseModel):
    x: str

    @model_serializer
    def ser_model(self) -> Dict[str, Any]:
        return {'x': f'serialized {self.x}'}


print(Model(x='test value').model_dump_json())
#> {"x":"serialized test value"}
```

### Serializing subclasses

### Subclasses of standard types

Subclasses of standard types are automatically dumped like their super-classes:

```py
from datetime import date, timedelta
from typing import Any, Type

from pydantic_core import core_schema

from pydantic import BaseModel, GetCoreSchemaHandler


class DayThisYear(date):
    """
    Contrived example of a special type of date that
    takes an int and interprets it as a day in the current year
    """

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source: Type[Any], handler: GetCoreSchemaHandler
    ) -> core_schema.CoreSchema:
        return core_schema.general_after_validator_function(
            cls.validate,
            core_schema.int_schema(),
            serialization=core_schema.format_ser_schema('%Y-%m-%d'),
        )

    @classmethod
    def validate(cls, v: int, _info):
        return date.today().replace(month=1, day=1) + timedelta(days=v)


class FooModel(BaseModel):
    date: DayThisYear


m = FooModel(date=300)
print(m.model_dump_json())
#> {"date":"2023-10-28"}
```

## Subclass instances for fields of BaseModel, dataclasses, TypedDict

When using fields whose annotations are themselves struct-like types (e.g., `BaseModel` subclasses, dataclasses, etc.),
the default behavior is to serialize the attribute value as though it was an instance of the annotated type,
even if it is a subclass. More specifically, only the fields from the _annotated_ type will be included in the
dumped object:

```py
from pydantic import BaseModel


class InnerModel(BaseModel):
    x: int


class SubInnerModel(InnerModel):
    y: int


class OuterModel(BaseModel):
    inner: InnerModel


m = OuterModel(inner=SubInnerModel(x=1, y=2))
print(m)
#> inner=SubInnerModel(x=1, y=2)
print(m.model_dump())
#> {'inner': {'x': 1}}
```
!!! warning "Migration Warning"
    This behavior is different from how things worked in Pydantic V1, where we would always include
    all (subclass) fields when recursively dumping models to dicts. The motivation behind this change in
    behavior is that it helps ensure that you know precisely which fields could be included when serializing,
    even if subclasses get passed when instantiating the object. In particular, this can help prevent surprises
    when adding sensitive information like secrets as fields of subclasses.

TODO: Add and document `SerializeAsAny` type.

## `pickle.dumps(model)`

Pydantic models support efficient pickling and unpickling.

```py test="skip"
# TODO need to get pickling to work
import pickle

from pydantic import BaseModel


class FooBarModel(BaseModel):
    a: str
    b: int


m = FooBarModel(a='hello', b=123)
print(m)
#> a='hello' b=123
data = pickle.dumps(m)
print(data[:20])
#> b'\x80\x04\x95\x95\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main_'
m2 = pickle.loads(data)
print(m2)
#> a='hello' b=123
```

## Advanced include and exclude

The `model_dump` and `model_dump_json` methods support `include` and `exclude` arguments which can either be
sets or dictionaries. This allows nested selection of which fields to export:

```py
from pydantic import BaseModel, SecretStr


class User(BaseModel):
    id: int
    username: str
    password: SecretStr


class Transaction(BaseModel):
    id: str
    user: User
    value: int


t = Transaction(
    id='1234567890',
    user=User(id=42, username='JohnDoe', password='hashedpassword'),
    value=9876543210,
)

# using a set:
print(t.model_dump(exclude={'user', 'value'}))
#> {'id': '1234567890'}

# using a dict:
print(t.model_dump(exclude={'user': {'username', 'password'}, 'value': True}))
#> {'id': '1234567890', 'user': {'id': 42}}

print(t.model_dump(include={'id': True, 'user': {'id'}}))
#> {'id': '1234567890', 'user': {'id': 42}}
```

The `True` indicates that we want to exclude or include an entire key, just as if we included it in a set.
This can be done at any depth level.

Special care must be taken when including or excluding fields from a list or tuple of submodels or dictionaries.
In this scenario, `model_dump` and related methods expect integer keys for element-wise inclusion or exclusion.
To exclude a field from **every** member of a list or tuple, the dictionary key `'__all__'` can be used, as shown here:

```py
import datetime
from typing import List

from pydantic import BaseModel, SecretStr


class Country(BaseModel):
    name: str
    phone_code: int


class Address(BaseModel):
    post_code: int
    country: Country


class CardDetails(BaseModel):
    number: SecretStr
    expires: datetime.date


class Hobby(BaseModel):
    name: str
    info: str


class User(BaseModel):
    first_name: str
    second_name: str
    address: Address
    card_details: CardDetails
    hobbies: List[Hobby]


user = User(
    first_name='John',
    second_name='Doe',
    address=Address(post_code=123456, country=Country(name='USA', phone_code=1)),
    card_details=CardDetails(
        number='4212934504460000', expires=datetime.date(2020, 5, 1)
    ),
    hobbies=[
        Hobby(name='Programming', info='Writing code and stuff'),
        Hobby(name='Gaming', info='Hell Yeah!!!'),
    ],
)

exclude_keys = {
    'second_name': True,
    'address': {'post_code': True, 'country': {'phone_code'}},
    'card_details': True,
    # You can exclude fields from specific members of a tuple/list by index:
    'hobbies': {-1: {'info'}},
}

include_keys = {
    'first_name': True,
    'address': {'country': {'name'}},
    'hobbies': {0: True, -1: {'name'}},
}

# would be the same as user.model_dump(exclude=exclude_keys) in this case:
print(user.model_dump(include=include_keys))
"""
{
    'first_name': 'John',
    'address': {'country': {'name': 'USA'}},
    'hobbies': [
        {'name': 'Programming', 'info': 'Writing code and stuff'},
        {'name': 'Gaming'},
    ],
}
"""

# To exclude a field from all members of a nested list or tuple, use "__all__":
print(user.model_dump(exclude={'hobbies': {'__all__': {'info'}}}))
"""
{
    'first_name': 'John',
    'second_name': 'Doe',
    'address': {'post_code': 123456, 'country': {'name': 'USA', 'phone_code': 1}},
    'card_details': {
        'number': SecretStr('**********'),
        'expires': datetime.date(2020, 5, 1),
    },
    'hobbies': [{'name': 'Programming'}, {'name': 'Gaming'}],
}
"""
```

The same holds for the `model_dump_json` method.

### Model- and field-level include and exclude

In addition to the explicit arguments `exclude` and `include` passed to `model_dump` and `model_dump_json` methods,
we can also pass the `include`/`exclude` arguments directly to the `Field` constructor:

```py
from pydantic import BaseModel, Field, SecretStr


class User(BaseModel):
    id: int
    username: str
    password: SecretStr = Field(..., exclude=True)


class Transaction(BaseModel):
    id: str
    user: User = Field(exclude={'username'})
    value: int = Field(exclude=True)


t = Transaction(
    id='1234567890',
    user=User(id=42, username='JohnDoe', password='hashedpassword'),
    value=9876543210,
)

print(t.model_dump())
#> {'id': '1234567890'}
# TODO: this is wrong! not all of "user" should be excluded
# TODO: do we need to fix the type of the argument to Field? Or do we want to just remove that functionality?
```

Explicitly setting `exclude`/`include` on `model_dump` and `model_dump_json` takes priority over the
`exclude`/`include` from the field constructor (i.e. `Field(..., exclude=True)`):

Note that while merging settings, `exclude` entries are merged by computing the "union" of keys, while `include`
entries are merged by computing the "intersection" of keys.

The resulting merged exclude settings:

```py
from pydantic import BaseModel, Field, SecretStr


class User(BaseModel):
    id: int
    username: str  # overridden by explicit exclude
    password: SecretStr = Field(exclude=True)


class Transaction(BaseModel):
    id: str
    user: User
    value: int


t = Transaction(
    id='1234567890',
    user=User(id=42, username='JohnDoe', password='hashedpassword'),
    value=9876543210,
)

print(t.model_dump(exclude={'value': True, 'user': {'username'}}))
#> {'id': '1234567890', 'user': {'id': 42}}
```

are the same as using merged include settings as follows:

```py
from pydantic import BaseModel, Field, SecretStr


class User(BaseModel):
    id: int = Field(..., include=True)
    username: str = Field(..., include=True)  # overridden by explicit include
    password: SecretStr


class Transaction(BaseModel):
    id: str
    user: User
    value: int


t = Transaction(
    id='1234567890',
    user=User(id=42, username='JohnDoe', password='hashedpassword'),
    value=9876543210,
)

print(t.model_dump(include={'id': True, 'user': {'id'}}))
#> {'id': '1234567890', 'user': {'id': 42}}
```

## `model_copy(...)`

`model_copy()` allows models to be duplicated (with optional updates), which is particularly useful when working with
frozen models. See the [API docs for `model_copy`](../api/main.md#pydantic.main.BaseModel.model_copy) for more
information.

Example:

```py
from pydantic import BaseModel


class BarModel(BaseModel):
    whatever: int


class FooBarModel(BaseModel):
    banana: float
    foo: str
    bar: BarModel


m = FooBarModel(banana=3.14, foo='hello', bar={'whatever': 123})

print(m.model_copy(update={'banana': 0}))
#> banana=0 foo='hello' bar=BarModel(whatever=123)
print(id(m.bar) == id(m.model_copy().bar))
#> True
# normal copy gives the same object reference for bar
print(id(m.bar) == id(m.model_copy(deep=True).bar))
#> False
# deep copy gives a new object reference for `bar`
```
