---
name: elixir-ecto
description: Ecto schemas, changesets, queries, migrations, and database patterns for Elixir applications
license: MIT
compatibility: opencode
metadata:
  audience: developers
  language: elixir
  database: ecto
---

# Elixir Ecto Skill

## Schema Design

### Basic Schema Definition

**Schema Structure:**
```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset
  
  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
  
  schema "users" do
    field :email, :string
    field :name, :string
    field :age, :integer
    field :is_active, :boolean, default: true
    field :password, :string, virtual: true
    field :password_hash, :string
    field :profile_data, :map
    
    has_many :posts, MyApp.Blog.Post
    has_one :profile, MyApp.Accounts.Profile
    many_to_many :roles, MyApp.Accounts.Role, join_through: "user_roles"
    
    timestamps()
  end
  
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :age, :password])
    |> validate_required([:email, :name])
    |> validate_format(:email, ~r/^[^\s]+@[^\s]+$/, message: "must have the @ sign and no spaces")
    |> validate_length(:name, min: 2, max: 100)
    |> validate_number(:age, greater_than: 0, less_than: 150)
    |> unique_constraint(:email)
    |> put_password_hash()
  end
  
  defp put_password_hash(%{valid?: true, changes: %{password: password}} = changeset) do
    put_change(changeset, :password_hash, Argon2.hash_pwd_salt(password))
  end
  defp put_password_hash(changeset), do: changeset
end
```

### Virtual Fields and Computed Values

**Virtual Fields for Input Processing:**
```elixir
defmodule MyApp.Accounts.Profile do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "profiles" do
    field :first_name, :string
    field :last_name, :string
    field :full_name, :string, virtual: true
    field :birth_date, :date
    field :age, :integer, virtual: true
    
    belongs_to :user, MyApp.Accounts.User
    
    timestamps()
  end
  
  def changeset(profile, attrs) do
    profile
    |> cast(attrs, [:first_name, :last_name, :birth_date, :full_name])
    |> validate_required([:first_name, :last_name])
    |> put_full_name()
    |> put_age()
  end
  
  defp put_full_name(%{changes: %{first_name: first, last_name: last}} = changeset) do
    put_change(changeset, :full_name, "#{first} #{last}")
  end
  defp put_full_name(changeset), do: changeset
  
  defp put_age(%{changes: %{birth_date: birth_date}} = changeset) do
    age = Date.diff(Date.utc_today(), birth_date) |> div(365)
    put_change(changeset, :age, age)
  end
  defp put_age(changeset), do: changeset
end
```

### Embedded Schemas

**For JSON/Map Fields:**
```elixir
defmodule MyApp.Accounts.Address do
  use Ecto.Schema
  import Ecto.Changeset
  
  @primary_key false
  embedded_schema do
    field :street, :string
    field :city, :string
    field :state, :string
    field :zip_code, :string
    field :country, :string, default: "US"
  end
  
  def changeset(address, attrs) do
    address
    |> cast(attrs, [:street, :city, :state, :zip_code, :country])
    |> validate_required([:street, :city, :state, :zip_code])
    |> validate_length(:zip_code, is: 5)
  end
end

# Usage in parent schema
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "users" do
    field :name, :string
    embeds_one :address, MyApp.Accounts.Address
    embeds_many :previous_addresses, MyApp.Accounts.Address
    
    timestamps()
  end
  
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:name])
    |> cast_embed(:address)
    |> cast_embed(:previous_addresses)
  end
end
```

### Associations

**Association Patterns:**
```elixir
# One-to-many with dependent deletion
defmodule MyApp.Blog.Post do
  use Ecto.Schema
  
  schema "posts" do
    field :title, :string
    field :body, :text
    field :published_at, :utc_datetime
    
    belongs_to :author, MyApp.Accounts.User
    has_many :comments, MyApp.Blog.Comment, on_delete: :delete_all
    has_many :post_tags, MyApp.Blog.PostTag, on_delete: :delete_all
    has_many :tags, through: [:post_tags, :tag]
    
    timestamps()
  end
end

# Many-to-many with join schema
defmodule MyApp.Blog.PostTag do
  use Ecto.Schema
  
  schema "post_tags" do
    belongs_to :post, MyApp.Blog.Post
    belongs_to :tag, MyApp.Blog.Tag
    
    field :created_by_id, :id
    
    timestamps()
  end
end

# Polymorphic associations
defmodule MyApp.Comments.Comment do
  use Ecto.Schema
  
  schema "comments" do
    field :body, :text
    field :commentable_type, :string
    field :commentable_id, :id
    
    belongs_to :user, MyApp.Accounts.User
    
    timestamps()
  end
  
  def for_post(query \\ __MODULE__) do
    from c in query, where: c.commentable_type == "post"
  end
  
  def for_user(query \\ __MODULE__) do
    from c in query, where: c.commentable_type == "user"
  end
end
```

## Changesets & Validation

### Advanced Changeset Patterns

**Conditional Validations:**
```elixir
defmodule MyApp.Accounts.User do
  import Ecto.Changeset
  
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :role, :company_id, :freelance_rate])
    |> validate_required([:email, :role])
    |> validate_role_specific_fields()
    |> validate_business_rules()
  end
  
  defp validate_role_specific_fields(changeset) do
    case get_field(changeset, :role) do
      :employee -> 
        changeset
        |> validate_required([:company_id])
        |> validate_exclusion(:freelance_rate, [nil], message: "must be empty for employees")
      
      :freelancer -> 
        changeset
        |> validate_required([:freelance_rate])
        |> validate_number(:freelance_rate, greater_than: 0)
        |> validate_exclusion(:company_id, [nil], message: "must be empty for freelancers")
      
      _ -> changeset
    end
  end
  
  defp validate_business_rules(changeset) do
    changeset
    |> validate_email_domain()
    |> validate_unique_email_per_company()
  end
  
  defp validate_email_domain(changeset) do
    email = get_change(changeset, :email)
    
    if email && String.ends_with?(email, "@competitor.com") do
      add_error(changeset, :email, "competitor email addresses not allowed")
    else
      changeset
    end
  end
end
```

**Custom Validators:**
```elixir
defmodule MyApp.Validators do
  import Ecto.Changeset
  
  def validate_password_strength(changeset, field) do
    validate_change changeset, field, fn _, password ->
      cond do
        String.length(password) < 8 ->
          [{field, "must be at least 8 characters"}]
        
        not Regex.match?(~r/[A-Z]/, password) ->
          [{field, "must contain uppercase letter"}]
        
        not Regex.match?(~r/[a-z]/, password) ->
          [{field, "must contain lowercase letter"}]
        
        not Regex.match?(~r/[0-9]/, password) ->
          [{field, "must contain number"}]
        
        true -> []
      end
    end
  end
  
  def validate_phone_number(changeset, field) do
    validate_change changeset, field, fn _, phone ->
      # Remove all non-digits
      digits = Regex.replace(~r/\D/, phone, "")
      
      case String.length(digits) do
        10 -> []
        11 when String.starts_with?(digits, "1") -> []
        _ -> [{field, "must be a valid phone number"}]
      end
    end
  end
  
  def validate_future_date(changeset, field) do
    validate_change changeset, field, fn _, date ->
      if Date.compare(date, Date.utc_today()) == :gt do
        []
      else
        [{field, "must be in the future"}]
      end
    end
  end
end
```

### Nested Changesets

**Complex Form Handling:**
```elixir
defmodule MyApp.Orders.Order do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "orders" do
    field :total_amount, :decimal
    field :status, :string, default: "pending"
    
    belongs_to :customer, MyApp.Accounts.User
    has_many :line_items, MyApp.Orders.LineItem, on_delete: :delete_all
    
    timestamps()
  end
  
  def changeset(order, attrs) do
    order
    |> cast(attrs, [:customer_id])
    |> cast_assoc(:line_items, required: true)
    |> validate_required([:customer_id])
    |> calculate_total()
    |> validate_number(:total_amount, greater_than: 0)
  end
  
  defp calculate_total(changeset) do
    case get_change(changeset, :line_items) do
      nil -> changeset
      line_items_changesets ->
        total = 
          line_items_changesets
          |> Enum.map(&get_field(&1, :amount))
          |> Enum.sum()
        
        put_change(changeset, :total_amount, total)
    end
  end
end

defmodule MyApp.Orders.LineItem do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "line_items" do
    field :quantity, :integer
    field :unit_price, :decimal
    field :amount, :decimal
    
    belongs_to :order, MyApp.Orders.Order
    belongs_to :product, MyApp.Catalog.Product
    
    timestamps()
  end
  
  def changeset(line_item, attrs) do
    line_item
    |> cast(attrs, [:product_id, :quantity, :unit_price])
    |> validate_required([:product_id, :quantity, :unit_price])
    |> validate_number(:quantity, greater_than: 0)
    |> validate_number(:unit_price, greater_than: 0)
    |> calculate_amount()
  end
  
  defp calculate_amount(changeset) do
    case {get_field(changeset, :quantity), get_field(changeset, :unit_price)} do
      {qty, price} when is_integer(qty) and not is_nil(price) ->
        amount = Decimal.mult(price, qty)
        put_change(changeset, :amount, amount)
      
      _ -> changeset
    end
  end
end
```

## Querying Patterns

### Basic Query Composition

**Query Building:**
```elixir
defmodule MyApp.Blog do
  import Ecto.Query
  alias MyApp.Repo
  alias MyApp.Blog.Post
  
  def list_posts(opts \\ []) do
    Post
    |> filter_by_status(opts[:status])
    |> filter_by_author(opts[:author_id])
    |> search_by_title(opts[:search])
    |> order_by_date(opts[:order])
    |> paginate(opts[:page], opts[:per_page])
    |> Repo.all()
    |> preload_associations()
  end
  
  defp filter_by_status(query, nil), do: query
  defp filter_by_status(query, status) do
    from p in query, where: p.status == ^status
  end
  
  defp filter_by_author(query, nil), do: query
  defp filter_by_author(query, author_id) do
    from p in query, where: p.author_id == ^author_id
  end
  
  defp search_by_title(query, nil), do: query
  defp search_by_title(query, search_term) do
    like_pattern = "%#{search_term}%"
    from p in query, where: ilike(p.title, ^like_pattern)
  end
  
  defp order_by_date(query, :asc) do
    from p in query, order_by: [asc: p.inserted_at]
  end
  defp order_by_date(query, _) do
    from p in query, order_by: [desc: p.inserted_at]
  end
  
  defp paginate(query, nil, _), do: query
  defp paginate(query, page, per_page) do
    offset = (page - 1) * per_page
    from p in query, limit: ^per_page, offset: ^offset
  end
  
  defp preload_associations(posts) do
    Repo.preload(posts, [:author, :tags, comments: :user])
  end
end
```

### Advanced Query Patterns

**Subqueries and Aggregations:**
```elixir
defmodule MyApp.Analytics do
  import Ecto.Query
  alias MyApp.Repo
  
  def popular_posts_with_stats do
    comment_counts = from c in Comment,
                     group_by: c.post_id,
                     select: %{post_id: c.post_id, count: count(c.id)}
    
    from p in Post,
      left_join: cc in subquery(comment_counts), on: p.id == cc.post_id,
      select: %{
        id: p.id,
        title: p.title,
        comment_count: coalesce(cc.count, 0),
        view_count: p.view_count
      },
      where: p.status == "published",
      order_by: [desc: coalesce(cc.count, 0)]
  end
  
  def user_activity_summary(user_id) do
    posts_query = from p in Post,
                   where: p.author_id == ^user_id,
                   select: count(p.id)
    
    comments_query = from c in Comment,
                      where: c.user_id == ^user_id,
                      select: count(c.id)
    
    %{
      posts_count: Repo.one(posts_query),
      comments_count: Repo.one(comments_query),
      last_activity: last_activity(user_id)
    }
  end
  
  defp last_activity(user_id) do
    last_post = from p in Post,
                where: p.author_id == ^user_id,
                select: p.inserted_at,
                order_by: [desc: p.inserted_at],
                limit: 1
    
    last_comment = from c in Comment,
                   where: c.user_id == ^user_id,
                   select: c.inserted_at,
                   order_by: [desc: c.inserted_at],
                   limit: 1
    
    [Repo.one(last_post), Repo.one(last_comment)]
    |> Enum.reject(&is_nil/1)
    |> Enum.max(Date, fn -> nil end)
  end
end
```

### Preloading Strategies

**Efficient Data Loading:**
```elixir
defmodule MyApp.Blog do
  # Basic preload
  def get_post_with_author(id) do
    Post
    |> Repo.get!(id)
    |> Repo.preload(:author)
  end
  
  # Selective preload based on query
  def get_post_with_comments(id, include_author: include_author) do
    preloads = if include_author do
      [author: [], comments: :user]
    else
      [comments: :user]
    end
    
    Post
    |> Repo.get!(id)
    |> Repo.preload(preloads)
  end
  
  # Custom preload query
  def get_post_with_recent_comments(id) do
    recent_comments = from c in Comment,
                      where: c.inserted_at > ago(7, "day"),
                      order_by: [desc: c.inserted_at],
                      preload: :user
    
    Post
    |> Repo.get!(id)
    |> Repo.preload([author: [], comments: recent_comments])
  end
  
  # Prevent N+1 with join preload for aggregation
  def list_posts_with_comment_counts do
    from p in Post,
      left_join: c in assoc(p, :comments),
      group_by: p.id,
      preload: [author: []],
      select: %{post: p, comment_count: count(c.id)}
  end
end
```

### Dynamic Queries

**Runtime Query Building:**
```elixir
defmodule MyApp.Search do
  import Ecto.Query
  
  def build_search_query(base_query, filters) do
    Enum.reduce(filters, base_query, &apply_filter/2)
  end
  
  defp apply_filter({:status, status}, query) when status != "" do
    from q in query, where: q.status == ^status
  end
  
  defp apply_filter({:date_range, %{start: start_date, end: end_date}}, query) do
    from q in query,
      where: q.inserted_at >= ^start_date and q.inserted_at <= ^end_date
  end
  
  defp apply_filter({:tags, tag_ids}, query) when is_list(tag_ids) and tag_ids != [] do
    from q in query,
      join: t in assoc(q, :tags),
      where: t.id in ^tag_ids,
      distinct: true
  end
  
  defp apply_filter({:search, term}, query) when is_binary(term) and term != "" do
    like_term = "%#{term}%"
    from q in query,
      where: ilike(q.title, ^like_term) or ilike(q.body, ^like_term)
  end
  
  defp apply_filter({:author_name, name}, query) when is_binary(name) and name != "" do
    from q in query,
      join: a in assoc(q, :author),
      where: ilike(a.name, ^"%#{name}%")
  end
  
  # Ignore unknown or empty filters
  defp apply_filter(_, query), do: query
end
```

## Migrations & Schema Evolution

### Migration Patterns

**Safe Migration Strategies:**
```elixir
defmodule MyApp.Repo.Migrations.CreateUsersTable do
  use Ecto.Migration
  
  def change do
    create table(:users, primary_key: false) do
      add :id, :binary_id, primary_key: true
      add :email, :string, null: false
      add :name, :string, null: false
      add :age, :integer
      add :is_active, :boolean, default: true, null: false
      
      timestamps()
    end
    
    create unique_index(:users, [:email])
    create index(:users, [:is_active])
    
    # Add constraint for data integrity
    create constraint(:users, :age_must_be_positive, check: "age > 0")
  end
end

# Adding columns safely
defmodule MyApp.Repo.Migrations.AddUserProfile do
  use Ecto.Migration
  
  def change do
    alter table(:users) do
      add :profile_data, :map, default: "{}"
      add :last_login_at, :utc_datetime
      add :login_count, :integer, default: 0
    end
    
    create index(:users, [:last_login_at])
  end
end

# Data migration
defmodule MyApp.Repo.Migrations.MigrateUserNames do
  use Ecto.Migration
  import Ecto.Query
  
  def up do
    # First add new columns
    alter table(:users) do
      add :first_name, :string
      add :last_name, :string
    end
    
    flush()
    
    # Migrate data
    from(u in "users", select: [:id, :name])
    |> MyApp.Repo.all()
    |> Enum.each(fn user ->
      names = String.split(user.name, " ", parts: 2)
      first_name = Enum.at(names, 0)
      last_name = Enum.at(names, 1, "")
      
      from(u in "users", where: u.id == ^user.id)
      |> MyApp.Repo.update_all(set: [first_name: first_name, last_name: last_name])
    end)
  end
  
  def down do
    alter table(:users) do
      remove :first_name
      remove :last_name
    end
  end
end
```

### Zero-Downtime Migrations

**Safe Schema Changes:**
```elixir
# Step 1: Add new nullable column
defmodule MyApp.Repo.Migrations.AddNewEmailColumn do
  use Ecto.Migration
  
  def change do
    alter table(:users) do
      add :new_email, :string
    end
    
    create index(:users, [:new_email])
  end
end

# Step 2: Deploy code that writes to both columns
# Step 3: Backfill data
defmodule MyApp.Repo.Migrations.BackfillNewEmail do
  use Ecto.Migration
  import Ecto.Query
  
  def up do
    from(u in "users", where: is_nil(u.new_email))
    |> MyApp.Repo.update_all(set: [new_email: fragment("email")])
  end
  
  def down, do: :ok
end

# Step 4: Add not null constraint
defmodule MyApp.Repo.Migrations.MakeNewEmailRequired do
  use Ecto.Migration
  
  def change do
    alter table(:users) do
      modify :new_email, :string, null: false
    end
    
    create unique_index(:users, [:new_email])
  end
end

# Step 5: Deploy code that only uses new column
# Step 6: Remove old column
defmodule MyApp.Repo.Migrations.RemoveOldEmailColumn do
  use Ecto.Migration
  
  def change do
    drop index(:users, [:email])
    
    alter table(:users) do
      remove :email
    end
    
    rename table(:users), :new_email, to: :email
  end
end
```

## Performance & Optimization

### Query Optimization

**N+1 Query Prevention:**
```elixir
# Bad: N+1 queries
def list_posts_bad do
  posts = Repo.all(Post)
  
  Enum.map(posts, fn post ->
    # This generates one query per post!
    author = Repo.get(User, post.author_id)
    %{post | author: author}
  end)
end

# Good: Use preload
def list_posts_good do
  Post
  |> Repo.all()
  |> Repo.preload(:author)
end

# Good: Use join when you don't need associated data
def list_post_summaries do
  from p in Post,
    join: u in User, on: p.author_id == u.id,
    select: %{id: p.id, title: p.title, author_name: u.name}
end
```

**Batch Loading:**
```elixir
defmodule MyApp.Accounts do
  def get_users_with_post_counts(user_ids) do
    # Get users
    users = from(u in User, where: u.id in ^user_ids) |> Repo.all()
    
    # Get post counts in single query
    post_counts = 
      from(p in Post,
        where: p.author_id in ^user_ids,
        group_by: p.author_id,
        select: {p.author_id, count(p.id)})
      |> Repo.all()
      |> Map.new()
    
    # Combine results
    Enum.map(users, fn user ->
      post_count = Map.get(post_counts, user.id, 0)
      %{user | post_count: post_count}
    end)
  end
end
```

**Database Indexes:**
```elixir
defmodule MyApp.Repo.Migrations.AddPerformanceIndexes do
  use Ecto.Migration
  
  def change do
    # Composite index for common query pattern
    create index(:posts, [:author_id, :status, :inserted_at])
    
    # Partial index for active records only
    create index(:users, [:email], where: "is_active = true")
    
    # Functional index for case-insensitive searches
    create index(:users, ["lower(email)"])
    
    # GIN index for JSON column searches
    create index(:posts, [:metadata], using: "GIN")
  end
end
```

### Connection Pool Management

**Pool Configuration:**
```elixir
# config/config.exs
config :my_app, MyApp.Repo,
  pool_size: String.to_integer(System.get_env("DB_POOL_SIZE") || "10"),
  queue_target: 5000,
  queue_interval: 5000,
  timeout: 15_000,
  pool_timeout: 5_000

# For read replicas
config :my_app, MyApp.ReadRepo,
  pool_size: 5,
  priv: "priv/repo"  # Share migration directory
```

**Transaction Management:**
```elixir
defmodule MyApp.Orders do
  alias MyApp.Repo
  alias Ecto.Multi
  
  def create_order_with_payment(order_attrs, payment_attrs) do
    Multi.new()
    |> Multi.insert(:order, Order.changeset(%Order{}, order_attrs))
    |> Multi.run(:payment, fn repo, %{order: order} ->
      payment_attrs = Map.put(payment_attrs, :order_id, order.id)
      Payment.changeset(%Payment{}, payment_attrs)
      |> repo.insert()
    end)
    |> Multi.run(:inventory, fn _repo, %{order: order} ->
      # Complex inventory update logic
      MyApp.Inventory.reserve_items(order.line_items)
    end)
    |> Repo.transaction()
    |> case do
      {:ok, %{order: order, payment: payment}} -> {:ok, order}
      {:error, :order, changeset, _} -> {:error, changeset}
      {:error, :payment, changeset, _} -> {:error, changeset}
      {:error, :inventory, reason, _} -> {:error, reason}
    end
  end
end
```

## Cross-References

### Phoenix Integration
For Phoenix-specific Ecto patterns, schema generation with generators, and web-layer database integration,
see the `elixir-phoenix-framework` skill.

### System Architecture
For designing data layer architecture, context boundaries with Ecto schemas, and database scaling patterns,
see the `elixir-architecture` skill.

### Performance Review
For database query optimization, N+1 detection, and Ecto performance monitoring,
see the `elixir-review` skill.

## Best Practices Summary

### Schema Design
1. **Use UUIDs for distributed systems** - avoid integer ID conflicts
2. **Validate at the database level** - use constraints and indexes
3. **Keep schemas focused** - single responsibility principle
4. **Use virtual fields judiciously** - for computed values and form handling

### Changeset Patterns
1. **Validate early and often** - fail fast with clear messages
2. **Separate update types** - different changesets for different operations
3. **Use custom validators** - for complex business rules
4. **Handle associations properly** - use cast_assoc and cast_embed

### Query Optimization
1. **Prevent N+1 queries** - use preload and joins appropriately
2. **Use indexes strategically** - for common query patterns
3. **Batch operations** - reduce round trips to database
4. **Profile queries** - measure before optimizing

### Migration Safety
1. **Add before remove** - maintain compatibility during deployment
2. **Use constraints** - enforce data integrity
3. **Test rollbacks** - ensure migrations are reversible
4. **Batch large changes** - avoid blocking operations

This skill provides comprehensive guidance for building efficient, maintainable database layers with Ecto in Elixir applications.