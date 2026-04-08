# Rust Uplift — Pattern Catalog

Detailed before/after examples for each pattern category. Reference this when building report findings.

## 1. Iterator Uplift

### Manual loop → combinators
```rust
// Before: mutable accumulator, imperative
let mut names = Vec::new();
for user in &users {
    if user.is_active() {
        names.push(user.name.to_uppercase());
    }
}

// After: declarative, no mutation
let names: Vec<_> = users.iter()
    .filter(|u| u.is_active())
    .map(|u| u.name.to_uppercase())
    .collect();
```

### Nested loops → flat_map
```rust
// Before
let mut all_items = Vec::new();
for order in &orders {
    for item in &order.items {
        all_items.push(item.clone());
    }
}

// After
let all_items: Vec<_> = orders.iter()
    .flat_map(|order| order.items.iter().cloned())
    .collect();
```

### Accumulator → fold/sum
```rust
// Before
let mut total: u64 = 0;
for item in &items {
    total += item.price * item.quantity;
}

// After
let total: u64 = items.iter()
    .map(|item| item.price * item.quantity)
    .sum();
```

### Search → find
```rust
// Before
let mut found = None;
for user in &users {
    if user.id == target_id {
        found = Some(user);
        break;
    }
}

// After
let found = users.iter().find(|u| u.id == target_id);
```

### Boolean check → any/all
```rust
// Before
let mut has_admin = false;
for user in &users {
    if user.role == Role::Admin {
        has_admin = true;
        break;
    }
}

// After
let has_admin = users.iter().any(|u| u.role == Role::Admin);
```

### itertools examples
```rust
use itertools::Itertools;

// Chunk consecutive elements by key (renamed from group_by in itertools 0.13+)
for (key, group) in &data.iter().chunk_by(|item| item.category) {
    // process each chunk
}

// Sliding windows
let pairs: Vec<_> = values.iter().tuple_windows::<(_, _)>().collect();

// Join strings
let csv = names.iter().join(", ");

// Unique elements
let unique: Vec<_> = items.iter().unique_by(|i| &i.id).collect();

// Sorted
let sorted: Vec<_> = items.iter().sorted_by_key(|i| &i.name).collect();
```

### rayon parallelism
```rust
use rayon::prelude::*;

// Before: sequential
let results: Vec<_> = data.iter().map(expensive_compute).collect();

// After: parallel (same API, prefix with par_)
let results: Vec<_> = data.par_iter().map(expensive_compute).collect();
```

---

## 2. Option/Result Combinator Uplift

### Nested match → map
```rust
// Before (moves nickname — fine if user is owned)
let display_name = match user.nickname {
    Some(nick) => Some(nick.to_uppercase()),
    None => None,
};

// After (consuming)
let display_name = user.nickname.map(|nick| nick.to_uppercase());

// After (borrowing — use .as_deref() to avoid consuming the field)
let display_name = user.nickname.as_deref().map(|nick| nick.to_uppercase());
```

### Chained Option → and_then
```rust
// Before
let city = match user.address {
    Some(addr) => match addr.city {
        Some(city) => Some(city),
        None => None,
    },
    None => None,
};

// After
let city = user.address.and_then(|addr| addr.city);
```

### Option to Result → ok_or
```rust
// Before
let user = match users.get(id) {
    Some(u) => u,
    None => return Err(AppError::NotFound("user")),
};

// After
let user = users.get(id).ok_or(AppError::NotFound("user"))?;
```

### Default fallback → unwrap_or / unwrap_or_else
```rust
// Before
let name = match config.name {
    Some(n) => n,
    None => String::from("default"),
};

// After
let name = config.name.unwrap_or_else(|| String::from("default"));
```

### Match-and-return → let-else (edition 2021+, Rust 1.65+)
```rust
// Before
let user = match users.get(id) {
    Some(u) => u,
    None => return HttpResponse::NotFound(),
};

// After
let Some(user) = users.get(id) else {
    return HttpResponse::NotFound();
};
```

### Verbose Result match → ? operator
```rust
// Before
let file = match File::open(path) {
    Ok(f) => f,
    Err(e) => return Err(e.into()),
};
let content = match std::io::read_to_string(&file) {
    Ok(s) => s,
    Err(e) => return Err(e.into()),
};

// After
let file = File::open(path)?;
let content = std::io::read_to_string(&file)?;
```

---

## 3. Error Handling Uplift

### Manual error enum → thiserror
```rust
// Before: ~40 lines of boilerplate
#[derive(Debug)]
enum AppError {
    Io(std::io::Error),
    Parse(serde_json::Error),
    NotFound(String),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "IO error: {}", e),
            AppError::Parse(e) => write!(f, "Parse error: {}", e),
            AppError::NotFound(s) => write!(f, "Not found: {}", s),
        }
    }
}

impl std::error::Error for AppError {}

impl From<std::io::Error> for AppError {
    fn from(e: std::io::Error) -> Self { AppError::Io(e) }
}

impl From<serde_json::Error> for AppError {
    fn from(e: serde_json::Error) -> Self { AppError::Parse(e) }
}

// After: ~12 lines with thiserror
#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    #[error("Parse error: {0}")]
    Parse(#[from] serde_json::Error),
    #[error("Not found: {0}")]
    NotFound(String),
}
```

### Application errors → anyhow
```rust
// Before
fn process() -> Result<Output, Box<dyn std::error::Error>> {
    let config = read_config().map_err(|e| format!("config error: {}", e))?;
    let data = fetch_data(&config).map_err(|e| format!("fetch error: {}", e))?;
    Ok(transform(data))
}

// After
use anyhow::{Context, Result};

fn process() -> Result<Output> {
    let config = read_config().context("failed to read config")?;
    let data = fetch_data(&config).context("failed to fetch data")?;
    Ok(transform(data))
}
```

---

## 4. Actor Pattern Uplift

### Arc<Mutex<T>> → Actor (Alice Ryhl pattern)
```rust
use std::collections::HashMap;
use tokio::sync::{mpsc, oneshot};

// Before: shared mutable state
struct SharedDb {
    data: Arc<Mutex<HashMap<String, String>>>,
}

async fn handle_request(db: SharedDb, key: String) -> Option<String> {
    let lock = db.data.lock().await;  // lock held during processing
    lock.get(&key).cloned()
}

// After: actor owns the state
// Message enum
enum DbMessage {
    Get {
        key: String,
        respond_to: oneshot::Sender<Option<String>>,
    },
    Set {
        key: String,
        value: String,
    },
}

// Actor
struct DbActor {
    receiver: mpsc::Receiver<DbMessage>,
    data: HashMap<String, String>,
}

impl DbActor {
    async fn run(mut self) {
        while let Some(msg) = self.receiver.recv().await {
            match msg {
                DbMessage::Get { key, respond_to } => {
                    // Ignore send error — if caller dropped rx, nothing to do
                    let _ = respond_to.send(self.data.get(&key).cloned());
                }
                DbMessage::Set { key, value } => {
                    self.data.insert(key, value);
                }
            }
        }
    }
}

// Handle (what callers use)
#[derive(Clone)]
struct DbHandle {
    sender: mpsc::Sender<DbMessage>,
}

impl DbHandle {
    fn new() -> Self {
        let (sender, receiver) = mpsc::channel(32);
        let actor = DbActor { receiver, data: HashMap::new() };
        tokio::spawn(actor.run());
        Self { sender }
    }

    async fn get(&self, key: String) -> Option<String> {
        let (tx, rx) = oneshot::channel();
        // Ignore send error — if actor shut down, rx.await returns Err
        let _ = self.sender.send(DbMessage::Get { key, respond_to: tx }).await;
        rx.await.ok().flatten()
    }
}
```

---

## 5. Type-State Uplift

### Runtime state checks → compile-time enforcement
```rust
// Sync example shown; same pattern works identically with async (tokio::net::TcpStream)
// Before: runtime panics on wrong state
struct Connection {
    state: ConnectionState,
    socket: Option<TcpStream>,
}

enum ConnectionState { Disconnected, Connected, Authenticated }

impl Connection {
    fn send(&self, data: &[u8]) -> Result<(), Error> {
        if self.state != ConnectionState::Authenticated {
            return Err(Error::NotAuthenticated); // runtime check
        }
        // ...
    }
}

// After: invalid states are unrepresentable
struct Disconnected;
struct Connected { socket: TcpStream }
struct Authenticated { socket: TcpStream, token: Token }

struct Connection<S> { state: S }

impl Connection<Disconnected> {
    fn connect(self, addr: &str) -> Result<Connection<Connected>, Error> {
        let socket = TcpStream::connect(addr)?;
        Ok(Connection { state: Connected { socket } })
    }
}

impl Connection<Connected> {
    fn authenticate(self, creds: &Credentials) -> Result<Connection<Authenticated>, Error> {
        let token = auth(&self.state.socket, creds)?;
        Ok(Connection { state: Authenticated { socket: self.state.socket, token } })
    }
}

impl Connection<Authenticated> {
    fn send(&self, data: &[u8]) -> Result<(), Error> {
        // Can only call send() on Authenticated connections — enforced by compiler
        Ok(())
    }
}
```

---

## 6. Derive & Macro Uplift

### Newtype boilerplate → derive_more
```rust
// Before: manual trait forwarding
struct UserId(u64);

impl std::fmt::Display for UserId {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl From<u64> for UserId {
    fn from(id: u64) -> Self { UserId(id) }
}

// After: derive_more 1.x (each derive is a separate feature and import)
use derive_more::{Display, From, Into};

#[derive(Debug, Display, From, Into, Clone, Copy, PartialEq, Eq, Hash)]
struct UserId(u64);
```

### Manual builder → bon
```rust
// Before: hand-written builder
struct ServerConfig { host: String, port: u16, workers: usize, tls: bool }

struct ServerConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    workers: Option<usize>,
    tls: Option<bool>,
}
// ... 30+ lines of impl with setters and build()

// After: bon
use bon::Builder;

#[derive(Builder)]
struct ServerConfig {
    host: String,
    port: u16,
    #[builder(default = 4)]
    workers: usize,
    #[builder(default)]
    tls: bool,
}

// Usage: ServerConfig::builder().host("0.0.0.0").port(8080).build()
```

### Manual CLI parsing → clap derive
```rust
// Before: manual env::args parsing
let args: Vec<String> = std::env::args().collect();
let input = args.get(1).expect("missing input file");
let verbose = args.iter().any(|a| a == "--verbose");

// After: clap derive
use clap::Parser;

#[derive(Parser)]
#[command(about = "Process data files")]
struct Args {
    /// Input file path
    input: PathBuf,
    /// Enable verbose output
    #[arg(short, long)]
    verbose: bool,
}

let args = Args::parse();
```

---

## 7. Ecosystem Crate Uplift

### log → tracing (async)
```rust
// Before: log messages interleave across async tasks
log::info!("processing request {}", request_id);
let result = process(request).await;
log::info!("done processing {}", request_id);  // which request?

// After: tracing spans track context across .await
use tracing::{info, instrument};

#[instrument(skip(request))]
async fn handle(request_id: u64, request: Request) -> Result<Response> {
    info!("processing request");
    let result = process(request).await;
    info!("done processing");  // span automatically includes request_id
    Ok(result)
}
```

### Raw SQL → sqlx compile-checked queries
```rust
// Before: typos and type mismatches caught at runtime
let row = sqlx::query("SELCT * FROM usrs WHERE id = $1")  // typo: not caught
    .bind(user_id)
    .fetch_one(&pool)
    .await?;

// After: checked against schema at compile time (PostgreSQL syntax shown; MySQL/SQLite use ?)
let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", user_id)
    .fetch_one(&pool)
    .await?;
```

---

## 8. Structural Uplift

### ..Default::default() — context matters
```rust
// In production/library code: prefer explicit fields so compiler catches new fields
let config = Config {
    host: "localhost".into(),
    port: 8080,
    workers: 4,
    tls_enabled: false,
    // Adding a new field to Config now causes a compile error here
};

// In tests: ..Default::default() is often fine — readability > exhaustiveness
let test_config = Config {
    host: "test-host".into(),
    ..Default::default()  // intentional: test only cares about host
};
```

### Many parameters → config struct
```rust
// Before: hard to read, easy to swap arguments
fn create_server(host: &str, port: u16, workers: usize, tls: bool, timeout: Duration, max_conn: usize) { }

// After: self-documenting
struct ServerConfig {
    host: String,
    port: u16,
    workers: usize,
    tls: bool,
    timeout: Duration,
    max_connections: usize,
}

fn create_server(config: ServerConfig) { }
```

---

## 9. Async Uplift

### Blocking in async → spawn_blocking
```rust
// Before: blocks the entire tokio runtime thread
async fn read_config() -> anyhow::Result<Config> {
    let content = std::fs::read_to_string("config.toml")?;  // BLOCKS
    Ok(toml::from_str(&content)?)
}

// After: offloaded to blocking thread pool
async fn read_config() -> anyhow::Result<Config> {
    let content = tokio::task::spawn_blocking(|| {
        std::fs::read_to_string("config.toml")
    }).await??;
    Ok(toml::from_str(&content)?)
}
```

### Manual join handles → try_join
```rust
// Before: sequential awaits
let users = fetch_users().await?;
let orders = fetch_orders().await?;
let inventory = fetch_inventory().await?;

// After: concurrent execution
let (users, orders, inventory) = tokio::try_join!(
    fetch_users(),
    fetch_orders(),
    fetch_inventory(),
)?;
```

---

## Expert References

These are the authoritative sources behind these patterns:

- **Effective Rust** (David Drysdale) — Items 3, 9 on transforms and iterators
- **Actors with Tokio** (Alice Ryhl) — Definitive actor pattern for async Rust
- **Async: What is Blocking?** (Alice Ryhl) — When spawn_blocking is needed
- **Be Simple** (corrode.dev) — When NOT to abstract
- **Defensive Programming in Rust** (corrode.dev) — Struct init pitfalls
- **The Typestate Pattern in Rust** (Cliffle) — Compile-time state machines
- **Rust Design Patterns** (rust-unofficial) — Community pattern catalog
- **Rust for Rustaceans** (Jon Gjengset) — Advanced API design and trait patterns
