```
|- Cargo.toml  # Workspace configuration
|- crates // All Rust code
   |- binance
   |- sim   // Fetch data from csv or database
   |- orderbook
      |- Cargo.toml
      |- src
         |- mod.rs
         |- orderbook.rs 
        |- handlers
           |- bar_handlers
           |- quote_tick_handlers
   |- orderbook            # The 
      |- Cargo.toml        # Define feature flags
      |- src
         |- mod.rs   
         |- orderbook.rs   # Models, ports, traits
         |- update-orderbook-by-bar.rs # Impl of UpdateOrderBookByBar trait
         |- update-orderbook-by-bar_http.rs # Impl UpdateOrderBookByBar for communication via http. Need feature flag
   |- bar-replay        # The feature of replaying the bar
```

# Module map

## Adapter

- Define the feature by APIs and data types that external party can provide.
- Register to websocket for data, converting the third party's type into the system's compatible data type.

## Data pipeline

When one data piece arrive, we need to distribute it to the one who need it.

# Mental philosophy

I don't want to write the service, which contains the internal state, then provide the list of methods from the service. The service become bigger and it's harder to write the unit test because of massive dependencies.

One module usually contains the state which is modified by the event, so we can replay later by applying the event. REMEMBER that not all module need to follow this pattern, only the module that contains the trading data need to use this pattern. For the module managing the transient data on memory, no need to define the event types, which make the code hard to read.

# Dependency injection

When creating an Action object, requiring many dependencies, we don't want to provide the instance manually.
When desgining the system, we need to know at which point or what module define all the related instance and group the instance in Provider.
So Dependency injection container have references to inject the instance.

File name: `<module_name>.di.rs`


# Module implementation

Usually module need to implement the entity, trait, core business without relying on external services
For the implementation depending on third-party library, it must define feature in Cargo.toml

```toml
[dependencies]
redis = { version = "0.24", optional = true }

[feature]
default = [] # Keep default empty so B gets nothing by default
storage-redis = ["dep:redis"]
```

```rust
#[cfg(feature = "storage-redis")]
pub struct RedisRepository {
    client: redis::Client,
}

#[cfg(feature = "storage-redis")]
impl UserRepository for RedisRepository {
    fn get_user(&self, id: i32) -> Option<User> {
        // Heavy redis logic here
        None
    }
}

```