# Backend Deployment and Language Comparison

## What this is / why it matters
A backend service in Node.js, Java, Python, or Go all end up doing the same job — run business logic, talk to a database, answer requests on a port — but each language gets there differently: different file extensions, different tools to pull in dependencies, different ways of turning source code into something that actually runs. Knowing those differences matters the moment you have to deploy, troubleshoot, or just read someone else's service.

## How it works

### The deployment steps, regardless of language
The same general sequence applies whether the service is Node.js, Java, Python, or Go:
1. Install the programming language runtime
2. Create a directory to hold the application
3. Create a dedicated system user to run it
4. Download the application code
5. Extract the code
6. Install its dependencies
7. Create a systemd service file
8. If any data loading is required (e.g. seeding a database), do it
9. Start the service

A real systemd service file, for a Node.js service called `catalogue` in the Roboshop example project:
```ini
[Unit]
Description=Catalogue Service
After=network-online.target

[Service]
User=roboshop
Environment=MONGO_URL="mongodb://mongodb.daws92s.store:27017/catalogue"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=catalogue
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
Same pattern as the system-user/systemd setup in [07-3-tier.md](07-3-tier.md) — only the `ExecStart` command and environment variables change per language.

### Language comparison at a glance

| | Node.js | Java | Python | Go |
|---|---|---|---|---|
| File extension | `.js` | `.java` | `.py` | `.go` |
| Build tool | npm | Maven | pip | the `go` command (modules) |
| Build file | `package.json` + `package-lock.json` | `pom.xml` | `requirements.txt` | `go.mod` + `go.sum` |
| Install/build command | `npm install` | `mvn clean package` | `pip install -r requirements.txt` | `go build` |
| Where dependencies are stored | `node_modules/` inside each app's own directory | `~/.m2/repository` — one shared location on the machine | A shared system directory by default, or isolated inside the app via a `venv` | `$GOPATH/pkg/mod` — one shared module cache on the machine |
| Compiled or not | Not compiled — interpreted/JIT-compiled at run time | Compiled to bytecode, then run on the JVM | Not compiled — interpreted at run time | Compiled ahead of time to a native machine-code binary |

### Node.js
`npm install` reads `package.json` and downloads every dependency into a `node_modules` folder *inside that specific application's directory*. `package-lock.json` pins the exact versions actually installed (including sub-dependencies), so an install today matches an install next month. Because every app gets its own `node_modules`, two apps on the same server can use completely different versions of the same dependency without conflicting.

### Java
Java needs a build tool because its source has to be compiled before it can run. **Maven** is the standard one:
- Build file: `pom.xml` — declares the project's name, version, description, and dependencies.
- Dependencies aren't stored per-app like Node — Maven keeps one shared location on the machine, `~/.m2/repository`, and tells different versions of the same dependency apart using the **GAV format**: **G**roup-id, **A**rtifact-id, **V**ersion (e.g. `com.hdfcbank`, `netbanking`, `1.2.0`). That's what lets multiple projects share one dependency cache without version collisions.
- **Maven lifecycle** — building a project runs through a fixed sequence of phases: clean old builds → validate → install dependencies → compile the source → run tests → package the result. Running `mvn clean package` triggers that whole sequence and produces an artifact (a `.jar` file) inside a `target/` folder, e.g. `target/shipping.jar`.
- **JDK vs JRE** — the **JRE** (Java Runtime Environment) is what's needed to *run* already-compiled Java bytecode; the **JDK** (Java Development Kit) is the JRE plus the extra tools needed to *develop and build* Java code (including Maven's compiler). JDK is a superset of JRE — install the JDK on a build machine, the JRE alone is enough on a machine that only runs the final artifact.
- Multiple Java versions can coexist on one server (e.g. `java-21` and `java-25` installed side by side) — `JAVA_HOME` (commonly set in `/etc/profile.d/java.sh`) tells Maven and other tools exactly which installed version to use.

### Python
- File extension: `.py`. Build tool: **pip**. Build file: `requirements.txt`.
- `pip install -r requirements.txt` reads that file and installs every dependency — by default, into a shared system-wide directory, the same one every Python app on that machine uses.
- That shared-by-default behavior is exactly why two Python apps needing different versions of the same package conflict with each other. The fix is a **venv** (virtual environment): `python3.14 -m venv /app/.venv` creates an isolated set of dependency versions scoped to just that one app directory — the same per-app isolation Node.js gives you automatically with `node_modules`, just opt-in rather than the default for Python.
- Installing a specific Python version on RHEL: `dnf install python3.14 python3.14-pip -y`. `dnf module list python` (the older multi-version-stream approach) is deprecated in favor of installing versioned packages directly.

### Go
- File extension: `.go`. There's no separate build-tool binary to install — the `go` command itself handles dependencies and building, using **Go modules**.
- Build files: `go.mod` (the module's name and its direct dependencies) and `go.sum` (checksums that lock those dependencies to exact, verified versions — the same role `package-lock.json` plays for npm).
- Dependencies are cached once per machine in `$GOPATH/pkg/mod` (the module cache), not duplicated per project.
- `go build` compiles the source straight down to a single, self-contained native binary — there's no bytecode layer and no runtime needed on the machine that runs it, unlike Java (needs a JRE) or Node.js/Python (need their interpreter installed). This is the biggest practical difference: deploying a Go service can be as simple as copying one binary file to the server.

### Compiled vs interpreted, summarized
- **Go** is truly compiled ahead of time to native machine code — the fastest startup, and no runtime dependency on the target machine.
- **Java** compiles to an intermediate form (bytecode), which then runs on the JVM — portable across operating systems, but a JRE has to be present wherever it runs.
- **Node.js and Python** ship source code directly and interpret/JIT-compile it at run time — no separate build step before running, but the language runtime (Node or Python itself) has to be installed on the server.

## Common problems and how to solve them
- **Dependency version conflicts** — happens by default with Python (shared system directory) if two apps need different versions of the same package. Fix: isolate with a `venv` per app, the same way Node.js isolates with `node_modules` per app automatically.
- **Wrong language version picked up at build time** — with multiple Java (or Python) versions installed side by side, a build can silently use the wrong one. Fix: set `JAVA_HOME` explicitly (or point at the right Python/venv interpreter) rather than relying on whatever happens to be first on the system `PATH`.
- **Forgetting `daemon-reload` after editing a systemd service file** — covered in [07-3-tier.md](07-3-tier.md); applies identically here regardless of which language the service is written in.

## Key takeaways
- The deployment steps are the same shape across languages — install runtime, create directory and system user, fetch code, install dependencies, wire up a systemd service, start it.
- Node.js and Go isolate dependencies per project by default (`node_modules`, the Go module cache keyed by version); Java shares one cache using GAV-versioned coordinates (`~/.m2/repository`); Python shares by default but can be isolated with a `venv`.
- Only Java and Go actually compile before running — Java to portable bytecode (needs a JRE), Go to a native binary (needs nothing extra). Node.js and Python run the source directly.

See also: [07-3-tier.md](07-3-tier.md), [11-nginx.md](11-nginx.md)
