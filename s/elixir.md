Elixir
======

Most Elixir applications use [Ecto](https://hexdocs.pm/ecto/Ecto.html) with
[ecto_sql](https://hexdocs.pm/ecto_sql/Ecto.Adapters.SQL.html) and a database
driver such as [Postgrex](https://hexdocs.pm/postgrex/Postgrex.html)
(PostgreSQL), [MyXQL](https://hexdocs.pm/myxql/MyXQL.html) (MySQL/MariaDB), or
[ecto_sqlite3](https://hexdocs.pm/ecto_sqlite3/Ecto.Adapters.SQLite3.html) (SQLite).
Ecto's query DSL and `Repo.query/2` and `Repo.query/3` (the latter when you pass
options) are built around **bound parameters**: values are sent separately from
the SQL text so the database never parses user data as part of the statement
structure.

## Dangerous patterns

Never build SQL strings from user input, whether through interpolation or
concatenation:

    # Do NOT do this - string interpolation, vulnerable to injection.
    sql = "UPDATE people SET name = '#{name}' WHERE id = #{id}"
    Repo.query!(sql, [])

    # Do NOT do this either - string concatenation, same risk.
    sql = "SELECT * FROM people WHERE name = '" <> name <> "'"
    Repo.query!(sql, [])

Both patterns hand the database one executable string. If `name` or `id` comes
from a user, an attacker can close the string literal and append arbitrary SQL.

## Ecto Query DSL

The safest path is Ecto's query DSL, which uses the **pin operator `^`** to
mark values as bound parameters -never as literal SQL text.

    import Ecto.Query

    from(p in Person, where: p.name == ^name) |> Repo.all()
    from(p in Person, where: p.id   == ^id)   |> Repo.one()

In these examples, `Repo` is your application's [`Ecto.Repo`](https://hexdocs.pm/ecto/Ecto.Repo.html)
module, usually aliased as `Repo` in contexts that `use MyApp.Repo`. Where a
fully-qualified name is clearer, `MyApp.Repo` is used instead.

For inserts and updates, use **changesets** and the corresponding `Repo`
functions. Ecto builds the parameterized query internally -no SQL is exposed to
user-supplied data:

    %Person{}
    |> Person.changeset(%{name: name, age: age})
    |> Repo.insert()

    person
    |> Person.changeset(%{name: name})
    |> Repo.update()

The pin operator works anywhere a value appears in a query expression:

    from(p in Person,
      where: p.age > ^min_age,
      limit: ^page_size
    )
    |> Repo.all()

### IN clauses

Pin a list directly to generate a safe `IN` predicate:

    ids = [1, 2, 3]
    from(p in Person, where: p.id in ^ids) |> Repo.all()

Ecto generates one bind parameter per element automatically.

### Dynamic queries

When the query shape depends on runtime conditions -for example, applying only
the filters the user actually provided -use
[`Ecto.Query.dynamic/2`](https://hexdocs.pm/ecto/Ecto.Query.html#dynamic/2)
to build the `where` expression incrementally. All values remain bound
parameters:

    defp build_filters(params) do
      Enum.reduce(params, dynamic(true), fn
        {:name, name}, acc -> dynamic([p], ^acc and p.name == ^name)
        {:age,  age},  acc -> dynamic([p], ^acc and p.age  > ^age)
        _,             acc -> acc
      end)
    end

    from(p in Person, where: ^build_filters(params)) |> Repo.all()

This eliminates the temptation to assemble SQL strings when the set of filters
is unknown at compile time.

If `params` is a Phoenix request map, keys are usually **strings** (`"name"`,
`"age"`), not atoms -normalize to the shape your `Enum.reduce` clauses expect
(for example with `Enum.map` / `Map.new` or by matching on string keys in the
`fn`). Otherwise filters can **silently not apply** because they fall through to
the catch-all clause.

## Raw SQL

When you need SQL that the DSL cannot express, keep the query text static and
pass all values in the parameter list. With the **PostgreSQL** adapter
(`$1`, `$2`, ...):

    Repo.query!("UPDATE people SET name = $1 WHERE id = $2", [name, id])

    Repo.query!("SELECT * FROM people WHERE name = $1", [name])

With the **MySQL/MariaDB** adapter (`?` for every parameter):

    Repo.query!("UPDATE people SET name = ? WHERE id = ?", [name, id])

    Repo.query!("SELECT * FROM people WHERE name = ?", [name])

Placeholder syntax is determined by the adapter; `ecto_sqlite3` also uses `?`.
Check your adapter's documentation when writing SQL by hand.

`Repo.query!/2` and `Repo.query!/3` raise on error; `Repo.query/2` and
`Repo.query/3` return `{:ok, result}` or `{:error, reason}`.

`Ecto.Adapters.SQL.query/4` is an alternative that names the repo explicitly:

    Ecto.Adapters.SQL.query(MyApp.Repo, "SELECT * FROM people WHERE id = $1", [id])

## `fragment/1`

When you need a raw SQL expression inside an Ecto query, use
[`fragment/1`](https://hexdocs.pm/ecto/Ecto.Query.API.html#fragment/1). Pass
runtime values through `?` placeholders -**not** through Elixir string
interpolation:

    # WRONG - interpolation inside the fragment string is injection-vulnerable.
    from(u in User, where: fragment("status = '#{status}'"))

    # Correct - ? placeholder keeps the value out of the SQL text.
    from(u in User, where: fragment("status = ?", ^status))

Schema field references are always safe because Ecto knows they come from your
schema, not user input:

    from(u in User, where: fragment("lower(?) = lower(?)", u.email, ^email))

## Dynamic identifiers

Table names and column names cannot be sent as bind parameters; the database
requires them in the query text itself. If a column name must vary at runtime
(for example, a user-chosen sort column), validate it against a compile-time
allowlist, then use [`field/2`](https://hexdocs.pm/ecto/Ecto.Query.API.html#field/2)
to reference it safely:

    @sortable_fields ~w(name age inserted_at)

    defp order_by_field(query, col_str) when col_str in @sortable_fields do
      col = String.to_existing_atom(col_str)
      order_by(query, [p], asc: field(p, ^col))
    end
    defp order_by_field(query, _), do: query

Use `String.to_existing_atom/1`, not `String.to_atom/1`. `String.to_atom/1`
creates a new atom for any input string, which can exhaust the atom table (a
denial-of-service risk) and lets callers name atoms your code never defined.
`String.to_existing_atom/1` raises for unknown atoms, making the guard
`when col_str in @sortable_fields` the real safety gate: it only allows strings
that correspond to atoms already present in the runtime. `field/2` then
references the schema field by that atom, keeping the identifier out of any raw
SQL string. In practice, every string that passes the guard will convert without
error because Ecto schema field names are registered as atoms when the module is
compiled.

The same idea extends to sort direction. Do not interpolate `"asc"` or
`"desc"` into a SQL string - validate against a fixed list and convert:

    @sort_directions ~w(asc desc)

    defp order_by_field(query, col_str, dir_str) when col_str in @sortable_fields and dir_str in @sort_directions do
      col = String.to_existing_atom(col_str)
      dir = String.to_existing_atom(dir_str)
      order_by(query, [p], [{^dir, field(p, ^col)}])
    end
    defp order_by_field(query, _, _), do: query

## LIKE wildcards

Parameterization prevents SQL injection but does not neutralize the special
characters `%` (match any sequence) and `_` (match any single character) inside
`LIKE` patterns. A user who submits `%` as a search term can match every row.

**PostgreSQL:** unless you add an `ESCAPE` clause, backslash is **not** special
in `LIKE`, so patterns that rely on `\%` or `\_` will not behave the way many
other databases imply. Pick an escape character that you forbid or normalize in
input (here `!`), double it when it appears literally, prefix wildcards with it,
then declare `ESCAPE '!'` in the SQL:

    safe_term =
      input
      |> String.replace("!", "!!")
      |> String.replace("%", "!%")
      |> String.replace("_", "!_")

    search = "%" <> safe_term <> "%"
    from(u in User, where: fragment("? LIKE ? ESCAPE '!'", u.name, ^search))

Other engines use different default rules for `LIKE` and escaping; read your
database's documentation instead of assuming backslash escapes wildcards.

Ecto's `ilike/2` is case-insensitive matching with the same `%` and `_`
wildcard semantics -use the same escaping approach when you need literal
substring matches. `ilike/2` is **PostgreSQL-only**; on other databases, use
`like/2` combined with a database-level case-folding strategy (e.g. `lower()`
via `fragment`).

## Further reading

- [Ecto.Query](https://hexdocs.pm/ecto/Ecto.Query.html)
- [Ecto.Repo - `query/2`, `query/3`, `query!/2`, `query!/3`](https://hexdocs.pm/ecto/Ecto.Repo.html#query/2)
- [Ecto.Query.API - `dynamic/2`, `field/2`, `fragment/1`, `ilike/2`, `like/2`](https://hexdocs.pm/ecto/Ecto.Query.API.html)
- [Ecto.Adapters.SQL - `query/4`, `query!/4`](https://hexdocs.pm/ecto_sql/Ecto.Adapters.SQL.html#query/4)
- [Ecto.Changeset](https://hexdocs.pm/ecto/Ecto.Changeset.html)
