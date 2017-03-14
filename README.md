[![Build Status](https://circleci.com/gh/izelnakri/paper_trail.svg?style=shield&circle-token=:circle-token)](https://circleci.com/gh/izelnakri/paper_trail) [![Hex Version](http://img.shields.io/hexpm/v/paper_trail.svg?style=flat)](https://hex.pm/packages/paper_trail) [![Hex docs](http://img.shields.io/badge/hex.pm-docs-green.svg?style=flat)](https://hexdocs.pm/paper_trail/PaperTrail.html)

# How does it work?

PaperTrail lets you record every change in your database in a separate database table called ```versions```. Library generates a new version record with associated data every time you run ```PaperTrail.insert/1```, ```PaperTrail.update/1``` or ```PaperTrail.delete/1``` functions. Simply these functions wrap your Repo insert, update or destroy actions in a database transaction, so if your database action fails you won't get a new version.

PaperTrail is assailed with hundreds of test assertions for each release. Data integrity is an important purpose of this project, please refer to the strict_mode if you want to ensure data correctness and integrity of your versions. For simpler use cases the default mode of PaperTrail should suffice.

## Example

```elixir
  changeset = Post.changeset(%Post{}, %{
    title: "Word on the street is Elixir got its own database versioning library",
    content: "You should try it now!"
  })

  PaperTrail.insert(changeset)
  # => on success:
  # {:ok,
  #  %{model: %Post{__meta__: #Ecto.Schema.Metadata<:loaded, "posts">,
  #     title: "Word on the street is Elixir got its own database versioning library",
  #     content: "You should try it now!", id: 1, inserted_at: #Ecto.DateTime<2016-09-15 21:42:38>,
  #     updated_at: #Ecto.DateTime<2016-09-15 21:42:38>},
  #    version: %PaperTrail.Version{__meta__: #Ecto.Schema.Metadata<:loaded, "versions">,
  #     event: "insert", id: 1, inserted_at: #Ecto.DateTime<2016-09-15 21:42:38>,
  #     item_changes: %{title: "Word on the street is Elixir got its own database versioning library",
  #       content: "You should try it now!", id: 1, inserted_at: #Ecto.DateTime<2016-09-15 21:42:38>,
  #       updated_at: #Ecto.DateTime<2016-09-15 21:42:38>},
  #     item_id: 1, item_type: "Post", originator_id: nil, originator: nil, meta: nil}}}

  # => on error(it matches Repo.insert\2):
  # {:error, Ecto.Changeset<action: :insert,
  #  changes: %{title: "Word on the street is Elixir got its own database versioning library", content: "You should try it now!"},
  #  errors: [content: {"is too short", []}], data: #Post<>,
  #  valid?: false>, %{}}

  post = Repo.get!(Post, 1)
  edit_changeset = Post.changeset(post, %{
    title: "Elixir matures fast",
    content: "Future is already here, you deserve to be awesome!"
  })

  PaperTrail.update(edit_changeset)
  # => on success:
  # {:ok,
  #  %{model: %Post{__meta__: #Ecto.Schema.Metadata<:loaded, "posts">,
  #     title: "Elixir matures fast", content: "Future is already here, you deserve to be awesome!",
  #     id: 1, inserted_at: #Ecto.DateTime<2016-09-15 21:42:38>,
  #     updated_at: #Ecto.DateTime<2016-09-15 22:00:59>},
  #    version: %PaperTrail.Version{__meta__: #Ecto.Schema.Metadata<:loaded, "versions">,
  #     event: "update", id: 2, inserted_at: #Ecto.DateTime<2016-09-15 22:00:59>,
  #     item_changes: %{title: "Elixir matures fast", content: "Future is already here, you deserve to be awesome!"},
  #     item_id: 1, item_type: "Post", originator_id: nil, originator: nil
  #     meta: nil}}}

  # => on error(it matches Repo.update\2):
  # {:error, Ecto.Changeset<action: :update,
  #  changes: %{title: "Elixir matures fast", content: "Future is already here, you deserve to be awesome!"},
  #  errors: [title: {"is too short", []}], data: #Post<>,
  #  valid?: false>, %{}}

  PaperTrail.get_version(post)
  #  %PaperTrail.Version{__meta__: #Ecto.Schema.Metadata<:loaded, "versions">,
  #   event: "update", id: 2, inserted_at: #Ecto.DateTime<2016-09-15 22:00:59>,
  #   item_changes: %{title: "Elixir matures fast", content: "Future is already here, you deserve to be awesome!"},
  #   item_id: 1, item_type: "Post", originator_id: nil, originator: nil, meta: nil}}}

  updated_post = Repo.get!(Post, 1)

  PaperTrail.delete(updated_post)
  # => on success:
  # {:ok,
  #  %{model: %Post{__meta__: #Ecto.Schema.Metadata<:deleted, "posts">,
  #     title: "Elixir matures fast", content: "Future is already here, you deserve to be awesome!",
  #     id: 1, inserted_at: #Ecto.DateTime<2016-09-15 21:42:38>,
  #     updated_at: #Ecto.DateTime<2016-09-15 22:00:59>},
  #    version: %PaperTrail.Version{__meta__: #Ecto.Schema.Metadata<:loaded, "versions">,
  #     event: "delete", id: 3, inserted_at: #Ecto.DateTime<2016-09-15 22:22:12>,
  #     item_changes: %{title: "Elixir matures fast", content: "Future is already here, you deserve to be awesome!",
  #       id: 1, inserted_at: #Ecto.DateTime<2016-09-15 21:42:38>,
  #       updated_at: #Ecto.DateTime<2016-09-15 22:00:59>},
  #     item_id: 1, item_type: "Post", originator_id: nil, originator: nil, meta: nil}}}

  Repo.aggregate(Post, :count, :id) # => 0
  Repo.aggregate(PaperTrail.Version, :count, :id) # => 3

  last(PaperTrail.Version, :id) |> Repo.one
  #  %PaperTrail.Version{__meta__: #Ecto.Schema.Metadata<:loaded, "versions">,
  #   event: "delete", id: 3, inserted_at: #Ecto.DateTime<2016-09-15 22:22:12>,
  #   item_changes: %{"title" => "Elixir matures fast", content: "Future is already here, you deserve to be awesome!", "id" => 1,
  #     "inserted_at" => "2016-09-15T21:42:38",
  #     "updated_at" => "2016-09-15T22:00:59"},
  #   item_id: 1, item_type: "Post", originator_id: nil, originator: nil, meta: nil}
```

PaperTrail is inspired by the ruby gem ```paper_trail```. However, unlike the ```paper_trail``` gem this library actually results in less data duplication, faster and more explicit programming model to version your record changes.

The library source code is minimal and well tested. It is suggested to read the source code.

## Installation

  1. Add paper_trail to your list of dependencies in `mix.exs`:

  ```elixir
    def deps do
      [{:paper_trail, "~> 0.6.0"}]
    end
  ```

  2. configure paper_trail to use your application repo in `config/config.exs`:

  ```elixir
  config :paper_trail, repo: YourApplicationName.Repo
  # if you don't specify this PaperTrail will assume your repo name is Repo
  ```

  3. install and compile your dependency:

  ```mix deps.compile```

  4. run this command to generate the migration:

  ```mix papertrail.install```

  5. run the migration:

  ```mix ecto.migrate```

Your application is now ready to collect some history!

#### Does this work with phoenix?

YES! Make sure you do the steps above.

### %PaperTrail.Version{} fields:

| Column Name   | Type    | Description                | Entry Method             |
| ------------- | ------- | -------------------------- | ------------------------ |
| event         | String  | either insert, update or delete  | Library generates |
| item_type     | String  | model name of the reference record | Library generates |
| item_id       | Integer | model id of the reference record | Library generates |
| item_changes  | Map     | all the changes in this version as a map | Library generates |
| originator_id | Integer | foreign key reference to the creator/owner of this change | Optionally set |
| origin        | String  | short reference to origin(eg. worker:activity-checker, migration, admin:33) | Optionally set |
| meta          | Map     | any extra optional meta information about the version(eg. %{slug: "ausername"}) | Optionally set |
| inserted_at   | Date    | inserted_at timestamp       | Ecto generates |

### Version origin references:
PaperTrail records have a string field called ``origin```. ```PaperTrail.insert/1```, ```PaperTrail.update/1```, ```PaperTrail.delete/1``` functions accept a second argument to describe the origin of this version. Example:
```elixir
PaperTrail.update(changeset, origin: "migration")
# or:
PaperTrail.update(changeset, origin: "user:1234")
# or:
PaperTrail.delete(changeset, origin: "worker:delete_inactive_users")
```

### Originator relationships
You can specify setter/originator relationship to paper_trail versions with ```originator_id``` assignment during PaperTrail operations. This feature is only possible by specifying `:originator` keyword list for your application configuration:

```elixir
  # in your config/config.exs
  config :paper_trail, originator: [name: :user, model: YourApp.User]
  # For most applications originator should be the user since models can be updated/created/deleted by several users.
```

Then originator name could be used for querying and preloading however originator setting must be done via originator_id:

```elixir
user = create_user()
PaperTrail.insert(changeset, originator_id: user.id)
{:ok, result} = PaperTrail.update(edit_changeset, originator_id: user.id)
result[:version] |> Repo.preload(:user) |> Map.get(:user) # we can access the user who made the change from the version thanks to originator relationships!
PaperTrail.delete(edit_changeset, originator_id: user.id)
```

Also make sure you have the foreign-key constraint in the database and in your version migration file.

# Strict mode
This is a feature more suitable for larger applications. Models can keep their version references via foreign key constraints. Therefore it would be impossible to delete the first and current version of a model, it also makes querying easier and the whole design more relational database/SQL friendly. In order to enable strict mode:

```elixir
# in your config/config.exs
config :paper_trail, strict_mode: true
```

Strict mode expects tracked models to have foreign-key reference to their first_version and current_version. These columns must be named ```first_version_id```, and ```current_version_id``` in their respective model tables. A tracked model example with a migration file:

```elixir
# in the migration file: priv/repo/migrations/create_company.exs
defmodule Repo.Migrations.AddVersions do
  def change do
    create table(:companies) do
      add :name,       :string, null: false
      add :founded_in, :string

      # null constraint is optional to make model insertion impossible without a version:
      add :first_version_id, references(:versions), null: false
      add :current_version_id, references(:versions), null: false

      timestamps()
    end

    create index(:companies, [:first_version_id])
    create index(:companies, [:current_version_id])
  end
end

# in the model definition:
defmodule Company do
  use Ecto.Schema

  import Ecto.Changeset

  schema "companies" do
    field :name, :string
    field :founded_in, :string

    belongs_to :first_version, PaperTrail.Version
    belongs_to :current_version, PaperTrail.Version, on_replace: :update # on_replace: is important!

    timestamps()
  end

  def changeset(struct, params) do
    struct
    |> cast(params, [:name, :founded_in])
  end
end
```

When you run PaperTrail.insert/2 transaction, ```insert_version_id``` and ```current_version_id``` automagically gets assigned for the model. Example:

```elixir
company = Company.changeset(%Company{}, %{name: "Acme LLC"}) |> PaperTrail.insert
# {:ok,
#  %{model: %Company{__meta__: #Ecto.Schema.Metadata<:loaded, "companies">,
#     name: "Acme LLC", founded_in: nil, id: 1, inserted_at: #Ecto.DateTime<2016-09-15 21:42:38>,
#     updated_at: #Ecto.DateTime<2016-09-15 21:42:38>, insert_version_id: 1, current_version_id: 1},
#    version: %PaperTrail.Version{__meta__: #Ecto.Schema.Metadata<:loaded, "versions">,
#      event: "insert", id: 1, inserted_at: #Ecto.DateTime<2016-09-15 22:22:12>,
#      item_changes: %{name: "Acme LLC", founded_in: nil, id: 1, inserted_at: #Ecto.DateTime<2016-09-15 21:42:38>},
#      originator_id: nil, origin: "unknown", meta: nil}}}
```

When you PaperTrail.update/2 a model, ```current_version_id``` gets updated during the transaction!:

```elixir
edited_company = Company.changeset(company, %{name: "Acme Inc."}) |> PaperTrail.update(origin: "documentation")
# {:ok,
#  %{model: %Company{__meta__: #Ecto.Schema.Metadata<:loaded, "companies">,
#     name: "Acme Inc.", founded_in: nil, id: 1, inserted_at: #Ecto.DateTime<2016-09-15 21:42:38>,
#     updated_at: #Ecto.DateTime<2016-09-15 23:22:12>, insert_version_id: 1, current_version_id: 2},
#    version: %PaperTrail.Version{__meta__: #Ecto.Schema.Metadata<:loaded, "versions">,
#      event: "update", id: 2, inserted_at: #Ecto.DateTime<2016-09-15 23:22:12>,
#      item_changes: %{name: "Acme Inc."}, originator_id: nil, origin: "documentation", meta: nil}}}
```

If the version ```origin``` field isn't provided with a value, default ```origin``` will be "unknown". Origin column has a null constraint on strict_mode by design, you should put an ```origin``` reference to describe who makes the change. This is important for big applications because a model can change from many sources.

### Storing version meta data
You might want to add some meta data that doesn't belong to ``originator_id`` and ``origin`` fields. Such data could be stored in one object named ```meta``` in paper_trail versions. Meta field could be passed as the second optional parameter to PaperTrail.insert\\2, PaperTrail.update\\2, PaperTrail.delete\\2 functions:

```elixir
company = Company.changeset(%Company{}, %{name: "Acme Inc."})
  |> PaperTrail.insert(meta: %{slug: "acme-llc"})
# you can also combine this with an origin:
edited_company = Company.changeset(company, %{name: "Acme LLC"})
  |> PaperTrail.update(origin: "documentation", meta: %{slug: "acme-llc"})
# or even with an originator:
user = create_user()
deleted_company = Company.changeset(edited_company, %{})
  |> PaperTrail.delete(origin: "worker:github", originator: user.id, meta: %{slug: "acme-llc", important: true})
```

## Suggestions
- PaperTrail.Version(s) order matter,
- don't delete your paper_trail versions, instead you can merge them
- If you have a question or a problem, do not hesitate to create an issue or submit a pull request

## TODO:
** remove wrong Elixir compiler errors
