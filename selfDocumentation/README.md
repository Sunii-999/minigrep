# Own documentation explained in own terms

_Source [here](https://doc.rust-lang.org/book/ch12-00-an-io-project.html)_

## Imports

```rust
use std::env;
use std::fs;
use std::process;
use std::error::Error;
```

- `env` → read CLI arguments and environment variables
- `fs` → read files
- `process` → exit the program with a status code
- `Error` → trait for generic error handling (`Box<dyn Error>`)

```rust
use minigrep::{search, search_case_insensitive};
```

- Imports **library logic** from your crate
- Keeps searching logic **out of `main`**, improving modularity

---

### Argument parsing

```rust
let config = Config::build(env::args()).unwrap_or_else(|err| {
    eprintln!("Problem parsing arguments: {err}");
    process::exit(1);
});
```

- `env::args()` → iterator of CLI arguments
- Passed to `Config::build`
- `build` returns `Result<Config, &str>`
- `unwrap_or_else`:

  - If `Ok` → continue
  - If `Err` → print error and exit cleanly

---

### Status output

```rust
println!("Searching for: {}", config.query);
println!("Insilde file: {}", config.file_path);
```

- Uses data stored in `Config`
- Confirms what the program will do

---

### Running core logic

```rust
if let Err(e) = run(config) {
    eprintln!("Application error: {e}");
    process::exit(1)
}
```

- Calls `run`, which returns `Result`
- If an error occurs:

  - Print it
  - Exit with error code

**Key idea**

- `main` handles _fatal_ errors
- `run` reports errors, but does not decide how to exit

---

## `fn run(config: Config) -> Result<(), Box<dyn Error>>`

### Responsibility

- Executes the **core application logic**
- Returns errors instead of crashing
- Is reusable and testable

---

### Reading the file

```rust
let contents = fs::read_to_string(config.file_path)?;
```

- Reads entire file into a `String`
- `?` operator:

  - If success → continue
  - If error → return early with the error

**Why `Box<dyn Error>`**

- Allows returning _any_ error type
- Keeps function flexible and simple

---

### Choosing search strategy

```rust
let results = if config.ignore_case {
    search_case_insensitive(&config.query, &contents)
} else {
    search(&config.query, &contents)
};
```

- Decision based on configuration
- Delegates searching to library functions
- `run` does not know _how_ searching works

---

### Output results

```rust
for line in results {
    println!("{line}");
}
```

- Iterates over matched lines
- Prints each match

---

### Successful completion

```rust
Ok(())
```

- Signals success to `main`

---

## `pub struct Config`

```rust
pub struct Config {
    pub query: String,
    pub file_path: String,
    pub ignore_case: bool,
}
```

### Purpose

- Holds **all configuration/state** for the program
- Groups related values together

### Why `pub`

- Fields must be accessible from `main` and `run`
- Represents shared program state

### Why a struct?

- Makes intent explicit
- Reduces argument passing
- Easier to extend later

---

## `impl Config`

```rust
impl Config {
```

- Defines behavior **associated with `Config`**
- Keeps configuration logic close to the data

---

## `fn build(...) -> Result<Config, &'static str>`

```rust
fn build(
    mut args: impl Iterator<Item = String>,
) -> Result<Config, &'static str>
```

### Responsibility

- Constructs a valid `Config`
- Validates CLI input
- Fails gracefully with meaningful errors

---

### Skipping program name

```rust
args.next();
```

- First CLI argument is the program name
- Not needed for configuration

---

### Reading query argument

```rust
let query = match args.next() {
    Some(arg) => arg,
    None => return Err("Didn't get a query string"),
};
```

- Attempts to read the next argument
- If missing → return an error instead of panicking

---

### Reading file path

```rust
let file_path = match args.next() {
    Some(arg) => arg,
    None => return Err("Didn't get a file path"),
};
```

- Same validation pattern
- Prevents index-out-of-bounds errors

---

### Environment variable handling

```rust
let ignore_case = env::var("IGNORE_CASE").is_ok();
```

- Checks if `IGNORE_CASE` exists
- Value doesn’t matter — only presence
- Enables feature without changing CLI syntax

---

### Returning Config

```rust
Ok(Config {
    query,
    file_path,
    ignore_case,
})
```

- If all inputs are valid
- Returns a fully constructed configuration object

---

## How everything communicates together

1. **`main`**

   - Calls `Config::build`
   - Calls `run`
   - Handles fatal errors

2. **`Config::build`**

   - Reads CLI args + env vars
   - Returns validated configuration

3. **`run`**

   - Receives `Config`
   - Reads file
   - Calls `search` functions
   - Prints results
   - Returns success or error

4. **`search` functions**

   - Pure logic
   - No I/O
   - Easy to test
