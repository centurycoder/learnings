# Zombie process vs. orphan process

| State | What happened | Still running? | What happens next? |
|---|---|---|---|
| **Zombie** | A process exited, but its exit status hasn’t been collected | No | Its current parent reaps it with `wait()`/`waitpid()` |
| **Orphan** | A process’s parent exited while the process was still running | Yes | The OS reparents it to a system reaper or designated subreaper |

## Zombie processes

When a child exits:

1. The OS releases its ordinary resources, including its memory.
2. A small process-table entry remains, holding its PID, **exit status,** and accounting information. (That's why the process-table needs remains)
3. Its parent calls `wait()` or `waitpid()` to collect that information. This is called **reaping**, and it removes the remaining entry.

Between exit and reaping, the process is a **zombie**. Tools such as `ps` show it as `Z` or `<defunct>`.

- Zombies consume **no CPU**, but occupy process-table slots. Large numbers can prevent new processes from starting.
- Brief zombies are normal while the parent has not yet collected the exit status.
- **Persistent zombies usually indicate a living parent that is not reaping its children.**
- `kill -9` cannot remove a zombie: it is already dead. Its current parent must reap it.

## Orphan processes and reparenting

If a parent exits before its running child:

1. The child becomes an **orphan**, not a zombie.
2. The OS reparents it to a system reaper—typically PID 1 (`init`/`systemd` on Linux, `launchd` on macOS), or a designated subreaper on Linux.
3. The child normally continues running.
4. When it eventually exits, its new parent normally collects its exit status promptly.

If the child was **already a zombie** when its parent exited, it is also reparented. Parent exit does not itself reap the zombie; the new parent must do so. A malfunctioning reaper can therefore leave zombies around.

The original parent does not need to call `wait()` or `waitpid()` before exiting for this reparenting and eventual cleanup to happen.

## Core rule

A process is a **zombie** precisely when **it has exited but has not yet been reaped**, regardless of whether its current parent is the original parent or an adoptive one.
