# Writing Language Injections

This is an example of how to setup pruner to format SQL embedded inside of Rust. Take the following Rust snippet with a
SQL query as a reference:

```rust
async fn get_users(pool: &PgPool) -> Result<Vec<User>> {
  let query = "SELECT
                 id,
                 name,
                 email
               FROM
                 users
               WHERE
                 active = TRUE
               ORDER BY
                 created_at DESC";
  let users = sqlx::query_as::<_, User>(query).fetch_all(pool).await?;
  Ok(users)
}
```

By default the query string is treated as an opaque string and Pruner has no idea that it contains SQL and should be
formatted as such.

We can change this by defining a simple treesitter language injection query:

```query
;; queries/rust/injections.scm
((string_content) @injection.content
  ;; Regex match for a string containing text that looks like a SQL query
  (#match? @injection.content "^(SET|TRUNCATE|SELECT|CREATE|DELETE|ALTER|UPDATE|DROP|INSERT|WITH)")
  ;; Inform Pruner that any '"' characters should be escaped.
  (#escape! @injection.content "\"")
  ;; Label this region as SQL
  (#set! injection.language "sql"))
```

Next Pruner will need to be configured appropriately:

- We need to add the `queries` directory to Pruners' query search paths
- We need to configure pruner to format Rust and SQL
- We need to add the Rust treesitter grammar to allow parsing the rust code

```toml
query_paths = ["queries"]

[grammars]
rust = "https://github.com/tree-sitter/tree-sitter-rust"

[formatters]
rustfmt = { cmd = "rustfmt", args = ["--emit=stdout", "--edition=2024"] }

pg_format = { cmd = "pg_format", args = [
  "--spaces=2",
  "--wrap-limit=$textwidth",
  "-",
] }

[languages]
sql = ["pg_format"]
rust = ["rustfmt"]
```

And that should be sufficient. Now you can run `pruner format --lang rust` against the above source code and the
embedded SQL will be formatted as well.

## Query Directives

In addition to the standard
[treesitter directives and predicates](https://tree-sitter.github.io/tree-sitter/using-parsers/queries/3-predicates-and-directives.html),
Pruner also implements some other important directives which can be very useful when writing custom injection queries.

#### `#offset!`

This directive is taken directly [from Neovim](https://neovim.io/doc/user/treesitter.html#treesitter-directive-offset!)
and effectively enables offsetting the range of a captured node by a defined amount.

**Parameters**:

- {capture_id}
- {start_row}
- {start_col}
- {end_row}
- {end_col}

**Example**:

```query
(#offset! @injection.content 0 1 0 -1)
```

#### `#escape!`

This directive informs pruner that the content of a embedded language should have any occurances of a given character
escaped.

For example, here is some SQL which contains `"` characters embedded in a rust string (surrounded by `"` characters).
The inner SQL chars need escaping.

```rust
let query = "CREATE DOMAIN accounting.kid text COLLATE \"C\"";
```

**Parameters**:

- {capture_id}
- {character}

**Example**:

```query
(#escape! @injection.content "\"")
```

## Editor Integration

If you want the above Rust snippet to also be highlighted like SQL in your editor, and you are using Neovim (or an
editor which uses treesitter under the hood), then you can!

For example, if you are on Neovim, you can take the same injection query defined above and place it at
`~/.config/nvim/queries/rust/injections.scm` and you will get SQL highlighting for the same snippet!

You can read more about this in [this article](https://julienvincent.io/posts/treesitter-language-injections).
