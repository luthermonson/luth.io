---
title: "Maintaining a Go Tool That Edits /etc/hosts on Every OS"
description: "Notes from maintaining goodhosts, the hosts file CLI and Go library behind vagrant-goodhosts and ddev. It covers Windows quirks, a HostsPerLine bug, and making the CLI testable."
date: 2024-02-03T19:22:00-07:00
tags:
 - go
 - goodhosts
 - open source
 - windows
 - testing
categories: [ go, tools, windows ]
images:
 - src: img/2024/02/goodhosts-hosts-per-line.svg
   alt: "hosts file output from hostsfile v0.1.4, v0.1.5 and v0.1.6 for the same sequence of adds"
   stretch: "none"
draft: false
---

Editing the hosts file sounds like the most boring problem in software. Read a text file, add a line, write it back. Then you find out
your binary is being shelled out to from a Ruby Vagrant plugin on a Windows laptop with UAC prompts. Your library is being called in a
loop by [ddev](https://github.com/ddev/ddev). And every hosts file you parse was hand-edited by somebody at some point. I've been
maintaining [goodhosts](https://github.com/goodhosts) for a few years now. I just spent a chunk of the fall and the holidays on it, so
here's what that looks like.

### Where it came from

I didn't start it. [Lex Toumbourou](https://github.com/lextoumbourou) wrote the original `goodhosts` CLI in Go in April 2015, and Windows
support landed that June. In January 2020 I moved it into the [goodhosts](https://github.com/goodhosts) org and split it in two:

- [goodhosts/hostsfile](https://github.com/goodhosts/hostsfile), the Go library that parses a hosts file into structs, indexes it, and
  flushes it back to disk
- [goodhosts/cli](https://github.com/goodhosts/cli), the `goodhosts` binary on top of it (`add`, `remove`, `list`, `check`, `clean`,
  `backup`, `restore`, `edit`)

The biggest consumer of the CLI is [vagrant-goodhosts](https://github.com/goodhosts/vagrant), which has a longer history than either.
It's descended from vagrant-hostsupdater (first commit April 2013). In May 2020 Daniele Scasciafratte swapped its Ruby hosts file
handling for the goodhosts CLI. The gem bundles a prebuilt binary per platform and picks one at runtime. This is trimmed, and the current
version also branches on arm64 vs amd64:

```ruby
def get_os_binary
  if OS.windows?
    return 'cli.exe'
  elsif OS.mac?
    return 'cli_osx'
  elsif OS.linux?
    return 'cli'
  else
    raise Error::WebDriverError, "unknown os: #{host_os.inspect}"
  end
end
```

That's the main reason the CLI is Go. Ruby plugins that edit system files on three operating systems are miserable. A static binary you
can drop next to the gem works on all three. ddev uses the library directly.

### "Every OS" is two files

The OS-specific surface of the library is small, and I want to keep it that way. Unix-likes get this:

```go
//go:build !windows

package hostsfile

var (
	HostsPerLine  = -1 // unlimited
	HostsFilePath = "/etc/hosts"
	eol           = "\n"
)

func (h *Hosts) preFlush() error  { return nil }
func (h *Hosts) postFlush() error { return nil }
```

And Windows gets this:

```go
package hostsfile

var (
	HostsPerLine  = 9
	HostsFilePath = "${SystemRoot}/System32/drivers/etc/hosts"
	eol           = "\r\n"
)

func (h *Hosts) preFlush() error {
	// need to force hosts per line always on windows see https://github.com/goodhosts/hostsfile/issues/18
	h.HostsPerLine(HostsPerLine)
	return nil
}
```

The `HostsPerLine` bit came from [issue #18](https://github.com/goodhosts/hostsfile/issues/18). A Vagrant user had 16 hostnames on one
`127.0.0.1` line and sites stopped loading. Windows only honours the first 9 names on a hosts line. That's undocumented, but easy to
reproduce. So on Windows every flush splits long lines into chunks of 9. Most of the other platform fixes over the years came from users
too:

- Till Wegmüller fixed the build tags so it builds on the other POSIX platforms.
- Guillaume Rose closed a file descriptor that leaked after the writable check.
- Christophe Fergeau added a missing `+build` directive.

### The bug that only shows up in a loop

From the end of September through October I did a round of cleanup on the library:

- A reindexer for the ip and hostname position lookups.
- A fix for duplicate removal reordering lines.
- Keeping comments when cleaning.
- Moving the build to [mage](https://magefile.org/).
- Coverage reporting.

That went out as v0.1.2 through v0.1.4. A few hours after the v0.1.4 release, Randy Fay from ddev opened
[#51](https://github.com/goodhosts/hostsfile/issues/51). ddev calls the library once per hostname, roughly like this:

```go
hosts, err := goodhosts.NewCustomHosts(osHostsFilePath)
if err != nil {
	return err
}
err = hosts.Add(ip, name)
if err != nil {
	return err
}
hosts.HostsPerLine(8)
err = hosts.Flush()
```

Run that 15 times and the first line fills to 8 hosts. Then every new hostname ends up on its own line, newest first. Each test in the
suite called `Add` a bunch of times and then `HostsPerLine` once, and that was fine. Nothing tested the load, add, split, flush cycle
repeated.

I wrote a reproduction that does the same thing as ddev against a scratch file, reloading from disk for every add. I ran it against
three tagged versions of the library. This is the real output:

![hosts file output from hostsfile v0.1.4, v0.1.5 and v0.1.6](/img/2024/02/goodhosts-hosts-per-line.svg)

The cause was in `Add`. When the ip already existed it appended the new hostname to the *first* line with that ip:

```go
// add new host to the first one we find
hostsCopy := h.Lines[position[0]].Hosts
```

After the first split, the first `127.0.0.1` line is always full. So each add pushes it to 9 hosts. `HostsPerLine(8)` splits off the
overflow into a new line directly below it, and the previous overflow lines get pushed down. That's the reversed staircase in the v0.1.4
output. It also wasn't copying the slice before appending. That's asking for trouble once `HostsPerLine` has handed out subslices of the
same backing array to multiple lines.

[#52](https://github.com/goodhosts/hostsfile/pull/52) switched to the last line for that ip and copied the slice properly. It also added
a test that does 50 rounds of add plus split and checks the result is identical to adding all 50 and splitting once:

```go
// test if multiple calls to HostsPerLine doesn't create havoc and has identical output as calling it once
hosts = newHosts()
for i := 1; i <= 50; i++ {
	assert.Nil(t, hosts.Add("127.0.0.1", fmt.Sprintf("a%d.ddev.site", i)))
	hosts.HostsPerLine(8) // 8
}

hosts2 := newHosts()
for i := 1; i <= 50; i++ {
	assert.Nil(t, hosts2.Add("127.0.0.1", fmt.Sprintf("a%d.ddev.site", i)))
}
hosts2.HostsPerLine(8) // 8
assert.Equal(t, hosts.String(), hosts2.String())
```

That went out as v0.1.5, and it had a bug of its own:

```go
loc := position[len(position)-1] // last element
hostsCopy := make([]string, len(h.Lines[position[loc]].Hosts))
```

`position` is the list of line numbers for that ip, so `loc` is already a line number. `position[loc]` indexes the list *with* a line
number. In the test fixtures and in Randy's repro the `127.0.0.1` lines start at line 0 and run consecutively. So `position[n]` always
happened to equal `n` and every test passed. Put two comments at the top of the file, like a real hosts file has, and the first add
panics. That's the orange box in the image: `index out of range [2] with length 1`. [#53](https://github.com/goodhosts/hostsfile/pull/53)
changed `h.Lines[position[loc]]` to `h.Lines[loc]`. v0.1.6 was tagged under an hour after v0.1.5.

The lesson for me was the fixtures. Hosts files in the wild start with comments, and most of the lines are for other ips. A fixture
where the line you care about is line 0 hides a whole class of index bugs.

### Making the CLI testable

The CLI had the opposite problem: it barely had tests. Over the holidays I fixed that in
[#40](https://github.com/goodhosts/cli/pull/40) through [#43](https://github.com/goodhosts/cli/pull/43). The main obstacle was that the
whole `cli.App` was built inside `main()` as a local variable. The commands wrote straight to logrus, which `main` configured as a
`fmt.Print` replacement. You can't drive that from a test without running a binary.

So the app moved into the `cmd` package as an exported `App`, and `main` shrank to almost nothing:

```go
func main() {
	if err := run(os.Args); err != nil {
		logrus.Fatal(err)
	}
}

func run(args []string) error {
	cmd.Version(version, commit, date)
	return cmd.App.Run(args)
}
```

Errors come back out of `App.Run` where a test can assert on them, and only `main` turns them into `logrus.Fatal`. Output still goes
through logrus, which makes capturing it easy. Point logrus at a buffer and run the app with fake args.

```go
// Setup takes args slice and resets logrus and returns args to pass to App.Run
func setup(args ...string) ([]string, *bytes.Buffer) {
	out := &bytes.Buffer{}
	logrus.SetOutput(out)
	a := os.Args[0:1]
	a = append(a, args...)
	return a, out
}
```

With that, `list` tests are table-driven against hosts files written to disk, including the malformed-line marker:

```go
"malformed comment added to row": {
	input: `127.x.x.x localhost
127.x.x.x localhost # comment
`,
	expected: `127.x.x.x localhost # <<< Malformed!
127.x.x.x localhost # comment <<< Malformed!
`,
},
```

Writing those turned up a real bug right away. `list` loaded the hosts file in write mode, so it refused to run without elevated
privileges just to print the file. It's read-only now. The tests also have to know which OS they're on. The error for a missing file is
`no such file or directory` on Linux and macOS and `The system cannot find the file specified.` on Windows. The `--debug` and `--quiet`
flags got tests too. Debug switches logrus to its text formatter, and quiet discards output entirely. `version` is now testable because
the version string gets injected instead of read from package globals in `main`.

The CLI and library both build with mage now instead of a Makefile. Windows is a first-class target for this project. So `mage ci`
working the same in PowerShell and bash is worth more than make's familiarity. cli [v1.1.2](https://github.com/goodhosts/cli/releases/tag/v1.1.2)
went out last week with all of this and a codecov badge.

### What's next

The library's position indexes are the fragile part. Both bugs in this post were a line number pointing at the wrong place. The
next round is testing `Remove` and `HostsPerLine` against hosts files that look like real ones, with comments at the top and other ips
in between. If you use goodhosts in something, issues with a hosts file attached are the most useful bug reports I get.

### Links
- [goodhosts/hostsfile](https://github.com/goodhosts/hostsfile)
- [goodhosts/cli](https://github.com/goodhosts/cli)
- [goodhosts/vagrant](https://github.com/goodhosts/vagrant)
- [hostsfile#51](https://github.com/goodhosts/hostsfile/issues/51), [#52](https://github.com/goodhosts/hostsfile/pull/52), [#53](https://github.com/goodhosts/hostsfile/pull/53)
- [ddev](https://github.com/ddev/ddev)
