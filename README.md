# docker-agent-sandbox

A reusable Docker Compose template for running AI agents safely on your machine.

This repository is a base template. It does not install Codex, Claude Code, OpenCode, OpenClaude, Hermes, or any other agent. Future agent projects should reuse this structure in separate repositories.

```text
+----------------------------------------------+
|              docker-agent-sandbox            |
+----------------------------------------------+
| Host machine: Windows                        |
| Container: Linux environment                 |
| Agent user: non-root                         |
| Workspace: controlled bind mount             |
| Goal: reusable secure agent base             |
+----------------------------------------------+
```

## Why This Exists

AI agents can run commands, edit files, install tools, and automate work.

That is useful, but it also creates risk.

This sandbox limits what an agent can access and modify. The agent works inside a Docker container and only sees the folders we explicitly mount.

This repository should stay generic. Agent-specific installation steps belong in future projects that copy or extend this base.

## Project Structure

```text
C:\docker-agents
|-- docker-agent-sandbox/
|   |-- agent/
|   |   `-- Dockerfile
|   |-- workspace/
|   |-- memory/
|   |-- skills/
|   |-- data/
|   |-- config/
|   |-- docker-compose.yml
|   |-- .gitignore
|   `-- README.md
|
`-- secrets/
    `-- docker-agent-sandbox.env
```

## Folder Roles

```text
+-----------+------------------------------------------+------------+
| Folder    | Purpose                                  | Access     |
+-----------+------------------------------------------+------------+
| workspace | Projects and files the agent can edit    | read/write |
| memory    | Persistent notes, logs, and agent memory | read/write |
| skills    | Reusable procedures and agent skills     | read/write |
| data      | Future persistent service data           | read/write |
| config    | Configuration files                      | read-only  |
| agent     | Docker image definition                  | host only  |
| secrets   | Real API keys and passwords              | not mounted |
+-----------+------------------------------------------+------------+
```

## Main Concepts

### Dockerfile

The `Dockerfile` is the recipe used to build the image.

```text
Dockerfile -> Image -> Container
```

### Image

An image is a packaged environment. It contains Linux, Node.js, Python, Git, curl, and other tools.

### Container

A container is a running instance of an image.

### Bind Mount

A bind mount connects a real Windows folder to a folder inside the container.

Example:

```text
./workspace  ->  /workspace
```

If the agent creates a file in `/workspace`, you can see it on Windows inside `workspace/`.

### Persistent Storage

Persistent storage means data survives container restarts or recreation.

In this project, persistence comes from bind mounts:

```text
workspace/
memory/
skills/
data/
```

These folders live on Windows and are mounted into the Linux container.

## Installed Tools

The base image currently includes the following tools:

```text
+----------+--------------------------------------+
| Tool     | Purpose                              |
+----------+--------------------------------------+
| Node.js  | Run JavaScript and Node-based CLIs   |
| npm      | Install and run Node packages        |
| Python 3 | Run Python scripts and tools         |
| pip      | Install Python packages              |
| venv     | Create isolated Python environments  |
| Git      | Clone repositories and manage code   |
| curl     | Make HTTP requests and download data |
| bash     | Linux shell inside the container     |
+----------+--------------------------------------+
```

Current versions can be checked with:

```bash
node --version
npm --version
python3 --version
pip3 --version
git --version
curl --version
bash --version
```

These tools are installed in the Docker image, not directly on Windows.

## Main Commands

Build the image manually:

```powershell
docker build -t agent-sandbox:base ./agent
```

Start the sandbox and build the image if needed:

```powershell
docker compose up -d --build
```

Start the sandbox without rebuilding:

```powershell
docker compose up -d
```

Enter the running container:

```powershell
docker compose exec agent bash
```

Stop the sandbox:

```powershell
docker compose down
```

Validate the Compose file:

```powershell
docker compose config
```

Check running services:

```powershell
docker compose ps
```

## Security Model

The container is configured with several restrictions:

```text
+-------------------+-------------------------------------+
| Protection        | Meaning                             |
+-------------------+-------------------------------------+
| non-root user     | Agent does not run as administrator |
| cap_drop: ALL     | Removes extra Linux capabilities    |
| no-new-privileges | Blocks privilege escalation         |
| read_only: true   | Main filesystem is read-only        |
| tmpfs /tmp        | Temporary writable space            |
| memory limit      | Prevents excessive RAM usage        |
| CPU limit         | Limits CPU usage                    |
| pids limit        | Limits number of processes          |
| config read-only  | Agent cannot change config files    |
+-------------------+-------------------------------------+
```

## Network and Secrets Policy

This sandbox allows outbound internet access.

That means processes inside the container can download packages, clone repositories, call APIs, and access external services.

There are no inbound ports exposed by default, so external systems cannot connect into the agent container unless ports are explicitly added later.

```text
Inbound network:  blocked by default
Outbound network: allowed
```

Secrets must not be stored inside mounted folders.

Do not place API keys, passwords, tokens, or private credentials in:

```text
workspace/
memory/
skills/
data/
config/
```

The `config/` folder is mounted as read-only, but read-only still means readable. It prevents modification, not access.

Real secrets should live outside the project:

```text
C:\docker-agents\secrets\docker-agent-sandbox.env
```

This file is not mounted into the agent container by default.

Do not add this to the `agent` service unless you intentionally want the agent to receive those secrets:

```yaml
env_file:
  - C:\docker-agents\secrets\docker-agent-sandbox.env
```

Security rule:

```text
If the agent can read a file, it can potentially send its contents to the internet.
```

Only mount folders that the agent is allowed to read and modify.

## Mounted Paths

Inside the container:

```text
/workspace  -> editable project workspace
/memory     -> editable persistent memory
/skills     -> editable reusable skills
/data       -> editable future service data
/config     -> read-only configuration
```

## Expected Container User

Inside the container, this command:

```bash
whoami
```

Should return:

```text
agent
```

The agent should not run as `root`.

## Basic Test

Enter the container:

```powershell
docker compose exec agent bash
```

Then run:

```bash
echo "workspace ok" > /workspace/test.txt
echo "memory ok" > /memory/test-memory.txt
echo "skills ok" > /skills/test-skills.txt
echo "data ok" > /data/test-data.txt
echo "config test" > /config/test-config.txt
```

Expected result:

```text
/workspace  works
/memory     works
/skills     works
/data       works
/config     fails because it is read-only
```

## Git Policy

The project includes a `.gitignore`.

Runtime content is ignored:

```text
workspace/*
memory/*
data/*
```

Folder placeholders are kept:

```text
workspace/.gitkeep
memory/.gitkeep
data/.gitkeep
skills/.gitkeep
config/.gitkeep
```

The `skills/` folder is versionable on purpose, because skills are reusable procedures that should be reviewed and committed intentionally.

The `config/` folder is versionable, but it must not contain secrets.

## Operating Rules

Do not mount your full user folder.

Avoid mounts like:

```text
C:\Users
C:\
Desktop
Documents
Downloads
```

Only mount specific project folders that the agent is allowed to read and modify.

Before committing, always inspect what Git will include:

```powershell
git status
```

Do not use `git add .` blindly if agents have recently generated files.

## Current Goal

This sandbox is a reusable base.

Future agents such as Codex, Claude Code, OpenCode, OpenClaude, Hermes, or custom agents should be added later in separate projects or specialized images that reuse this template.
