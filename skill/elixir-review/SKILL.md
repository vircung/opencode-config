# Elixir Code Review Skill

## Overview
Comprehensive code review patterns for Elixir applications focusing on BEAM VM performance, security, concurrency safety, and code quality metrics.

## BEAM VM Performance Analysis

### Process Management
- **Process Count:** Monitor with `:observer.start()` - avoid creating >1M processes without justification
- **Process Isolation:** Ensure processes don't share mutable state - use message passing exclusively
- **Process Linking:** Review supervisor trees - ensure proper process linking and error handling strategies
- **Memory per Process:** Check process dictionary usage - avoid storing large data structures

### Concurrency Patterns Review
```elixir
# ❌ Avoid: Shared mutable state
Agent.update(pid, fn state -> Map.put(state, key, large_data) end)

# ✅ Prefer: Immutable message passing
GenServer.cast(pid, {:update, key, value})
```

### Performance Bottlenecks
- **Hot Code Paths:** Use `:fprof` or `:eprof` for function-level profiling
- **Memory Usage:** Check for memory leaks with `:observer` memory tab
- **Scheduler Utilization:** Monitor with `:scheduler.utilization(1)`
- **ETS Table Size:** Review ETS table growth patterns and cleanup strategies

## Elixir Security Checklist

### Input Validation
```elixir
# ✅ Always validate external input
def create_user(params) do
  changeset = User.changeset(%User{}, params)
  if changeset.valid? do
    Repo.insert(changeset)
  else
    {:error, changeset}
  end
end

# ❌ Avoid: Direct parameter usage
def create_user(params) do
  Repo.insert(%User{name: params["name"], email: params["email"]})
end
```

### Phoenix Security Patterns
- **CSRF Protection:** Ensure `protect_from_forgery` in controllers
- **Content Security Policy:** Configure CSP headers in endpoint configuration
- **Secure Headers:** Use `Plug.SecureHeaders` for HSTS, frame options
- **Input Sanitization:** Always use Ecto changesets for data validation

### Database Security
- **SQL Injection Prevention:** Use parameterized queries exclusively
- **Database Credentials:** Store in environment variables, never in code
- **Connection Pooling:** Review pool size configuration in production
- **Migration Safety:** Review migrations for data loss potential

## Concurrency Safety Review

### Race Conditions
```elixir
# ❌ Race condition potential
def increment_counter(pid) do
  current = GenServer.call(pid, :get)
  GenServer.call(pid, {:set, current + 1})
end

# ✅ Atomic operation
def increment_counter(pid) do
  GenServer.call(pid, :increment)
end
```

### Deadlock Prevention
- **Message Order:** Ensure consistent message ordering between processes
- **Timeout Usage:** Always specify timeouts for GenServer calls
- **Supervisor Strategy:** Review restart strategies - avoid cascading failures
- **Process Dependencies:** Map process dependency graphs to identify potential deadlocks

### Resource Management
```elixir
# ✅ Proper resource cleanup
def handle_call({:process_file, path}, _from, state) do
  try do
    file = File.open!(path, [:read])
    result = process_file_content(file)
    {:reply, {:ok, result}, state}
  after
    File.close(file)
  end
end
```

## Code Quality Metrics

### Cyclomatic Complexity
- **Function Length:** Functions >20 lines need justification
- **Pattern Matching:** Prefer pattern matching over conditional statements
- **Pipe Operator Usage:** Review pipe chains >5 steps for clarity
- **Module Size:** Modules >500 lines should be split by responsibility

### Error Handling Patterns
```elixir
# ✅ Explicit error handling
with {:ok, user} <- Users.find(id),
     {:ok, permissions} <- Auth.get_permissions(user),
     {:ok, data} <- fetch_data(user, permissions) do
  {:ok, data}
else
  {:error, :user_not_found} -> {:error, "User not found"}
  {:error, :insufficient_permissions} -> {:error, "Access denied"}
  {:error, reason} -> {:error, "Operation failed: #{reason}"}
end

# ❌ Avoid: Nested case statements
case Users.find(id) do
  {:ok, user} ->
    case Auth.get_permissions(user) do
      {:ok, permissions} -> fetch_data(user, permissions)
      {:error, _} -> {:error, "Access denied"}
    end
  {:error, _} -> {:error, "User not found"}
end
```

### Documentation Standards
- **Module Documentation:** Every public module needs `@moduledoc`
- **Function Documentation:** Public functions need `@doc` with examples
- **Type Specifications:** Use `@spec` for all public functions
- **Examples:** Include `@doc` examples that can be tested with doctests

## Red Flags in Elixir Code

### Process Anti-Patterns
- **Long-Running Processes:** Processes that block for >5 seconds without justification
- **Stateful Modules:** Module attributes used as mutable state
- **Process Leaks:** Processes created without proper supervision
- **Infinite Loops:** Recursive functions without proper termination conditions

### Performance Red Flags
- **String Concatenation:** Using `<>` in loops instead of building lists
- **Large Messages:** Sending >1MB messages between processes
- **Synchronous Calls:** Excessive use of `GenServer.call` vs `cast`
- **Database N+1:** Missing `Repo.preload` causing multiple queries

### Security Red Flags
```elixir
# ❌ Security anti-patterns
def execute_command(user_input) do
  System.cmd("sh", ["-c", user_input])  # Command injection risk
end

def build_query(table, user_where) do
  "SELECT * FROM #{table} WHERE #{user_where}"  # SQL injection risk
end

# ✅ Secure alternatives
def allowed_commands, do: ["ls", "pwd", "whoami"]

def execute_safe_command(cmd) when cmd in @allowed_commands do
  System.cmd(cmd, [])
end
```

## Cross-Skill References

- **Architecture Review:** Consult `elixir-architecture` for context boundary validation
- **OTP Patterns:** Use `elixir-otp` for supervisor tree and GenServer pattern review
- **Database Review:** Apply `elixir-ecto` patterns for query and schema validation
- **Phoenix Security:** Reference `elixir-phoenix-framework` for web-specific security patterns

## Tool Integration

### Static Analysis
```bash
# Run these tools during review
mix credo --strict              # Code quality analysis
mix dialyzer                    # Type checking
mix deps.audit                  # Security vulnerability scanning
mix format --check-formatted    # Code formatting verification
```

### Performance Monitoring
```elixir
# Add to review checklist
:observer.start()              # BEAM VM monitoring
:debugger.start()              # Process debugging
:fprof.start()                 # Function profiling
```

## Review Checklist Template

### Pre-Review Setup
- [ ] Run `mix test` - all tests pass
- [ ] Run `mix credo --strict` - no issues
- [ ] Run `mix dialyzer` - no type errors
- [ ] Check git diff for sensitive data

### Code Review Focus Areas
- [ ] Process design follows OTP principles
- [ ] Error handling uses `with` statements appropriately
- [ ] Database queries use Ecto changesets
- [ ] Security: Input validation present
- [ ] Performance: No obvious bottlenecks
- [ ] Documentation: Public functions documented
- [ ] Tests: Edge cases covered

### Phoenix-Specific Additions
- [ ] Controllers are thin, contexts handle business logic
- [ ] LiveView updates use proper `assign` patterns
- [ ] Routes follow RESTful conventions
- [ ] CSRF protection enabled for forms

Use this skill to ensure Elixir code meets BEAM VM best practices, security standards, and performance requirements while maintaining the functional programming paradigms that make Elixir powerful.