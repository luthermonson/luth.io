---
title: "Waiting on Proxmox: UPIDs, Tasks and WaitFor in Go"
description: "Most Proxmox VE API calls that change something hand you a UPID instead of a result. Here is how go-proxmox turns that string into something you can wait on."
date: 2022-03-20T16:42:00-07:00
tags: [ proxmox, go, go-proxmox, homelab ]
categories: [ go, homelab ]
images:
 - src: img/2022/03/proxmox-tasks-upid-wait-go-anatomy.svg
   alt: "Diagram breaking a Proxmox UPID string into node, pid, pstart, starttime, type, id and user"
   stretch: "none"
draft: false
---

If you start automating [Proxmox VE](https://www.proxmox.com/en/) through the API, the first thing that trips you up is
that almost nothing you ask it to do is finished when the HTTP call returns. Start a VM, clone a template, download an
ISO to storage, delete a disk... the response is a single string that looks like a typo and the actual work is happening
in a worker process on the node. If your next call depends on that work being done you have to go find out when it's done.

I've been building that part of [go-proxmox](https://github.com/luthermonson/go-proxmox) over the last couple of months
(the [xterm.js post](/blog/2022/02/recreating-proxmox-ve-xtermjs-in-go/) covered the shell side) and with the simpler
`WaitFor` that just landed I think the shape is settled enough to write up.

### The UPID

Here is what a download-url call on my home lab node `i7` gave back:

```
UPID:i7:00303F51:09D93CFE:61CCA568:download:8fd77349e9f6.iso:root@pam:
```

That is a Unique Process ID, and it's more useful than it looks. It's colon separated and every field means something:

![Anatomy of a Proxmox UPID](/img/2022/03/proxmox-tasks-upid-wait-go-anatomy.svg)

The middle three are hex. `00303F51` is the PID of the worker on the node (3161937), `09D93CFE` is the process start time
in clock ticks that Proxmox uses to tell PIDs apart after they get reused, and `61CCA568` is a unix timestamp (1640801640,
December 29th 2021 at 18:14 UTC, which is when I was testing this). The rest is plain text: the task type, an id (the VMID
for VM tasks, the file name for a download) and the user that started it.

The important bit is the node. Task status lives at `/nodes/{node}/tasks/{upid}/status`, so if you parse the node out of
the UPID you can poll it without the caller having to remember where the work was scheduled. That's the entire reason
`NewTask` exists:

```go
func NewTask(upid UPID, client *Client) *Task {
	if upid == "" {
		return nil
	}

	task := &Task{
		UPID:   upid,
		client: client,
	}

	sp := strings.Split(string(task.UPID), ":")
	if len(sp) == 0 || len(sp) < 7 {
		return task
	}

	task.Node = sp[1]
	task.Type = sp[5]
	task.ID = sp[6]
	task.User = sp[7]

	return task
}
```

Every method in the client that kicks off work now returns a `*Task` built this way instead of the raw string. VM start
is the simplest example:

```go
func (v *VirtualMachine) Start() (*Task, error) {
	var upid UPID
	if err := v.client.Post(fmt.Sprintf("/nodes/%s/qemu/%d/status/start", v.Node, v.VMID), nil, &upid); err != nil {
		return nil, err
	}

	return NewTask(upid, v.client), nil
}
```

The response body for these calls is `{"data": "UPID:..."}`, so unmarshalling straight into a `UPID` (which is just a
`string` type) works with no special handling.

### Ping, Wait and WaitFor

Once you have a `*Task`, `Ping` refreshes it from the status endpoint. The status payload has `status` (`running` or
`stopped`) and, once it's stopped, `exitstatus`, which is `OK` or the error text.

```go
func (t *Task) Ping() error {
	return t.client.Get(fmt.Sprintf("/nodes/%s/tasks/%s/status", t.Node, t.UPID), t)
}
```

Note it unmarshals right back into the task itself. The status response has the same fields the UPID encodes plus the
ones that change, so the struct just keeps filling in as you poll it.

`Wait` is a polling loop with an interval and a max. I went with a plain `select` on a `time.After` because I wanted
the timeout to be a real error you can check for, not a silent return:

```go
func (t *Task) Wait(interval, max time.Duration) error {
	// ping it quick to fill in all the details we need in case they're not there
	t.Ping()
	t.client.log.Debugf("waiting for %s, checking every %fs for %fs", t.UPID, interval.Seconds(), max.Seconds())

	timeout := time.After(max)
	for {
		select {
		case <-timeout:
			t.client.log.Debugf("timed out waiting for task %s for %fs", t.UPID, max.Seconds())
			return ErrTimeout
		default:
			if err := t.Ping(); err != nil {
				return err
			}

			if t.Status != TaskRunning {
				t.client.log.Debugf("task %s has completed with status %s", t.UPID, t.Status)
				return nil
			}
			t.client.log.Debugf("waiting on task %s sleeping for %fs", t.UPID, interval.Seconds())
		}
		time.Sleep(interval)
	}
}
```

After writing `task.Wait(1*time.Second, 10*time.Second)` twenty times across the integration tests I added the lazy
version. Most of the time you just want "poll every second, give up after N seconds":

```go
var DefaultWaitInterval = 1 * time.Second

func (t *Task) WaitFor(seconds int) error {
	return t.Wait(DefaultWaitInterval, time.Duration(seconds)*time.Second)
}
```

One thing to be aware of: `Wait` returning `nil` means the task is no longer *running*, not that it worked. A clone onto
storage that's full stops just as fast as one that succeeds. Check `task.ExitStatus` after the wait if you care, which
you probably do.

### Watching the log

Some tasks take long enough that you want to see what they're doing. A big ISO download is the obvious one, Proxmox
writes progress lines to the task log as it goes. The log endpoint is paged with `start` and `limit`, so `Watch` reads
what's there, hands it back on a channel and then tails it until the task stops:

```go
func tasktail(start int, watch chan string, task *Task) error {
	for {
		task.client.log.Debugf("tailing log for task %s", task.UPID)
		if err := task.Ping(); err != nil {
			return err
		}

		if task.Status != TaskRunning {
			task.client.log.Debugf("task %s is no longer running, closing down watcher", task.UPID)
			close(watch)
			return nil
		}

		logs, err := task.Log(start, 50)
		if err != nil {
			return err
		}
		for _, ln := range logs {
			watch <- ln
		}
		start = start + len(logs)
		time.Sleep(2 * time.Second)
	}
}
```

There is a small race at the front of this. If you call `Watch` immediately after starting a task the log can be empty
for a moment, so `Watch` retries the first read three times with a one second sleep before it gives up with
`no logs available`. The channel is closed when the task finishes so you can just `range` over it.

### Putting it together

The integration test fixture for VMs is the best example of all of this in one place. Download a tinycore ISO, wait for
it, create a VM with it attached, wait, configure it, wait:

```go
isoName := name + ".iso"
task, err := td.storage.DownloadURL("iso", isoName, tinycoreURL)
assert.Nil(t, err)
assert.Nil(t, task.Wait(time.Duration(5*time.Second), time.Duration(5*time.Minute)))

iso, err := td.storage.ISO(isoName)
assert.Nil(t, err)

cluster, err := client.Cluster()
require.NoError(t, err)

nextid, err := cluster.NextID()
require.NoError(t, err)

task, err = node.NewVirtualMachine(nextid, VirtualMachineOption{Name: "cdrom", Value: iso.VolID})
require.NoError(t, err)
require.NoError(t, task.Wait(1*time.Second, 10*time.Second))
```

And the task test itself downloads an Ubuntu ISO big enough to still be running after 30 seconds, waits on it in a
goroutine with a 30 second max so it has to hit the timeout, stops the task with a `DELETE` on the task path, and reads
the `Watch` channel on the main goroutine until it closes. One test, the whole lifecycle: start, poll, time out, tail,
cancel.

That's the plumbing. With tasks working, clone (which landed in February) and the rest of the VM lifecycle calls are
just "post, wrap the UPID, return the task", and the next thing I want on top of it is booting a VM straight from a qcow
URL without touching the node's shell.

### Links
- [go-proxmox](https://github.com/luthermonson/go-proxmox)
- [tasks.go as of this post](https://github.com/luthermonson/go-proxmox/blob/2f93c715a02de89f6d619fe50978bb93bf2d4f7e/tasks.go)
- [Proxmox VE API viewer, node tasks](https://pve.proxmox.com/pve-docs/api-viewer/index.html#/nodes/{node}/tasks)
- [Recreating Proxmox VE xterm.js in Go](/blog/2022/02/recreating-proxmox-ve-xtermjs-in-go/)
