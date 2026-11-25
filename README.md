# Nest

Entities, tables and migrations for Antril programs, over `System.Data`. The mapping is made when compiling: each entity
class gets its own code (one statement per field) for its SQL, its parameters and reading its rows, so nothing is looked
up while the program runs.

## Entities and the database

```antril
import System.Data.Sqlite;
import Nest;

public class User : IEntity<User>
{
	public:
		[Key] i64 Id;                       // an integer key is made by the database (Add sets it)
		[Unique] str Email;
		[Column("full_name")] str Name;
		i32 Age;
		str? Phone;                         // nullable: NULL allowed
		[Ignore] i32 Scratch;               // not a column

		new() { Email = ""; Name = ""; }    // rows are read into a new object
}

public class ShopDb : Database, IModel<ShopDb>
{
	public:
		Table<User> Users;

		new(str path) : base(SqliteConnection.Open(path)) { OpenTables(Connection); }
}
```

Fields map from integers, `bool`, `f32`, `f64`, `str`, `u8[]` and their nullables (`str?`, `u8[]?`, `i64?`, `i32?`,
`f64?`, `bool?`). `static TableName() : str` in the class names its table (by default the class's name).

```antril
ShopDb db = new ShopDb("shop.db");
db.Migrate(Migrations.All());            // the migrations below, those the database has not had

User ana = new User();
ana.Email = "ana@x.org";
db.Users.Add(ana);                       // ana.Id is set
if (db.Users.Find(ana.Id) is User found) { … }
db.Users.Update(ana);
db.Users.Remove(ana);
loop (User u in db.Users.Where("Age >= 18"))
	…
db.InTransaction(() => { … });
```

A table prepares each statement once and keeps it.

## Queries from functions

```antril
i32 limit = 18;
str prefix = "A";
loop (User u in db.Users.Where(u => u.Age >= limit && u.Name.StartsWith(prefix)).OrderBy(u => u.Name).Take(20))
	…
i64 adults = db.Users.Where(u => u.Age >= 18).Count();
User? first = db.Users.Where(u => u.Phone == null).OrderByDescending(u => u.Id).First();
i64 gone = db.Users.Where(u => u.Age > 120).Delete();          // one DELETE, nothing read
```

`Where` and `OrderBy` take quoted functions (`const func<T> : bool`): the compiler hands them over with their body as a
tree, and each sub-expression that does not name the entity (`limit`, `prefix`, `limit * 2`) as a value worked out at the
call. The SQL is made from the trees the first time a query's places run and kept with its prepared statement; after
that a query is binding those values and stepping (about 0.1 µs more than the same statement written by hand).

`larva build` translates them while building: `Where` and `OrderBy` name their translator (`[CompileWith("Nest.Sql.Where")]`),
which larva runs on every function written there, so a query that SQL cannot say is an error of the build, at the
function:

```
Main.ant(59,18): error: `u.Name.Replace("a", "b")`: `Replace` cannot be said in SQL
```

and the program carries the SQL as a constant (nothing is translated while it runs). Translations are kept between builds
(by the translator's library, the entity and the tree), so a build only translates the functions that changed. Code
that is not built by `larva build` (the VM, the editor) translates a function the first time its place runs.

What a query can say: the entity's fields (`u.Name`: its column), values from around it, `null` (`IS NULL`), `== != < <=
> >=`, `&& || !`, `+ - * / %`, `??`, `c ? a : b`, and on texts `StartsWith`, `EndsWith`, `Contains`, `ToUpper`,
`ToLower`, `Trim`, `Length`. Anything else is a `QueryError` saying what (`Replace` cannot be said in SQL). Queries:
`Where`, `OrderBy`, `OrderByDescending`, `Take`, `Skip`, then `loop`, `ToList`, `First`, `Count`, `Any`, `Delete`.

## Migrations: `larva nest`

The tool is a larva tool: install it in the project (it is written to the project's `TOOLS`):

```
larva tool install Antril-Organization/Nest.Tool
```

| Command | What it does |
|---|---|
| `larva nest model` | the model the code declares (its tables and columns) |
| `larva nest migrate add Name` | `migrations/M0001_Name.ant`: from the last migration's model to the code's |
| `larva nest migrate script` | the SQL such a migration would run, written nowhere |
| `larva nest migrate check` | exit code 1 when the code has changes no migration has (for CI) |
| `larva nest migrate list` | the migrations there are |

The tool builds the project with a program of its own (its entry in place of `Main`) that prints the model, and compares
it with `migrations/model.txt`, the model the last migration left. Migrations are Antril classes with `Up` and `Down`:
edit them when data has to move. `migrations/Migrations.ant` lists them (`Migrations.All()`); `db.Migrate(…)` applies the
pending ones, each in a transaction, noted in the database's `__nest_migrations` table; `db.Revert(all, n)` undoes those
after number `n`.

### Settings: `nest.queen`

A project can say how `larva nest` works in a `nest.queen` next to its `project.queen`. The tool brings its profile
(`tool/profiles/nest/template.tqueen`), so where it is installed the editor checks and completes the file, and the tool
reads it with `larva queen read`. Without one, the defaults apply.

```queen
queen 1.0;

chunk NEST:
   model: Shop.Data.ShopDb,
   migrations: data/migrations
```

| Key | What it says |
|---|---|
| `model` | the IModel class (by its name, or with its namespace): needed when the project has more than one; `--db Name` wins for one command |
| `migrations` | the folder, under the project, where migrations are written and read (default `migrations`) |

What a migration does (SQLite):

- a table added or dropped: `CREATE TABLE` / `DROP TABLE`;
- a column added or dropped: `ALTER TABLE … ADD COLUMN` (a `NOT NULL` one gets its type's empty value) / `DROP COLUMN`;
- a column renamed (`[Renamed("Old")]` on the field): `RENAME COLUMN`, its values kept;
- anything else (a type, `NOT NULL`, `UNIQUE`, the key): the table is made again and its rows copied (a new `UNIQUE NOT
  NULL` column gets each row's id: change it in `Up` when the rows need other values).

## Layout

- `src/`: the library (`Nest.dll`: `larva lib -o bin/Nest.dll src/*.ant`)
- `tool/`: the tool (`tool/project.queen`, `type: tool`, command `nest`), and its queen profile (`tool/profiles/nest/`)
