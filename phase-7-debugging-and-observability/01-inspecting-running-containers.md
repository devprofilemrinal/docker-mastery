# Inspecting Running Containers

Before diagnosing anything broken, you need fluency with the tools for looking at something that's running normally. This chapter is a working toolkit — `docker inspect`, `docker stats`, `docker exec`, `docker top`, and reading the cgroup/namespace files directly — building directly on mechanisms established across Phases 1, 3, and 4, now framed specifically as debugging tools.

---

## `docker inspect`: The Ground-Truth Configuration Source

```bash
docker inspect demo-service
```

This single command surfaces the actual, resolved configuration of a running container — not what you *intended* to run, but what's *actually* running. Specific fields worth knowing by heart:

```bash
# Exact resource limits actually applied (Phase 3, Chapter 3):
docker inspect demo-service --format '{{.HostConfig.Memory}}'
docker inspect demo-service --format '{{.HostConfig.NanoCpus}}'

# Network configuration actually in effect (Phase 4):
docker inspect demo-service --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool

# Mounted volumes (Phase 5):
docker inspect demo-service --format '{{json .Mounts}}' | python3 -m json.tool

# Exit/health state (Phase 3, this phase):
docker inspect demo-service --format '{{.State.Status}} {{.State.ExitCode}} {{.State.OOMKilled}}'
```

Whenever a container is behaving unexpectedly, `docker inspect` should be your first stop — before assuming an application bug, confirm the container's *actual* runtime configuration matches what you believe you configured.

---

## `docker stats`: Live Resource Usage

```bash
docker stats demo-service --no-stream
# CONTAINER   CPU %   MEM USAGE/LIMIT    MEM %   NET I/O          BLOCK I/O
# demo-svc    12.3%   287.4MiB/512MiB    56.1%   1.2MB / 3.4MB    0B / 12MB
```

This reads directly from the cgroup files covered in Phase 3 Chapter 3 (`memory.current`, `cpu.stat`) — it's a live, human-readable view of the same ground truth you can read yourself at `/sys/fs/cgroup/...`. Run without `--no-stream` for a continuously updating view while reproducing a suspected resource issue.

---

## `docker exec`: Getting a Shell Inside

```bash
docker exec -it demo-service sh
```

This starts a **new process inside the container's existing namespaces** (Phase 1, Chapter 2) — it does not create a new container, and it doesn't affect the container's actual PID 1. From inside, ordinary Linux tools work as expected:

```bash
docker exec demo-service ps aux            # process tree inside the container's PID namespace
docker exec demo-service cat /proc/1/status # detailed info on the container's actual PID 1
docker exec demo-service netstat -tlnp      # what's listening inside this container's network namespace
```

Remember from Phase 2 Chapter 4: this only works if the image has a shell at all — distroless images have none, and remote JVM debugging (Chapter 3 of this phase) becomes the alternative there.

---

## `docker top`: Process View From the Host, No Exec Needed

```bash
docker top demo-service
# UID    PID     PPID    CMD
# root   48213   48190   java -jar /app/app.jar
```

Unlike `docker exec ... ps`, this reads the container's process list **from the host's view** — useful when the image has no shell (works even on distroless images, since it doesn't require executing anything inside the container at all) or when you specifically want to correlate a container process with its real host-level PID for further host-side investigation (e.g., attaching `strace` or `perf` from the host).

---

## Reading cgroup and Namespace Files Directly

For anything `docker stats`/`inspect` doesn't surface precisely enough, go straight to the source, as established in Phase 3 Chapter 3:

```bash
docker exec demo-service cat /sys/fs/cgroup/memory.current
docker exec demo-service cat /sys/fs/cgroup/cpu.stat
docker exec demo-service cat /sys/fs/cgroup/pids.current
```

This is ground truth with zero abstraction — genuinely useful when you suspect Docker's own reporting tools might be stale, cached, or when working through an issue methodically and wanting to verify each layer of the stack independently rather than trusting a summarized view.

---

## Common Misconceptions

- **"`docker exec` gives you access to the container's actual init process."** It starts an entirely separate process inside the same namespaces — it has no special relationship to the container's PID 1 beyond sharing namespaces with it.
- **"`docker stats` and reading cgroup files directly can show different numbers."** They shouldn't — `docker stats` is a formatted view of exactly the same underlying cgroup files; if they seem to disagree, suspect a stale/cached view or a misunderstanding of the specific field being compared, not two independent data sources.
- **"You always need a shell in the container to inspect its process list."** `docker top` reads from the host side and requires no shell inside the container at all.

---

## What's Next

**Next:** [`02-debugging-a-container-that-wont-start.md`](./02-debugging-a-container-that-wont-start.md)