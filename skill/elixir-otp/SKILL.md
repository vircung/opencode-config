---
name: elixir-otp
description: OTP design patterns, supervision trees, and process management with KISS principle emphasis
license: MIT
compatibility: opencode
metadata:
  audience: developers
  language: elixir
  focus: otp-processes
---

# Elixir OTP Skill

## Essential OTP Patterns (KISS First)

Use these patterns for 80% of use cases. Start simple, optimize later.

### GenServer: Stateful Processes

**When to use GenServer:**
- Need to maintain state across function calls
- State needs to be shared across processes
- Need to serialize access to a resource
- Long-running processes with periodic tasks

**Basic GenServer Pattern:**
```elixir
defmodule MyApp.Counter do
  use GenServer
  
  # Client API (public interface)
  def start_link(initial_value \\ 0) do
    GenServer.start_link(__MODULE__, initial_value, name: __MODULE__)
  end
  
  def get_count do
    GenServer.call(__MODULE__, :get_count)
  end
  
  def increment do
    GenServer.cast(__MODULE__, :increment)
  end
  
  def increment_by(amount) do
    GenServer.call(__MODULE__, {:increment_by, amount})
  end
  
  # Server Callbacks (private implementation)
  @impl true
  def init(initial_value) do
    {:ok, initial_value}
  end
  
  @impl true
  def handle_call(:get_count, _from, count) do
    {:reply, count, count}
  end
  
  @impl true
  def handle_call({:increment_by, amount}, _from, count) do
    new_count = count + amount
    {:reply, new_count, new_count}
  end
  
  @impl true
  def handle_cast(:increment, count) do
    {:noreply, count + 1}
  end
end
```

**Common GenServer Patterns:**

**1. Resource Pool:**
```elixir
defmodule MyApp.ConnectionPool do
  use GenServer
  
  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  def checkout do
    GenServer.call(__MODULE__, :checkout)
  end
  
  def checkin(connection) do
    GenServer.cast(__MODULE__, {:checkin, connection})
  end
  
  @impl true
  def init(opts) do
    pool_size = Keyword.get(opts, :pool_size, 10)
    connections = Enum.map(1..pool_size, fn _ -> create_connection() end)
    {:ok, %{available: connections, in_use: []}}
  end
  
  @impl true
  def handle_call(:checkout, from, %{available: [conn | rest]} = state) do
    monitor_ref = Process.monitor(from)
    new_state = %{
      state | 
      available: rest, 
      in_use: [{conn, from, monitor_ref} | state.in_use]
    }
    {:reply, {:ok, conn}, new_state}
  end
  
  def handle_call(:checkout, _from, %{available: []} = state) do
    {:reply, {:error, :no_connections}, state}
  end
end
```

**2. Cache with TTL:**
```elixir
defmodule MyApp.Cache do
  use GenServer
  
  def start_link(_opts) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end
  
  def put(key, value, ttl \\ 60_000) do
    GenServer.cast(__MODULE__, {:put, key, value, ttl})
  end
  
  def get(key) do
    GenServer.call(__MODULE__, {:get, key})
  end
  
  @impl true
  def init(_) do
    # Schedule cleanup every minute
    :timer.send_interval(60_000, :cleanup)
    {:ok, %{}}
  end
  
  @impl true
  def handle_cast({:put, key, value, ttl}, cache) do
    expires_at = System.system_time(:millisecond) + ttl
    {:noreply, Map.put(cache, key, {value, expires_at})}
  end
  
  @impl true
  def handle_call({:get, key}, _from, cache) do
    now = System.system_time(:millisecond)
    case Map.get(cache, key) do
      {value, expires_at} when expires_at > now -> {:reply, {:ok, value}, cache}
      _ -> {:reply, :not_found, cache}
    end
  end
  
  @impl true
  def handle_info(:cleanup, cache) do
    now = System.system_time(:millisecond)
    clean_cache = cache |> Enum.reject(fn {_key, {_value, expires_at}} -> expires_at <= now end) |> Map.new()
    {:noreply, clean_cache}
  end
end
```

### Supervisor: Process Supervision

**When to use Supervisor:**
- ALWAYS supervise your processes (GenServers, Tasks)
- Need fault tolerance and automatic restart
- Managing a group of related processes

**Basic Supervision Strategies:**

**1. `:one_for_one` (Most Common - 90% of cases)**
```elixir
defmodule MyApp.Supervisor do
  use Supervisor
  
  def start_link(opts) do
    Supervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end
  
  @impl true
  def init(_opts) do
    children = [
      MyApp.Counter,
      MyApp.Cache,
      {MyApp.Worker, [arg1: "value"]}
    ]
    
    # If one child crashes, restart only that child
    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

**2. `:rest_for_one` (When Order Matters)**
```elixir
defmodule MyApp.DatabaseSupervisor do
  use Supervisor
  
  @impl true
  def init(_opts) do
    children = [
      MyApp.Repo,           # Database connection
      MyApp.Cache,          # Depends on DB
      MyApp.DataProcessor   # Depends on both
    ]
    
    # If DataProcessor crashes, restart it only
    # If Cache crashes, restart Cache AND DataProcessor
    # If Repo crashes, restart ALL children
    Supervisor.init(children, strategy: :rest_for_one)
  end
end
```

**3. `:one_for_all` (All or Nothing)**
```elixir
defmodule MyApp.ClusterSupervisor do
  use Supervisor
  
  @impl true  
  def init(_opts) do
    children = [
      MyApp.ClusterNode1,
      MyApp.ClusterNode2,
      MyApp.ClusterCoordinator
    ]
    
    # If ANY child crashes, restart ALL children
    # Use when children are tightly coupled
    Supervisor.init(children, strategy: :one_for_all)
  end
end
```

### Task: One-off Async Work

**When to use Task:**
- One-off async operations
- Fire-and-forget work
- Parallel processing without state

**Basic Task Patterns:**

**1. Simple Async/Await:**
```elixir
# Fire and forget
Task.start(fn -> 
  send_email(user.email, "Welcome!")
end)

# Async with result
task = Task.async(fn ->
  expensive_calculation(data)
end)

result = Task.await(task, 10_000)  # 10 second timeout
```

**2. Supervised Tasks:**
```elixir
defmodule MyApp.TaskSupervisor do
  use Task.Supervisor
  
  def start_link(_opts) do
    Task.Supervisor.start_link(name: __MODULE__)
  end
end

# In your application supervisor
children = [
  MyApp.TaskSupervisor,
  # other children...
]

# Use supervised tasks
Task.Supervisor.start_child(MyApp.TaskSupervisor, fn ->
  process_large_file(file_path)
end)
```

**3. Parallel Processing:**
```elixir
defmodule MyApp.BatchProcessor do
  def process_batch(items) do
    items
    |> Task.async_stream(&process_item/1, max_concurrency: 10, timeout: 30_000)
    |> Enum.to_list()
  end
  
  defp process_item(item) do
    # Process individual item
    {:ok, processed_item}
  end
end
```

### Agent: Simple Shared State

**When to use Agent:**
- Simple shared state (get/update operations)
- No complex logic or async operations
- No need for custom message handling

**Agent vs GenServer Decision:**
- **Use Agent**: Simple key-value store, counters, flags
- **Use GenServer**: Complex state logic, async operations, custom messages

```elixir
defmodule MyApp.Settings do
  use Agent
  
  def start_link(initial_settings \\ %{}) do
    Agent.start_link(fn -> initial_settings end, name: __MODULE__)
  end
  
  def get(key) do
    Agent.get(__MODULE__, &Map.get(&1, key))
  end
  
  def put(key, value) do
    Agent.update(__MODULE__, &Map.put(&1, key, value))
  end
  
  def get_all do
    Agent.get(__MODULE__, & &1)
  end
end
```

## Process Communication

### Message Passing Patterns

**1. Direct Send/Receive:**
```elixir
defmodule MyApp.DirectMessaging do
  def start_receiver do
    spawn(fn -> message_loop() end)
  end
  
  defp message_loop do
    receive do
      {:hello, sender_pid} -> 
        send(sender_pid, {:reply, "Hello back!"})
        message_loop()
      :stop -> 
        :ok
      _ -> 
        message_loop()
    end
  end
  
  def send_message(receiver_pid, message) do
    send(receiver_pid, message)
    
    receive do
      {:reply, response} -> response
    after
      5000 -> :timeout
    end
  end
end
```

**2. GenServer Calls (Preferred):**
```elixir
# Synchronous communication
result = GenServer.call(MyWorker, {:process, data})

# Asynchronous communication  
GenServer.cast(MyWorker, {:update_state, new_data})
```

### Process Monitoring and Linking

**1. Process Monitoring:**
```elixir
defmodule MyApp.ProcessMonitor do
  use GenServer
  
  def start_link(_opts) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end
  
  def monitor_process(pid) do
    GenServer.cast(__MODULE__, {:monitor, pid})
  end
  
  @impl true
  def handle_cast({:monitor, pid}, monitored) do
    ref = Process.monitor(pid)
    {:noreply, Map.put(monitored, ref, pid)}
  end
  
  @impl true
  def handle_info({:DOWN, ref, :process, pid, reason}, monitored) do
    IO.puts("Process #{inspect(pid)} died with reason: #{inspect(reason)}")
    {:noreply, Map.delete(monitored, ref)}
  end
end
```

**2. Process Linking (Use with Caution):**
```elixir
# Bidirectional link - if one dies, both die
Process.link(other_pid)

# Better: Use supervision for managed restarts
```

## Advanced OTP Patterns (Use When Needed)

These patterns solve specific scaling problems. Don't use unless you have measured the need.

### DynamicSupervisor: Runtime Child Spawning

**When you outgrow static supervision:**
- Need to spawn processes at runtime
- Don't know number of processes at startup
- Managing user sessions, connections

```elixir
defmodule MyApp.SessionSupervisor do
  use DynamicSupervisor
  
  def start_link(_opts) do
    DynamicSupervisor.start_link(__MODULE__, [], name: __MODULE__)
  end
  
  @impl true
  def init(_opts) do
    DynamicSupervisor.init(strategy: :one_for_one)
  end
  
  def start_session(user_id) do
    spec = {MyApp.UserSession, user_id}
    DynamicSupervisor.start_child(__MODULE__, spec)
  end
  
  def stop_session(user_id) do
    case Registry.lookup(MyApp.SessionRegistry, user_id) do
      [{pid, _}] -> DynamicSupervisor.terminate_child(__MODULE__, pid)
      [] -> :ok
    end
  end
end
```

### PartitionSupervisor: Reducing GenServer Bottlenecks

**When single GenServer becomes bottleneck:**
- High-frequency operations
- Contention on single process
- Need to distribute load

```elixir
defmodule MyApp.CounterSupervisor do
  use PartitionSupervisor
  
  def start_link(_opts) do
    PartitionSupervisor.start_link(__MODULE__, [], name: __MODULE__)
  end
  
  @impl true
  def init(_opts) do
    PartitionSupervisor.init(MyApp.Counter, strategy: :one_for_one)
  end
  
  def increment(key) do
    PartitionSupervisor.get_child_pid(__MODULE__, key)
    |> MyApp.Counter.increment()
  end
end
```

### Registry: Process Discovery

**When you have many named processes:**
- Dynamic process naming
- Process lookups by custom keys
- Avoiding atom exhaustion

```elixir
defmodule MyApp.GameRegistry do
  def start_link do
    Registry.start_link(keys: :unique, name: __MODULE__)
  end
  
  def register_game(game_id, pid) do
    Registry.register(__MODULE__, game_id, pid)
  end
  
  def find_game(game_id) do
    case Registry.lookup(__MODULE__, game_id) do
      [{pid, _}] -> {:ok, pid}
      [] -> {:error, :not_found}
    end
  end
  
  def list_games do
    Registry.select(__MODULE__, [{{:"$1", :"$2", :"$3"}, [], [{{:"$1", :"$2"}}]}])
  end
end
```

## OTP Design Principles (KISS Emphasis)

### When to Use What (KISS Guide)

**Start Simple (80% of cases):**
```
Need state? → GenServer
Need supervision? → Supervisor (one_for_one)
Need one-off async? → Task
Need simple shared state? → Agent
```

**Advance When Needed (specific problems only):**
```
Spawning processes at runtime? → DynamicSupervisor
GenServer bottleneck (proven via profiling)? → PartitionSupervisor  
Many named processes? → Registry
Need supervised async tasks? → Task.Supervisor
```

### Red Flags (Premature Optimization)

**Don't do these without measuring:**
- Using PartitionSupervisor "just in case"
- Registry when you have < 100 processes
- Complex supervision trees without proven need
- DynamicSupervisor when static children work fine

### Decision Framework

**1. Start with simplest pattern that works**
```elixir
# Start here
defmodule MyApp.SimpleCounter do
  use GenServer
  # Basic implementation
end

# Not here
defmodule MyApp.DistributedPartitionedCounter do
  use PartitionSupervisor
  # Complex implementation you don't need yet
end
```

**2. Profile to find actual bottlenecks**
```elixir
# Use Observer to find real problems
:observer.start()

# Use telemetry for metrics
:telemetry.execute([:my_app, :counter, :increment], %{value: 1})
```

**3. Optimize specific problem with specific solution**
```elixir
# If profiling shows GenServer is bottleneck:
# THEN consider PartitionSupervisor

# If you're running out of atoms for process names:
# THEN consider Registry

# If you need runtime process spawning:
# THEN consider DynamicSupervisor
```

**4. Measure improvement, iterate**

## OTP Testing Patterns

### Testing GenServers

```elixir
defmodule MyApp.CounterTest do
  use ExUnit.Case
  
  setup do
    {:ok, pid} = MyApp.Counter.start_link(0)
    %{counter: pid}
  end
  
  test "increments count", %{counter: pid} do
    assert MyApp.Counter.get_count(pid) == 0
    MyApp.Counter.increment(pid)
    assert MyApp.Counter.get_count(pid) == 1
  end
  
  test "handles concurrent increments" do
    {:ok, pid} = MyApp.Counter.start_link(0)
    
    tasks = for _ <- 1..100 do
      Task.async(fn -> MyApp.Counter.increment(pid) end)
    end
    
    Enum.each(tasks, &Task.await/1)
    assert MyApp.Counter.get_count(pid) == 100
  end
end
```

### Testing Supervision Trees

```elixir
defmodule MyApp.SupervisorTest do
  use ExUnit.Case
  
  test "restarts failed children" do
    {:ok, supervisor_pid} = MyApp.Supervisor.start_link([])
    
    # Find counter process
    [{counter_pid, _}] = Registry.lookup(MyApp.Registry, MyApp.Counter)
    
    # Kill the process
    Process.exit(counter_pid, :kill)
    
    # Wait for restart
    :timer.sleep(100)
    
    # Verify new process is running
    [{new_counter_pid, _}] = Registry.lookup(MyApp.Registry, MyApp.Counter)
    assert new_counter_pid != counter_pid
    assert Process.alive?(new_counter_pid)
  end
end
```

## Cross-References

### System Architecture
For high-level system design, context boundaries, and distributed architecture patterns,
see the `elixir-architecture` skill.

### Phoenix Integration  
For integrating OTP patterns with Phoenix contexts, LiveView, and PubSub systems,
see the `elixir-phoenix-framework` skill.

### Performance Review
For profiling OTP applications, identifying process bottlenecks, and optimization strategies,
see the `elixir-review` skill.

## Summary

**Remember KISS:**
1. **Start simple**: GenServer + Supervisor covers most needs
2. **Profile before optimizing**: Measure actual problems, not theoretical ones  
3. **Advance when needed**: Use complex patterns to solve specific measured problems
4. **Test concurrent behavior**: OTP makes concurrency easier but testing is still important

The BEAM VM and OTP give you powerful tools for building fault-tolerant, concurrent systems. Use them wisely by starting simple and evolving complexity only when requirements demand it.