# Managing Child Processes in Elixir with GenServer: Linking, Monitoring, and Supervision Patterns

In Elixir applications, GenServers often manage dynamic tasks and temporary workers. One common pattern is a GenServer managing a multiplayer match, with related processes like timers running alongside. In this article, we explore robust patterns for:

- Handling GenServer crashes
- Spawning and linking child processes (like timers)
- Monitoring and reacting to child exits
- Lifecycle management without full-blown supervision trees

Let’s walk through this with a simple multiplayer match process and an associated timer.

---

## 1. GenServer Crash Behavior

When a GenServer crashes, its linked processes are also terminated unless they’re supervised separately or explicitly unlinked. By default, any state held in the GenServer is lost unless it's restored externally (e.g., from a database or ETS).

### 🔥 Example

```elixir
defmodule MatchServer do
  use GenServer

  def start_link(match_id) do
    GenServer.start_link(__MODULE__, %{id: match_id, players: [], timer_ref: nil}, name: via(match_id))
  end

  defp via(id), do: {:via, Registry, {MyApp.MatchRegistry, id}}
end
```

If `MatchServer` crashes, all internal state is gone — including any reference to a timer.

---

## 2. Spawning Child Processes from a GenServer

To start related child processes (like a countdown timer), use `DynamicSupervisor`. This ensures the timer is supervised and restartable.

```elixir
defmodule TimerServer do
  use GenServer

  def start_link(duration_ms) do
    GenServer.start_link(__MODULE__, duration_ms)
  end

  def init(duration) do
    Process.send_after(self(), :done, duration)
    {:ok, duration}
  end

  def handle_info(:done, state) do
    Process.exit(self(), :normal)
    {:noreply, state}
  end
end
```

In `MatchServer`, spawn the timer like this:

```elixir
{:ok, pid} = DynamicSupervisor.start_child(MatchTimerSupervisor, {TimerServer, 5_000})
```

You **do not** need another supervisor inside your GenServer. Let a separate `DynamicSupervisor` (defined in your app supervision tree) manage timer lifecycles.

---

## 3. Monitoring and Being Notified of Child Process Termination

Use `Process.monitor/1` to get `:DOWN` messages when a child terminates.

```elixir
{:ok, pid} = DynamicSupervisor.start_child(MatchTimerSupervisor, {TimerServer, 5_000})
ref = Process.monitor(pid)
{:noreply, %{state | timer_ref: ref}}
```

Then handle it in `handle_info/2`:

```elixir
def handle_info({:DOWN, ref, :process, _pid, reason}, state) do
  IO.puts("Timer finished or crashed: #{inspect(reason)}")
  broadcast_timer_complete()
  {:noreply, %{state | timer_ref: nil}}
end
```

---

## 4. Starting and Linking a Related Timer Process

If you want the timer to **crash when the match crashes**, link it:

```elixir
pid = spawn_link(fn -> run_timer(5_000) end)
```

Or with a GenServer:

```elixir
{:ok, pid} = GenServer.start_link(TimerServer, 5_000)
Process.link(pid)
```

You can **link and monitor** at the same time if you want crash propagation **and** notifications:

```elixir
Process.link(pid)
ref = Process.monitor(pid)
```

---

## 5. Handling Events When the Timer Completes Normally

You’ll get a `:DOWN` message with reason `:normal` if the timer exits gracefully.

```elixir
def handle_info({:DOWN, _ref, :process, _pid, :normal}, state) do
  broadcast("match:timer_done", %{match_id: state.id})
  {:noreply, state}
end
```

This is where you can trigger side effects like ending the match, notifying players, etc.

---

## 6. Ensuring Child Processes Terminate When Parent Crashes

To kill the timer if the match crashes, use `Process.link/1`. This makes sure both live and die together.

```elixir
{:ok, pid} = GenServer.start_link(TimerServer, 5_000)
Process.link(pid)
```

If `MatchServer` crashes, the linked `TimerServer` will receive an exit signal and shut down too.

---

## 7. Stopping or Cleaning Up All Child Processes When Parent Terminates

Rather than supervising timers under the match GenServer, use `link` for simple lifecycle tie-ins. No need to manage child pids manually or trap exits unless you want fine-grained shutdown.

Alternatively, trap exits and clean up manually:

```elixir
def init(state) do
  Process.flag(:trap_exit, true)
  {:ok, state}
end

def terminate(_reason, state) do
  if state.timer_pid, do: Process.exit(state.timer_pid, :kill)
end
```

---

## 8. High-Level Pattern Summary

| Technique                      | Use Case                                        |
|-------------------------------|--------------------------------------------------|
| `DynamicSupervisor`           | Start child GenServers dynamically               |
| `Process.monitor/1`           | Be notified when a process terminates            |
| `Process.link/1`              | Crash child when parent crashes                  |
| `handle_info({:DOWN, ...})`   | React to child finishing/crashing                |
| Manual cleanup / `terminate`  | Kill children on exit if not linked              |

---

## Conclusion

When building systems like multiplayer matches in Elixir, managing related processes like timers can be simple and robust:

- Use `DynamicSupervisor` for child management.
- Use `monitor` to react to child exits.
- Use `link` to tie process lifecycles.
- Let the parent handle side effects like broadcasting or cleanup.

This pattern scales well without needing a full supervision tree per GenServer. Perfect for simple, fault-tolerant multiplayer games.
