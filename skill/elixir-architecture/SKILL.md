---
name: elixir-architecture
description: Elixir system design patterns, context boundaries, and scalable architecture for distributed applications
license: MIT
compatibility: opencode
metadata:
  audience: architects
  language: elixir
  focus: system-design
---

# Elixir Architecture Skill

## Context-Driven Design

### Phoenix Context Boundaries

**What are Contexts?**
Contexts are dedicated modules that expose and group related functionality. They represent bounded contexts in your domain and serve as the public API for your business logic.

```elixir
# Good context organization
defmodule MyApp.Accounts do
  @moduledoc "The Accounts context - manages users, authentication"
  
  def list_users, do: Repo.all(User)
  def get_user!(id), do: Repo.get!(User, id)
  def create_user(attrs), do: %User{} |> User.changeset(attrs) |> Repo.insert()
  def update_user(%User{} = user, attrs), do: user |> User.changeset(attrs) |> Repo.update()
  def delete_user(%User{} = user), do: Repo.delete(user)
end

defmodule MyApp.Blog do
  @moduledoc "The Blog context - manages posts, comments"
  
  alias MyApp.Accounts
  
  def list_posts, do: Repo.all(Post) |> Repo.preload(:author)
  def create_post(attrs, %Accounts.User{} = author) do
    %Post{author_id: author.id}
    |> Post.changeset(attrs)
    |> Repo.insert()
  end
end
```

### Context Design Principles

**1. Single Responsibility**
Each context should handle one business domain:
```elixir
# Good - focused contexts
MyApp.Accounts    # Users, authentication, profiles
MyApp.Blog        # Posts, comments, categories  
MyApp.Catalog     # Products, inventory
MyApp.Sales       # Orders, payments, invoices

# Bad - mixed concerns
MyApp.Admin       # Too vague, what does it handle?
MyApp.Data        # Not domain-focused
```

**2. Context Boundaries**
```elixir
# Contexts should not directly access other contexts' schemas
# Bad
defmodule MyApp.Blog do
  def list_posts_by_user(user_id) do
    # DON'T directly use Accounts.User schema
    MyApp.Repo.all(from p in Post, join: u in MyApp.Accounts.User, where: p.user_id == ^user_id)
  end
end

# Good  
defmodule MyApp.Blog do
  def list_posts_by_user(user_id) do
    # Use the public Accounts context API
    with {:ok, user} <- MyApp.Accounts.get_user(user_id) do
      Repo.all(from p in Post, where: p.user_id == ^user.id)
    end
  end
end
```

**3. Context APIs Should Hide Implementation**
```elixir
# Good - clean public API
defmodule MyApp.Accounts do
  def authenticate_user(email, password) do
    with user when not is_nil(user) <- get_user_by_email(email),
         true <- verify_password(password, user.password_hash) do
      {:ok, user}
    else
      _ -> {:error, :invalid_credentials}
    end
  end
  
  # Private implementation details
  defp get_user_by_email(email), do: Repo.get_by(User, email: email)
  defp verify_password(password, hash), do: Argon2.verify_pass(password, hash)
end
```

### When to Split Contexts

**Split when:**
- Context grows beyond 10-15 public functions
- Mixed business domains in one context
- Different teams responsible for different functionality
- Different data access patterns or scaling needs

**Example Split:**
```elixir
# Before: Large Accounts context
defmodule MyApp.Accounts do
  # User management
  def create_user(attrs), do: ...
  def list_users(), do: ...
  
  # Authentication  
  def authenticate_user(email, password), do: ...
  def generate_session_token(user), do: ...
  
  # Profile management
  def update_profile(user, attrs), do: ...
  def upload_avatar(user, file), do: ...
  
  # User preferences
  def get_preferences(user), do: ...
  def update_preferences(user, prefs), do: ...
end

# After: Split into focused contexts
defmodule MyApp.Accounts do
  # Core user management only
  def create_user(attrs), do: ...
  def list_users(), do: ...
end

defmodule MyApp.Auth do
  # Authentication concerns
  def authenticate_user(email, password), do: ...
  def generate_session_token(user), do: ...
end

defmodule MyApp.Profiles do  
  # Profile and preference management
  def update_profile(user, attrs), do: ...
  def get_preferences(user), do: ...
end
```

## System Architecture Patterns

### Application Structure

**Standard Phoenix Application:**
```
my_app/
├── lib/
│   ├── my_app/           # Business logic contexts
│   │   ├── accounts/     # Account-related schemas
│   │   ├── blog/         # Blog-related schemas
│   │   ├── accounts.ex   # Accounts context
│   │   └── blog.ex       # Blog context
│   ├── my_app_web/       # Web interface
│   │   ├── controllers/
│   │   ├── live/         # LiveView modules
│   │   ├── templates/
│   │   └── router.ex
│   └── my_app.ex         # Application module
└── test/
```

**Umbrella Application (for larger systems):**
```
my_umbrella/
├── apps/
│   ├── my_app_core/      # Business logic
│   ├── my_app_web/       # Web interface  
│   ├── my_app_admin/     # Admin interface
│   └── my_app_api/       # API interface
└── config/
```

### When to Use Umbrella Apps

**Use Umbrella when:**
- Multiple applications with shared business logic
- Different deployment requirements (web vs API vs admin)
- Team boundaries align with application boundaries
- Different scaling characteristics

**Example Umbrella Organization:**
```elixir
# apps/my_app_core/lib/my_app_core.ex
defmodule MyAppCore.Accounts do
  # Shared business logic
end

# apps/my_app_web/lib/my_app_web.ex  
defmodule MyAppWeb.UserController do
  def index(conn, _params) do
    users = MyAppCore.Accounts.list_users()
    render(conn, "index.html", users: users)
  end
end

# apps/my_app_api/lib/my_app_api.ex
defmodule MyAppApi.UserController do
  def index(conn, _params) do
    users = MyAppCore.Accounts.list_users()
    json(conn, %{data: users})
  end
end
```

## Scalability & Distributed Systems

### Horizontal Scaling Strategies

**1. Stateless Application Design**
```elixir
# Good - stateless operations
defmodule MyApp.Orders do
  def calculate_total(order_items) do
    order_items
    |> Enum.map(&(&1.price * &1.quantity))
    |> Enum.sum()
  end
end

# Problematic - relies on local state
defmodule MyApp.Cache do
  use Agent
  
  def start_link(_), do: Agent.start_link(fn -> %{} end, name: __MODULE__)
  def get(key), do: Agent.get(__MODULE__, &Map.get(&1, key))
  # This won't work across multiple nodes
end
```

**2. Database as Source of Truth**
```elixir
# Good pattern - database-driven
defmodule MyApp.Inventory do
  def reserve_item(item_id, quantity) do
    Multi.new()
    |> Multi.run(:check_stock, fn repo, _ ->
      item = repo.get!(Item, item_id)
      if item.quantity >= quantity do
        {:ok, item}
      else
        {:error, :insufficient_stock}
      end
    end)
    |> Multi.update(:update_item, fn %{check_stock: item} ->
      Item.changeset(item, %{quantity: item.quantity - quantity})
    end)
    |> Repo.transaction()
  end
end
```

### Node Clustering and Distribution

**Basic Cluster Setup:**
```elixir
# config/runtime.exs
if config_env() == :prod do
  config :my_app, MyApp.Repo,
    # Database config...
  
  config :libcluster,
    topologies: [
      k8s: [
        strategy: Cluster.Strategy.Kubernetes,
        config: [
          mode: :dns,
          kubernetes_node_basename: "my_app",
          kubernetes_selector: "app=my_app"
        ]
      ]
    ]
end

# lib/my_app/application.ex
def start(_type, _args) do
  children = [
    MyApp.Repo,
    {Cluster.Supervisor, [Application.get_env(:libcluster, :topologies), [name: MyApp.ClusterSupervisor]]},
    MyAppWeb.Endpoint
  ]
  
  Supervisor.start_link(children, strategy: :one_for_one)
end
```

### PubSub for Decoupling

**Cross-Context Communication:**
```elixir
# Instead of direct context calls, use PubSub
defmodule MyApp.Orders do
  def complete_order(order) do
    with {:ok, order} <- update_order_status(order, :completed) do
      # Notify other contexts without direct coupling
      Phoenix.PubSub.broadcast(MyApp.PubSub, "orders", {:order_completed, order})
      {:ok, order}
    end
  end
end

defmodule MyApp.Notifications do
  use GenServer
  
  def start_link(_) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end
  
  def init(state) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "orders")
    {:ok, state}
  end
  
  def handle_info({:order_completed, order}, state) do
    send_completion_email(order)
    {:noreply, state}
  end
end
```

### Event-Driven Architecture

**Event Sourcing Pattern:**
```elixir
defmodule MyApp.Events do
  defstruct [:id, :type, :data, :timestamp, :version]
  
  def create_event(type, data) do
    %__MODULE__{
      id: Ecto.UUID.generate(),
      type: type,
      data: data,
      timestamp: DateTime.utc_now(),
      version: 1
    }
  end
end

defmodule MyApp.EventStore do
  def append_event(stream_id, event) do
    # Store event in database
    %EventRecord{}
    |> EventRecord.changeset(%{
      stream_id: stream_id,
      event_type: event.type,
      event_data: event.data,
      version: event.version
    })
    |> Repo.insert()
  end
  
  def get_events(stream_id) do
    EventRecord
    |> where(stream_id: ^stream_id)
    |> order_by(:version)
    |> Repo.all()
  end
end
```

## API Design Patterns

### Internal Module APIs

**Good API Design:**
```elixir
defmodule MyApp.Accounts do
  @type user_attrs :: %{email: String.t(), name: String.t()}
  @type user :: %MyApp.Accounts.User{}
  
  @spec create_user(user_attrs()) :: {:ok, user()} | {:error, Ecto.Changeset.t()}
  def create_user(attrs) do
    %User{}
    |> User.changeset(attrs)
    |> Repo.insert()
  end
  
  @spec get_user(integer()) :: {:ok, user()} | {:error, :not_found}
  def get_user(id) when is_integer(id) do
    case Repo.get(User, id) do
      nil -> {:error, :not_found}
      user -> {:ok, user}
    end
  end
end
```

### Error Handling Patterns

**Consistent Error Returns:**
```elixir
# Good - consistent {:ok, result} | {:error, reason} pattern
defmodule MyApp.Payments do
  def process_payment(amount, card_token) do
    with {:ok, card} <- validate_card(card_token),
         {:ok, charge} <- charge_card(card, amount),
         {:ok, payment} <- save_payment(charge) do
      {:ok, payment}
    else
      {:error, :invalid_card} -> {:error, :payment_failed}
      {:error, :insufficient_funds} -> {:error, :payment_declined}
      {:error, reason} -> {:error, reason}
    end
  end
end
```

## Cross-References

### OTP Patterns
For OTP design patterns (GenServer, Supervisor, process architecture), 
see the `elixir-otp` skill.

### Phoenix Context Design
For Phoenix-specific context patterns, generator usage, and web-layer integration,
see the `elixir-phoenix-framework` skill.

### Data Layer Architecture
For Ecto schema design, database patterns, and query optimization within contexts,
see the `elixir-ecto` skill.

## Decision Framework

### Context Design Decisions
1. **Start with broad contexts** - split when they become unwieldy
2. **Follow domain boundaries** - not technical boundaries
3. **Contexts should not know about web layer** - keep them pure business logic
4. **Use context APIs** - don't access schemas directly from other contexts

### Scaling Decisions
1. **Measure before optimizing** - profile actual bottlenecks
2. **Start simple** - single node deployment first
3. **Scale horizontally** - add nodes before vertical scaling
4. **Database is the bottleneck** - optimize queries before adding complexity

### Architecture Evolution
1. **Monolith first** - extract services when team/domain boundaries are clear
2. **Umbrella apps** - for logical separation within monolith
3. **Microservices** - only when organizational scale demands it

This architectural foundation provides the structure for building maintainable, scalable Elixir applications that can grow with your needs.