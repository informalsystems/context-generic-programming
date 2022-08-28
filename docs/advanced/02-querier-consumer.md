# Querier Consumer


```rust
# use std::fmt::Display;
# use std::convert::{TryFrom, TryInto};
#
# trait NamedPerson {
#      fn name(&self) -> &str;
# }
#
# trait HasError {
#      type Error;
# }
#
# trait PersonContext {
#      type PersonId;
#      type Person: NamedPerson;
# }
#
# trait Greeter<Context>
# where
#      Context: PersonContext + HasError,
# {
#      fn greet(&self, context: &Context, person_id: &Context::PersonId)
#          -> Result<(), Context::Error>;
# }
#
# trait KvStore: HasError {
#      fn get(&self, key: &str) -> Result<Vec<u8>, Self::Error>;
# }
#
# trait KvStoreContext {
#      type Store: KvStore;
#
#      fn store(&self) -> &Self::Store;
# }
#
# struct KvStorePersonQuerier;
#
# impl<Context, Store, PersonId, Person, Error, ParseError, StoreError>
#      PersonQuerier<Context> for KvStorePersonQuerier
# where
#      Context: KvStoreContext<Store=Store>,
#      Context: PersonContext<Person=Person, PersonId=PersonId>,
#      Context: HasError<Error=Error>,
#      Store: KvStore<Error=StoreError>,
#      PersonId: Display,
#      Person: TryFrom<Vec<u8>, Error=ParseError>,
#      Error: From<StoreError>,
#      Error: From<ParseError>,
# {
#      fn query_person(context: &Context, person_id: &PersonId)
#          -> Result<Person, Error>
#      {
#          let key = format!("persons/{}", person_id);
#
#          let bytes = context.store().get(&key)?;
#
#          let person = bytes.try_into()?;
#
#          Ok(person)
#      }
# }
#
# trait PersonQuerier<Context>
# where
#      Context: PersonContext + HasError,
# {
#         fn query_person(context: &Context, person_id: &Context::PersonId)
#             -> Result<Context::Person, Context::Error>;
# }
#
struct SimpleGreeter;

impl<Context> Greeter<Context> for SimpleGreeter
where
    Context: PersonContext + HasError,
    KvStorePersonQuerier: PersonQuerier<Context>,
{
    fn greet(&self, context: &Context, person_id: &Context::PersonId)
        -> Result<(), Context::Error>
    {
        let person = KvStorePersonQuerier::query_person(context, person_id)?;
        println!("Hello, {}", person.name());
        Ok(())
    }
}
```

## Generic Querier Consumer

Now that we have a context-generic implementation of `KvStorePersonQuerier`,
we can try to use it from `SimpleGreeter`. To do that, `SimpleGreeter` has
to somehow get `KvStorePersonQuerier` from `Context` and use it as a
`PersonQuerier`.

Recall that `KvStorePersonQuerier` itself is not a context (though it does
implement `PersonQuerier<Context>` in order to query a context), and therefore
it does not implement other context traits like `PersonContext`. What we
need instead is for concrete contexts like `AppContext` to specify that
their implementation of `PersonQuerier` is `KvStorePersonQuerier`.
We can do that by defining a `HasPersonQuerier` trait as follows:

```rust
# trait NamedPerson {
#      fn name(&self) -> &str;
# }
#
# trait HasError {
#      type Error;
# }
#
# trait PersonContext {
#      type PersonId;
#      type Person: NamedPerson;
# }
#
# trait Greeter<Context>
# where
#      Context: PersonContext + HasError,
# {
#      fn greet(&self, context: &Context, person_id: &Context::PersonId)
#          -> Result<(), Context::Error>;
# }
#
trait PersonQuerier<Context>
where
    Context: PersonContext + HasError,
{
     fn query_person(context: &Context, person_id: &Context::PersonId)
         -> Result<Context::Person, Context::Error>;
}

trait HasPersonQuerier:
    PersonContext + HasError + Sized
{
    type PersonQuerier: PersonQuerier<Self>;

    fn query_person(&self, person_id: &Self::PersonId)
        -> Result<Self::Person, Self::Error>
    {
        Self::PersonQuerier::query_person(self, person_id)
    }
}
```

While the `PersonQuerier` trait is implemented by component types like
`KvStorePersonQuerier`, the `HasPersonQuerier` trait is implemented by
context types like `AppContext`. Compared to the earlier design of
`PersonQuerier`, the context is now offering a _component_ for
querying for a person that will work in the _current context_.

We can see that the `HasPersonQuerier` trait has `PersonContext`
and `HasError` as its supertraits, indicating that the concrete context
also needs to implement these two traits first. Due to quirks in Rust,
the trait also requires the `Sized` supertrait, which is already implemented
by most types other than `dyn Trait` types, so that we can use `Self` inside
other generic parameters.

In the body of `HasPersonQuerier`, we define a `PersonQuerier` associated
type, which implements the trait `PersonQuerier<Self>`. This is because
we want to have the following constraints satisfied:

- `AppContext: HasPersonQuerier` - `AppContext` implements the trait `HasPersonQuerier`.
- `AppContext::PersonQuerier: PersonQuerier<AppContext>` - The associated type
  `AppContext::PersonQuerier` implements the trait `PersonQuerier<AppContext>`.
- `KvStorePersonQuerier: PersonQuerier<AppContext>` - The type `KvStorePersonQuerier`,
  which we defined earlier, should implement `PersonQuerier<AppContext>`.
- `AppContext: HasPersonQuerier<PersonQuerier=KvStorePersonQuerier>` - We want to
  set the associated type `AppContext::PersonQuerier` to be `KvStorePersonQuerier`.

In general, since we want any type `Ctx` that implements `HasPersonQuerier` to
have the associated type `Ctx::PersonQuerier` to implement `PersonQuerier<Ctx>`.
Hence inside the trait definition, we define the associated type as
`type PersonQuerier: PersonQuerier<Self>`, where `Self` refers to the `Ctx` type.

This may look a little self-referential, as the context is providing a type
that is referencing back to itself. But with the dependency injection mechanism
of the traits system, this in fact works most of the time as long as there are
no actual cyclic dependencies.

Inside `HasPersonQuerier`, we also implement a `query_person` method with a `&self`,
which calls `Self::PersonQuerier::query_person` to do the actual query. This method
is not meant to be overridden by implementations. Rather, it is a convenient method
that allows us to query from the context directly using `context.query_person()`.

Now inside the `Greet` implementation for `SimpleGreeter`, we can require the
generic `Context` to implement `HasPersonQuerier` as follows:


```rust
# trait NamedPerson {
#      fn name(&self) -> &str;
# }
#
# trait HasError {
#      type Error;
# }
#
# trait PersonContext {
#      type PersonId;
#      type Person: NamedPerson;
# }
#
# trait Greeter<Context>
# where
#      Context: PersonContext + HasError,
# {
#      fn greet(&self, context: &Context, person_id: &Context::PersonId)
#          -> Result<(), Context::Error>;
# }
#
# trait PersonQuerier<Context>
# where
#     Context: PersonContext + HasError,
# {
#      fn query_person(context: &Context, person_id: &Context::PersonId)
#          -> Result<Context::Person, Context::Error>;
# }
#
# trait HasPersonQuerier:
#     PersonContext + HasError + Sized
# {
#     type PersonQuerier: PersonQuerier<Self>;
#
#     fn query_person(&self, person_id: &Self::PersonId)
#         -> Result<Self::Person, Self::Error>
#     {
#         Self::PersonQuerier::query_person(self, person_id)
#     }
# }
#
struct SimpleGreeter;

impl<Context> Greeter<Context> for SimpleGreeter
where
    Context: HasPersonQuerier,
{
    fn greet(&self, context: &Context, person_id: &Context::PersonId)
        -> Result<(), Context::Error>
    {
        let person = context.query_person(person_id)?;
        println!("Hello, {}", person.name());
        Ok(())
    }
}
```


Inside the `greet` method, we can call `context.query_person()` and pass in
the `context` as the first argument to query for the person details.

In summary, what we achieved at this point is as follows:

- We define a context-generic component for `PersonQuerier` as
    `KvStorePersonQuerier`.
- We define another context-generic component for `Greet` as `SimpleGreeter`,
    which _depends_ on a `PersonQuerier` component provided from the context.
- The Rust trait system _resolves_ the dependency graph, constructs a
    `KvStorePersonQuerier` using its _indirect dependencies_ from the context,
    and passes it as the `PersonQuerier` dependency to `SimpleGreeter`.

By using dependency injection, we don't need to know about the fact that in
order to build `SimpleGreeter`, we need to first build `KvStorePersonQuerier`,
but in order to build `KvStorePersonQuerier`, we need to first build
`FsKvStore`.

By leveraging dependency injection, we don't need to know that building
`SimpleGreeter` requires first building `KvStorePersonQuerier`, which itself
requires first building `FsKvStore`. The compiler resolves all of these
dependencies at compile time for free, and we do not even need to pay for the
cost of doing such wiring at run time.

## Selfless Components

```rust
# trait NamedPerson {
#      fn name(&self) -> &str;
# }
#
# trait HasError {
#      type Error;
# }
#
# trait PersonContext {
#      type PersonId;
#      type Person: NamedPerson;
# }
#
# trait Greeter<Context>
# where
#      Context: PersonContext + HasError,
# {
#      fn greet(&self, context: &Context, person_id: &Context::PersonId)
#          -> Result<(), Context::Error>;
# }
#
trait PersonQuerier<Context>
where
    Context: PersonContext + HasError,
{
     fn query_person(&self, context: &Context, person_id: &Context::PersonId)
         -> Result<Context::Person, Context::Error>;
}

trait HasPersonQuerier:
    PersonContext + HasError + Sized
{
    type PersonQuerier: PersonQuerier<Self>;

    fn person_querier(&self) -> &Self::PersonQuerier;
}

struct SimpleGreeter;

impl<Context> Greeter<Context> for SimpleGreeter
where
    Context: HasPersonQuerier,
{
    fn greet(&self, context: &Context, person_id: &Context::PersonId)
        -> Result<(), Context::Error>
    {
        let person = context.person_querier().query_person(context, person_id)?;
        println!("Hello, {}", person.name());
        Ok(())
    }
}
```
