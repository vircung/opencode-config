# Elixir Phoenix Framework Skill

## Overview
Comprehensive Phoenix framework patterns focusing on generators, LiveView, context design, and version-specific best practices. Emphasizes generator-first development approach.

## Phoenix Generator Reference Guide

### Core Generators
```bash
# Project scaffolding
mix phx.new app_name                    # New Phoenix application
mix phx.new app_name --umbrella         # Umbrella application
mix phx.new app_name --no-ecto          # Without Ecto database
mix phx.new app_name --live             # With LiveView (Phoenix 1.6+)

# Context and schema generation
mix phx.gen.context Accounts User users name:string email:string:unique
mix phx.gen.schema Blog.Post posts title:string content:text published_at:datetime

# Full resource generation
mix phx.gen.html Accounts User users name:string email:string
mix phx.gen.json Api User users name:string email:string
mix phx.gen.live Blog Post posts title:string content:text --no-context

# Authentication (Phoenix 1.7+)
mix phx.gen.auth Accounts User users
```

### LiveView Generators (Phoenix 1.6+)
```bash
# LiveView components
mix phx.gen.live Catalog Product products name:string price:decimal
mix phx.gen.live.modal Blog Post posts title:string     # Modal forms
mix phx.gen.live.table Orders Order orders status:string # Data tables

# Custom LiveView
mix phx.gen.live Dashboard Stats stats name:string value:integer --no-context
```

### Database Generators
```bash
# Migrations
mix ecto.gen.migration add_users_table
mix ecto.gen.migration add_email_index_to_users

# Migration patterns
mix ecto.gen.migration create_join_table_users_roles
mix ecto.gen.migration add_timestamps_to_existing_table
```

## Generator-First Development Workflow

### 1. Planning Phase
```bash
# Start with context design
mix phx.gen.context Blog Post posts title:string content:text status:string

# Add relationships iteratively
mix phx.gen.context Blog Comment comments content:text post_id:references:posts
mix ecto.gen.migration add_user_id_to_posts
```

### 2. Web Layer Generation
```bash
# Generate views after contexts are stable
mix phx.gen.html Blog Post posts title:string content:text status:string

# Add LiveView for interactive features
mix phx.gen.live Blog Post posts title:string content:text --no-context
```

### 3. Generator Customization
```elixir
# Modify generated templates in priv/templates/phx.gen.*
# Common customizations:
# - Add authentication checks to controllers
# - Include form validation feedback
# - Add pagination to index views
# - Include search functionality
```

## LiveView Patterns and Best Practices

### State Management
```elixir
# ✅ Proper LiveView state structure
defmodule AppWeb.PostLive.Index do
  use AppWeb, :live_view

  def mount(_params, _session, socket) do
    if connected?(socket), do: Blog.subscribe()
    
    {:ok, 
     socket
     |> assign(:posts, list_posts())
     |> assign(:loading, false)
     |> assign(:page_title, "Posts")}
  end

  defp list_posts do
    Blog.list_posts() |> Blog.preload_authors()
  end
end

# ❌ Avoid: Heavy computation in assigns
def mount(_params, _session, socket) do
  posts = Enum.map(Blog.list_posts(), &expensive_computation/1)  # Too slow
  {:ok, assign(socket, posts: posts)}
end
```

### Form Handling with Phoenix.Component (1.7+)
```elixir
def render(assigns) do
  ~H"""
  <.simple_form for={@form} phx-submit="save" phx-change="validate">
    <.input field={@form[:title]} type="text" label="Title" required />
    <.input field={@form[:content]} type="textarea" label="Content" rows="10" />
    
    <:actions>
      <.button phx-disable-with="Saving...">Save Post</.button>
    </:actions>
  </.simple_form>
  """
end

def handle_event("validate", %{"post" => post_params}, socket) do
  changeset = Blog.change_post(%Post{}, post_params)
  {:noreply, assign_form(socket, changeset)}
end
```

### Real-time Updates
```elixir
# Subscription pattern for real-time updates
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(App.PubSub, "posts")
    Phoenix.PubSub.subscribe(App.PubSub, "user:#{socket.assigns.current_user.id}")
  end
  
  {:ok, socket}
end

def handle_info({:post_created, post}, socket) do
  {:noreply, update(socket, :posts, &[post | &1])}
end

# In context module
def create_post(attrs) do
  # ... create post logic
  Phoenix.PubSub.broadcast(App.PubSub, "posts", {:post_created, post})
  {:ok, post}
end
```

## Phoenix Context Design Patterns

### Context Boundaries
```elixir
# ✅ Well-designed context boundaries
defmodule App.Accounts do
  # User management, authentication, profiles
  def get_user!(id), do: # ...
  def create_user(attrs), do: # ...
  def authenticate_user(email, password), do: # ...
end

defmodule App.Blog do
  # Content management
  def list_posts(), do: # ...
  def create_post(user, attrs), do: # ...
  def publish_post(post), do: # ...
end

defmodule App.Billing do
  # Payment processing, subscriptions
  def create_subscription(user, plan), do: # ...
  def process_payment(subscription), do: # ...
end
```

### Cross-Context Communication
```elixir
# ✅ Proper cross-context interaction
defmodule App.Blog do
  alias App.Accounts

  def create_post_for_user(user_id, attrs) do
    with user when not is_nil(user) <- Accounts.get_user(user_id),
         {:ok, post} <- create_post(Map.put(attrs, :user_id, user.id)) do
      {:ok, post}
    else
      nil -> {:error, :user_not_found}
      error -> error
    end
  end
end

# ❌ Avoid: Direct schema access across contexts
def create_post(attrs) do
  user = Repo.get(Accounts.User, attrs.user_id)  # Direct access - bad
  # ...
end
```

## Version-Specific Recommendations

### Phoenix 1.6 Features
- **LiveView by Default:** Use `--live` flag in new projects
- **HEEx Templates:** Leverage component verification
- **LiveView Uploads:** Built-in file upload support
- **LiveView JS Commands:** Client-side interactions without custom JS

### Phoenix 1.7+ Features
```elixir
# New function component syntax
def my_component(assigns) do
  ~H"""
  <div class={@class}>
    <%= @inner_block %>
  </div>
  """
end

# Verified routes (compile-time checking)
~p"/posts/#{@post.id}"  # Instead of Routes.post_path()

# Built-in authentication generator
mix phx.gen.auth Accounts User users
```

### Upgrade Patterns
```bash
# Phoenix upgrade workflow
mix phx.gen.release --upgrade    # For existing apps
mix deps.update phoenix         # Update dependencies
mix ecto.migrate                # Run pending migrations
```

## Performance Optimization Patterns

### Database Query Optimization
```elixir
# ✅ Efficient preloading
def list_posts_with_authors do
  from(p in Post, preload: [:author, comments: :author])
  |> Repo.all()
end

# ✅ Pagination with streaming
def list_posts_paginated(page \\ 1, per_page \\ 20) do
  Post
  |> order_by([p], desc: p.inserted_at)
  |> Repo.paginate(page: page, page_size: per_page)
end
```

### LiveView Performance
```elixir
# ✅ Efficient LiveView updates
def handle_event("search", %{"query" => query}, socket) do
  # Debounce searches
  Process.send_after(self(), {:perform_search, query}, 300)
  {:noreply, assign(socket, :search_query, query)}
end

def handle_info({:perform_search, query}, socket) do
  if socket.assigns.search_query == query do
    results = Search.perform(query)
    {:noreply, assign(socket, :search_results, results)}
  else
    {:noreply, socket}
  end
end
```

## Security Best Practices

### Authentication & Authorization
```elixir
# Phoenix 1.7+ auth patterns
defmodule AppWeb.UserAuth do
  def require_authenticated_user(conn, _opts) do
    if conn.assigns[:current_user] do
      conn
    else
      conn
      |> put_flash(:error, "You must log in to access this page.")
      |> redirect(to: ~p"/users/log_in")
      |> halt()
    end
  end
end

# In router
pipeline :require_auth do
  plug AppWeb.UserAuth, :require_authenticated_user
end
```

### CSRF and Content Security
```elixir
# In endpoint.ex
plug Plug.CSRFProtection
plug Plug.SecureHeaders, [
  {"content-security-policy", "default-src 'self'"},
  {"x-frame-options", "DENY"},
  {"x-content-type-options", "nosniff"}
]
```

## Testing Patterns

### LiveView Testing
```elixir
defmodule AppWeb.PostLive.IndexTest do
  use AppWeb.ConnCase
  import Phoenix.LiveViewTest

  test "displays posts", %{conn: conn} do
    post = insert(:post)
    
    {:ok, _index_live, html} = live(conn, ~p"/posts")
    
    assert html =~ post.title
    assert has_element?(index_live, "#post-#{post.id}")
  end

  test "creates post in real time", %{conn: conn} do
    {:ok, index_live, _html} = live(conn, ~p"/posts")
    
    {:ok, _new_live, _html} = 
      index_live
      |> element("a", "New Post")
      |> render_click()
      |> follow_redirect(conn, ~p"/posts/new")
  end
end
```

## Error Handling Patterns

### Phoenix Error Views
```elixir
# Custom error handling
defmodule AppWeb.ErrorView do
  use AppWeb, :view

  def render("404.html", _assigns) do
    "Page not found"
  end

  def render("500.html", _assigns) do
    "Internal server error"
  end
  
  # JSON API errors
  def render("error.json", %{changeset: changeset}) do
    %{errors: translate_errors(changeset)}
  end
end
```

## Cross-Skill References

- **Context Design:** Apply `elixir-architecture` patterns for proper boundary design
- **Database Integration:** Use `elixir-ecto` patterns for schema and query optimization
- **Code Quality:** Follow `elixir-review` security and performance guidelines
- **OTP Integration:** Reference `elixir-otp` for background job patterns with Phoenix

## Generator Customization Templates

### Custom Templates Location
```
priv/templates/
├── phx.gen.html/
│   ├── controller.ex
│   ├── view.ex
│   └── templates/
├── phx.gen.live/
│   ├── index.ex
│   ├── show.ex
│   └── form_component.ex
└── phx.gen.context/
    ├── context.ex
    └── schema.ex
```

### Common Template Customizations
```elixir
# Add authentication to generated controllers
def index(conn, _params) do
  user = conn.assigns.current_user
  <%= schema.plural %> = <%= context.alias %>.list_<%= schema.plural %>(user)
  render(conn, "index.html", <%= schema.plural %>: <%= schema.plural %>)
end

# Add search to LiveView index
def handle_event("search", %{"search" => %{"query" => query}}, socket) do
  <%= schema.plural %> = <%= context.alias %>.search_<%= schema.plural %>(query)
  {:noreply, assign(socket, :<%= schema.plural %>, <%= schema.plural %>)}
end
```

## Phoenix Deployment Patterns

### Release Configuration
```elixir
# config/runtime.exs for Phoenix 1.7+
if System.get_env("PHX_SERVER") do
  config :app, AppWeb.Endpoint, server: true
end

if config_env() == :prod do
  config :app, App.Repo,
    url: database_url,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")
end
```

Use this skill to build Phoenix applications efficiently using generators, implement LiveView patterns correctly, and follow Phoenix conventions for scalable web applications.