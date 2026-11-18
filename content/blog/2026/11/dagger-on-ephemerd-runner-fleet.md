---
title: "Running Dagger on an ephemerd Runner Fleet"
description: "Dagger decides how CI steps execute while ephemerd and GitHub Actions decide where the job runs. Here is how to wire one shared Dagger engine to a fleet of ephemeral self-hosted runners."
date: 2026-11-18T09:00:00-07:00
tags: [ infrastructure, ephemerd ]
categories: [ ci, containers ]
draft: false
---

A friend asked me if Dagger replaces ephemerd. It doesn't. They sit at different layers. Since I just rode this merry-go-round here is how I put the two together, with every command run against a real engine on a Linux box.

### Two layers, not two competitors

[Dagger](https://dagger.io) is a client plus an engine. The engine is a long-running daemon that wraps BuildKit. Every step in a pipeline runs as a container under runc inside that engine. The client is the `dagger` CLI or an SDK. It sends the pipeline to the engine and prints the result.

Dagger has no scheduler and no event source. It does not speak the GitHub Actions runner protocol. It fetches git repos only when a pipeline asks for them. Something else has to receive the `workflow_job` event and provide a machine to run the client on.

That something is GitHub Actions plus [ephemerd](https://github.com/ephemerd/ephemerd). ephemerd receives the `workflow_job` webhook and registers a just-in-time runner for that one job. The job runs in a fresh single-use container on a runner node. When it finishes the container is gone. To Dagger an ephemerd runner is an ordinary client host. Nothing more.

```
GitHub workflow_job event
        |
        v
ephemerd node  --->  fresh runner container (unprivileged)
                              |
                              |  dagger call  (the client)
                              v
                     Dagger engine (privileged, one trusted host)
                              |
                              |  runc
                              v
                     one container per pipeline step
```

ephemerd decides where and when a job runs. Dagger decides how the steps inside that job execute. Swapping `go test` in a `run:` step for `dagger call test` changes the second thing only.

One limit up front. The Dagger engine runs Linux containers and nothing else. The CLI runs on macOS and Windows but execution is always Linux. So this post applies to ephemerd's Linux pools. It gives you nothing on the Windows and macOS runners. Those are the pools where ephemerd earns its keep. Dagger has no answer there.

### Prerequisites

- A Linux host for the engine with Docker or Podman and a disk you don't mind filling. I used a Fedora 44 Podman machine with rootful podman.
- The Dagger CLI at the same version as the engine. I used v0.21.12, released November 17, 2026.
- An ephemerd Linux pool and a repo whose workflows can target it.
- Network reachability from the runner pool to the engine host on one TCP port. More on that below.

All of it is open source. The engine image `registry.dagger.io/engine` is Apache-2.0. Dagger Cloud is not required. A local volume at `/var/lib/dagger` is the cache. Cloud only adds a distributed cache and trace viewing on top.

### Pick a model

There are two ways to run this. I recommend the second one.

**Model A, an engine per job.** The job itself runs `docker run --privileged registry.dagger.io/engine` and points the CLI at it. No shared infrastructure. The cost is a cold cache on every job and privileged Docker inside an untrusted runner. Details in the appendix.

**Model B, one shared engine as a trusted fleet peer.** You stand up one privileged engine on a dedicated host with a persistent `/var/lib/dagger`. The ephemerd runners stay unprivileged. They run the CLI and set one environment variable. Privilege lives on one box you control and the cache stays warm across jobs. This fits how ephemerd already treats a runner. The runner is a client. It is not infrastructure.

### Model B: start the engine

The engine needs root capabilities including `CAP_SYS_ADMIN`. The Dagger docs say it plainly: "Currently, the Dagger Engine cannot be run as a rootless container". So this is `--privileged` on a host you treat as a trust boundary. Anyone who can reach the port can run containers on it.

Name the container and name the volume. Anonymous volumes pile up and nobody finds them again. Pass `--addr` to add a TCP listener. The engine keeps its unix sockets too.

Docker:

```sh
docker run -d --name dagger-engine --privileged --restart always \
  -v dagger-engine-cache:/var/lib/dagger \
  -p 2345:2345 \
  registry.dagger.io/engine:v0.21.12 \
  --addr tcp://0.0.0.0:2345
```

Podman, as root:

```sh
sudo podman run -d --name dagger-engine --privileged \
  -v dagger-engine-cache:/var/lib/dagger \
  -p 2345:2345 \
  registry.dagger.io/engine:v0.21.12 \
  --addr tcp://0.0.0.0:2345
```

The image is 671 MB. Check the log once it is up:

```
$ sudo podman logs dagger-engine 2>&1 | grep -v '^\s*#' | tail -6
level=info msg="auto snapshotter: using overlayfs"
level=debug msg="starting main engine api listeners"
level=warning msg="TLS is not enabled for tcp://0.0.0.0:2345. enabling mutual TLS authentication is highly recommended"
level=info msg="running server on /run/dagger/engine.sock"
level=info msg="running server on /run/buildkit/buildkitd.sock"
level=info msg="running server on [::]:2345"
```

Two things to notice. The snapshotter is overlayfs. That is why the volume matters. Without a real volume you get overlayfs on overlayfs and the docs warn it gets extremely slow. The TLS warning is real. A plain `tcp://` listener has no authentication. The engine supports mutual TLS with `--tlscert`, `--tlskey` and `--tlscacert`. On a private fleet VLAN with a firewall in front I skipped it for this walkthrough. On a shared network set it up.

### Model B: point a client at it

Install the CLI at the engine's version. The install script takes a `BIN_DIR` so it needs no root.

```sh
curl -fsSL https://dl.dagger.io/dagger/install.sh \
  | DAGGER_VERSION=0.21.12 BIN_DIR=$HOME/.local/bin sh
dagger version
```

```
dagger v0.21.12 (image://registry.dagger.io/engine:v0.21.12) linux/amd64
```

That parenthetical is the default engine source. With no override the CLI pulls that image and starts its own engine through whatever container runtime it finds. We don't want that on a runner. The override is `_EXPERIMENTAL_DAGGER_RUNNER_HOST`.

```sh
export _EXPERIMENTAL_DAGGER_RUNNER_HOST=tcp://127.0.0.1:2345
dagger --progress=plain core container from --address=alpine:3.21 \
  with-exec --args=echo,hello,from,dagger stdout
```

Trimmed output:

```
8   : Container.from DONE [2.3s]
9   : Container.withExec DONE [0.1s]
9   : [1.0s] | hello from dagger
10  : Container.stdout DONE [1.0s]

hello from dagger
```

Every invocation also prints a connect block near the top. This one is from a `dagger functions` call later in the walkthrough:

```
3   : connecting to engine
3   : [0.0s] | INF connected name=a24f4fd5fdc3 client-version=v0.21.12 server-version=v0.21.12
3   : connecting to engine DONE [0.0s]
```

The `connected name=a24f4fd5fdc3` line is the proof. That is the engine container's ID. Look for it in every job log. If it is missing the CLI went somewhere else.

The variable accepts other schemes. `container://NAME` connects to a running engine container through the local runtime. `container+podman://NAME` forces podman. `unix://PATH` and `kube-pod://POD?namespace=NS` exist too. I tested `container+podman://dagger-engine` against the same engine and got the same `connected name=a24f4fd5fdc3` line. For a shared engine on another host `tcp://` is the one you want.

### Model B: a real pipeline as a module

A one-liner proves the loop. A module proves the workflow. This is a tiny Go app with a test plus a Dagger module written in the Go SDK. You do not need Go on the client. The SDK runtime builds inside the engine.

The app:

```go
// main.go
package main

import (
	"fmt"
	"os"
)

func Greeting(name string) string {
	if name == "" {
		name = "fleet"
	}
	return fmt.Sprintf("hello, %s", name)
}

func main() {
	fmt.Println(Greeting(os.Getenv("HELLO_NAME")))
}
```

```go
// main_test.go
package main

import "testing"

func TestGreeting(t *testing.T) {
	cases := map[string]string{
		"":       "hello, fleet",
		"luther": "hello, luther",
	}
	for in, want := range cases {
		if got := Greeting(in); got != want {
			t.Errorf("Greeting(%q) = %q, want %q", in, got, want)
		}
	}
}
```

Initialize the module in a `.dagger` subdirectory so it stays out of the app's package tree:

```sh
dagger init --sdk=go --name=hello-ci --source=.dagger
cat dagger.json
```

```json
{
  "name": "hello-ci",
  "engineVersion": "v0.21.12",
  "sdk": {
    "source": "go"
  },
  "source": ".dagger"
}
```

Replace the generated `.dagger/main.go` with three functions. `Test` runs vet and test. `Build` returns a static binary as a file. `Run` drops that binary into a clean alpine image and runs it.

```go
// .dagger/main.go
package main

import (
	"context"
	"dagger/hello-ci/internal/dagger"
)

type HelloCi struct{}

// base is a Go toolchain container with the source mounted and the
// module and build caches attached as engine cache volumes.
func (m *HelloCi) base(source *dagger.Directory) *dagger.Container {
	return dag.Container().
		From("golang:1.25-alpine").
		WithMountedCache("/go/pkg/mod", dag.CacheVolume("go-mod")).
		WithMountedCache("/root/.cache/go-build", dag.CacheVolume("go-build")).
		WithMountedDirectory("/src", source).
		WithWorkdir("/src")
}

// Test runs go vet and go test and returns their output.
func (m *HelloCi) Test(
	ctx context.Context,
	// +defaultPath="/"
	// +ignore=[".dagger", "out"]
	source *dagger.Directory,
) (string, error) {
	return m.base(source).
		WithExec([]string{"go", "vet", "./..."}).
		WithExec([]string{"go", "test", "-v", "./..."}).
		Stdout(ctx)
}

// Build compiles a static linux binary and returns it as a file.
func (m *HelloCi) Build(
	// +defaultPath="/"
	// +ignore=[".dagger", "out"]
	source *dagger.Directory,
) *dagger.File {
	return m.base(source).
		WithEnvVariable("CGO_ENABLED", "0").
		WithExec([]string{"go", "build", "-o", "/out/hello", "."}).
		File("/out/hello")
}

// Run drops the built binary into a clean alpine image and runs it.
func (m *HelloCi) Run(
	ctx context.Context,
	// +defaultPath="/"
	// +ignore=[".dagger", "out"]
	source *dagger.Directory,
	// +default="fleet"
	name string,
) (string, error) {
	return dag.Container().
		From("alpine:3.21").
		WithFile("/usr/local/bin/hello", m.Build(source)).
		WithEnvVariable("HELLO_NAME", name).
		WithExec([]string{"hello"}).
		Stdout(ctx)
}
```

The `+defaultPath="/"` pragma makes the module root the default source. The `+ignore` pragma keeps the module's own generated code and the build output out of the upload. The two `CacheVolume` mounts are Go's module and build caches. They live in the engine and survive across jobs. That is the whole reason for a shared engine.

Regenerate the bindings and list the functions:

```sh
dagger develop
dagger functions
```

```
Name    Description
build   Build compiles a static linux binary and returns it as a file.
run     Run drops the built binary into a clean alpine image and runs it.
test    Test runs go vet and go test and returns their output.
```

Now the first test run. The engine has to pull `golang:1.25-alpine` and run the tests:

```sh
time dagger --progress=plain call test
```

```
11  : Container.from DONE [8.7s]
16  : Container.withExec DONE [0.0s]
17  : Container.withExec DONE [0.9s]
17  : [12.1s] | === RUN   TestGreeting
17  : [12.1s] | --- PASS: TestGreeting (0.00s)
17  : [12.1s] | PASS
17  : [12.1s] | ok  	example.com/hello	0.002s
9   : HelloCi.test DONE [14.0s]

real	0m14.355s
```

Run it again with nothing changed:

```
9   : HelloCi.test CACHED [0.0s]

=== RUN   TestGreeting
--- PASS: TestGreeting (0.00s)
PASS
ok  	example.com/hello	0.002s

real	0m0.711s
```

14.4 seconds down to 0.7 seconds. Nothing re-executed. On an ephemeral runner with its own engine the second run would be 14 seconds again because the cache died with the job.

Build and export the binary, then run it through the alpine stage:

```sh
dagger --progress=plain call build export --path=out/hello
```

```
20  : export file hello to host out/hello DONE [0.0s]
19  : File.export DONE [0.3s]

real	0m1.436s
```

```sh
head -c 4 out/hello | od -c | head -1
```

```
0000000 177   E   L   F
```

```sh
dagger --progress=plain call run --name=ephemerd
```

```
9   : HelloCi.run DONE [1.2s]

hello, ephemerd
```

A 2.3 MB static ELF binary and a run through a second container. After these runs the cache volume on the engine held 1.6 GB. That is the Go image, the module cache and the build cache. It stays there for the next job.

### Model B: the workflow

This is the full workflow for an ephemerd Linux pool. It installs the CLI into `$RUNNER_TEMP`, checks the engine port is open, then calls the module. The `--progress=plain` flag matters. There is no TTY on a runner and the default progress output is unreadable in the job log.

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

env:
  DAGGER_VERSION: "0.21.12"

jobs:
  test-and-build:
    # An ephemerd Linux pool claims this. Match whatever labels your pool
    # requires in its dispatch policy.
    runs-on: [self-hosted, linux, x64]
    env:
      # The shared engine on the trusted fleet host. The runner VLAN must be
      # allowed to reach this port. If it is blocked the CLI hangs at
      # "connecting to engine" until the job times out.
      _EXPERIMENTAL_DAGGER_RUNNER_HOST: tcp://dagger-engine.fleet.internal:2345
      DAGGER_NO_NAG: "1"
    steps:
      - uses: actions/checkout@v4

      - name: Install the Dagger CLI
        run: |
          curl -fsSL https://dl.dagger.io/dagger/install.sh \
            | DAGGER_VERSION="$DAGGER_VERSION" BIN_DIR="$RUNNER_TEMP/bin" sh
          echo "$RUNNER_TEMP/bin" >> "$GITHUB_PATH"

      - name: Check the engine is reachable
        run: |
          dagger version
          timeout 5 bash -c 'cat < /dev/null > /dev/tcp/dagger-engine.fleet.internal/2345'

      - name: Test
        run: dagger --progress=plain call test

      - name: Build
        run: dagger --progress=plain call build export --path=out/hello

      - uses: actions/upload-artifact@v4
        with:
          name: hello
          path: out/hello
```

The reachability step is deliberate. It fails fast with a clear message when the firewall is wrong. Without it you get the behavior described in the gotchas below.

If you prefer the official action, `dagger/dagger-for-github@v8.4.1` installs the CLI and runs one verb. It picks up `_EXPERIMENTAL_DAGGER_RUNNER_HOST` from the job environment like any other step:

```yaml
      - uses: dagger/dagger-for-github@v8.4.1
        with:
          version: "0.21.12"
          verb: call
          args: test
```

I validated both workflows with actionlint 1.7.12.

### Model B: the ephemerd side

Nothing changes in the runner container. It stays unprivileged. It needs curl, bash and outbound access to download the CLI. The source directory is uploaded to the engine over the same TCP connection. The build output comes back the same way.

Three fleet-level notes:

1. **Linux pools only.** Put the Dagger workflows on `runs-on: [self-hosted, linux, x64]` or your arm64 label. Windows and macOS jobs keep their plain `run:` steps.
2. **Open the port.** The runner VLAN needs a firewall rule allowing TCP to the engine host on 2345. Allow that one destination and nothing else. Verify it from inside a job, not from your laptop.
3. **Pin versions together.** The CLI version in the workflow and the engine image tag must match. Bump them in the same change.

If you gate dispatch with ephemerd's `required_labels`, add that label to `runs-on` too. The workflow above uses the plain label set so it works on a pool with no gate.

### Appendix: Model A, an engine per job

If you have no spare host and your pool allows privileged Docker inside the runner, the job can bring its own engine. This is what the CLI does by default when it finds a Docker socket and no runner host variable.

```yaml
name: ci-engine-per-job

on:
  pull_request:

jobs:
  test:
    # Only a pool that allows privileged Docker can run this.
    runs-on: [self-hosted, linux, x64, privileged]
    steps:
      - uses: actions/checkout@v4

      - name: Start a throwaway engine
        run: |
          docker run -d --name dagger-engine --privileged \
            -v dagger-engine:/var/lib/dagger \
            registry.dagger.io/engine:v0.21.12

      - name: Test
        uses: dagger/dagger-for-github@v8.4.1
        env:
          _EXPERIMENTAL_DAGGER_RUNNER_HOST: container://dagger-engine
          DAGGER_NO_NAG: "1"
        with:
          version: "0.21.12"
          verb: call
          args: test

      - name: Stop the engine
        if: always()
        run: docker rm -f dagger-engine
```

Be honest with yourself about what this is. A privileged container inside a runner that executes code from a pull request. ephemerd isolates jobs from each other with single-use containers. My fleet adds a VLAN and an egress firewall on top. Privileged Docker-in-Docker is a hole through that. Keep Model A on trusted pools that never see fork PRs. The cache is also cold every single time. Every job pulls the Go image again and recompiles from scratch. For the sample above that is the 14 second run on every push instead of the 0.7 second one.

### Gotchas and limits

**Privileged is not optional.** The engine will not start rootless. The docs are explicit about it and I didn't find a workaround. Treat the engine host like you treat a Docker host. Anyone with the port has root-equivalent access to that box.

**The wrong variable name fails silently.** The 1.0 beta docs at docs.dagger.io rename the setting to `DAGGER_ENGINE`. On v0.21.12 that name does nothing. I tested it. The call succeeded, which looked fine. Then `podman ps` showed a `dagger-engine-v0.21.12` container the CLI had quietly started on the client, plus a second 671 MB copy of the image. On a runner with Docker available that is exactly the Model A behavior you were trying to avoid. On a runner without Docker it is a confusing failure. Grep the job log for `connected name=` and confirm it is your engine's container ID.

**A blocked port does not error. It hangs.** I pointed the CLI at a closed port to see what a bad firewall rule looks like:

```
$ _EXPERIMENTAL_DAGGER_RUNNER_HOST=tcp://127.0.0.1:9 timeout 30 dagger --progress=plain call test
1   : connect
2   : starting engine DONE [0.0s]
3   : connecting to engine
$ echo $?
124
```

It sat at `connecting to engine` until `timeout` killed it. No message. On a real fleet that means a job that burns its whole time budget before GitHub cancels it. This is the same class of bug as the pull-through registry cache on my fleet. It sat at a zero percent hit ratio after the runners moved to an isolated VLAN. They could no longer route to it. Nothing failed. Pulls went to the internet and the cache sat idle. If your runners live behind an egress firewall, open the engine port explicitly and put the `/dev/tcp` check in the workflow.

**No shared engine means no cache.** The Dagger docs say it directly: "Do not run engines on a pool of ephemeral machines. Each new machine starts with an empty cache." An ephemerd runner is the definition of an ephemeral machine.

**Linux execution only.** The engine runs Linux containers. A Windows build or a macOS signing step cannot run through Dagger. Those stay as plain steps on ephemerd's Windows and macOS pools.

**`tcp://` has no auth.** The engine warns about it on every start. Mutual TLS is supported with `--tlscert`, `--tlskey` and `--tlscacert`. Use it if the engine is reachable from any network you don't fully control.

**Give the engine disk.** Default garbage collection keeps the cache under 75 percent of the disk and leaves 20 percent free. You can tune `maxUsedSpace`, `reservedSpace` and `minFreeSpace` under `gc` in an `engine.json` mounted at `/etc/dagger/engine.json`. A small disk means constant eviction and a cache that never gets warm.

### Why this is nice

The runner does not get more powerful. It stays a disposable unprivileged container that runs a CLI. All the privilege and all the state sit on one host you can name. That is the posture ephemerd already has. A runner is a client of the fleet. Dagger fits that shape without a fight.

You also get real caching on ephemeral runners without paying for Dagger Cloud. And the pipeline definition is Go code that runs the same way on a laptop as it does on the fleet. Point your local CLI at the same engine and you share the cache with CI.

### What I tested

Everything in the Model B sections ran on a Fedora 44 Podman machine under WSL2 with rootful podman 5.8.5. The engine was `registry.dagger.io/engine:v0.21.12` started with the podman command above. My test container and volume carried a `-blog-` in their names. The CLI was v0.21.12 on the same host and connected over `tcp://127.0.0.1:2345`. Every terminal block in those sections is captured output, trimmed of progress noise. The `container+podman://` scheme, the dead-port hang, the `DAGGER_ENGINE` fallback and the cache volume size were all observed the same way. Both workflow files pass actionlint 1.7.12 with a config declaring the custom `privileged` label.

What I did not run here: a job on an actual ephemerd runner, the fleet firewall rule and the Model A workflow end to end. Those are written from the ephemerd docs and the Dagger docs and the commands follow the tested pattern. Run the reachability step first when you set it up.

### Links

- [Dagger](https://github.com/dagger/dagger) and the [self-hosting docs](https://docs.dagger.io/self-hosting)
- [Dagger 0.21 custom runner reference](https://docs.dagger.io/0.21/reference/configuration/custom-runner)
- [Dagger 0.21 engine configuration](https://docs.dagger.io/0.21/reference/configuration/engine)
- [dagger-for-github action](https://github.com/dagger/dagger-for-github)
- [ephemerd](https://github.com/ephemerd/ephemerd)
- [mayfly](https://github.com/ephemerd/mayfly)
- [actionlint](https://github.com/rhysd/actionlint)
