---
title: "Staying Logged In to Proxmox: Ticket Renewal, Expiry and Where API Tokens Stop"
description: "Proxmox tickets die after two hours. How go-proxmox went from never re-authenticating to renewing tickets in place, plus the one thing API tokens can't do."
date: 2026-05-31T20:24:00-07:00
tags: [ proxmox, go, go-proxmox, authentication, websockets ]
categories: [ go, homelab ]
images:
 - src: img/2026/05/staying-logged-in-to-proxmox-timeline.svg
   alt: "Timeline of a Proxmox ticket: valid for two hours server side, considered stale by the client after one"
   stretch: "none"
draft: false
---

Back in 2023 I wrote up [how go-proxmox logs in](/blog/2023/08/proxmox-api-auth-tickets-tokens-go/) and ended it with a
shrug: once a session existed the client never logged in again, so a ticket-based client died at the two hour mark and
my advice was "use an API token for anything long running". It took until this month to actually fix that properly, and
it took three PRs from two of us to get there. Here's the whole arc, plus the one place where an API token
isn't an option at all.

![Ticket timeline](/img/2026/05/staying-logged-in-to-proxmox-timeline.svg)

### Step one: stop racing yourself

The original 401 handler looked like this:

```go
if c.credentials != nil && c.session == nil {
	// credentials passed but no session started, try a login and retry the request
	if _, err = c.Ticket(ctx, c.credentials); err != nil {
		return err
	}
	return c.Req(ctx, method, path, data, v)
}
return ErrNotAuthorized
```

Two problems. The obvious one is `c.session == nil`, which means "log in once, ever". The less obvious one is that
`c.session` is read and written by whatever goroutine got the 401, and a client is usually shared. Fire ten requests at a
fresh client from ten goroutines and you get ten logins writing the same field.
[#223](https://github.com/luthermonson/go-proxmox/pull/223) from Serge Logvinov put a mutex around it and moved the login
into `CreateSession`, which returns `ErrSessionExists` if somebody else already did the work.

### Step two: know when it's stale

That still never re-logged in. [#239](https://github.com/luthermonson/go-proxmox/pull/239), also from Serge, added an
expiry so `CreateSession` treats an old session as gone:

```go
if c.session != nil && time.Now().Before(c.sessionExpiresAt) {
	return ErrSessionExists
}
// ...
c.sessionExpiresAt = time.Now().Add(time.Hour)
```

Proxmox tickets are good for two hours (`$ticket_lifetime = 3600 * 2` in `PVE::AccessControl`). The client calls them
stale after one, which leaves plenty of slack for clock skew between you and the cluster. From here a credential-based
client survives: the ticket dies at two hours, the next request gets a 401, `CreateSession` sees a stale session and
logs in again with the stored password.

### Step three: renew without the password

That fixed username and password logins. It didn't fix two cases people had been asking about since 2024:
[#181](https://github.com/luthermonson/go-proxmox/issues/181) wanted to renew with the existing ticket instead of keeping
the password around, and [#196](https://github.com/luthermonson/go-proxmox/issues/196) was throwing away the whole
`*Client` on auth failure and building a new one, which orphaned every cached `*VirtualMachine` and `*Node` that held a
pointer to the old client. And if your login used a TOTP code, re-sending the stored credentials doesn't work at all
because the code is single use.

Proxmox has a documented answer for this: `POST /access/ticket` with the *current ticket* as the password. On the server
side `create_ticket` runs `verify_ticket` on the password field first and only falls back to a real password check if
that fails. [#267](https://github.com/luthermonson/go-proxmox/pull/267) wires that up:

```go
func (c *Client) RefreshTicket(ctx context.Context) error {
	c.sessionMux.Lock()
	defer c.sessionMux.Unlock()
	return c.refreshTicketLocked(ctx)
}

func (c *Client) refreshTicketLocked(ctx context.Context) error {
	if c.session == nil || c.session.Ticket == "" {
		return ErrNoSession
	}
	if _, err := c.Ticket(ctx, &Credentials{
		Username: c.session.Username,
		Password: c.session.Ticket,
	}); err != nil {
		return err
	}
	c.sessionExpiresAt = time.Now().Add(time.Hour)
	return nil
}
```

`CreateSession` now tries a renewal first when it has a session and only falls back to the full credential login if that
fails. The session is replaced on the existing `*Client`, so every resource handle hanging off it picks up the new ticket
without doing anything. That's the #196 fix: stop rebuilding clients.

### Renewal is on you, though

Here is the part that I think is easy to get wrong, and I had to read the Perl to be sure. Renewal only works with a
ticket that is still valid. `verify_ticket` checks the signature *and* the age against the same two hour lifetime, so an
expired ticket is just a wrong password. The client only reacts to a 401, and Proxmox doesn't send a 401 until the ticket
is already past two hours. So the automatic path in `CreateSession` is mostly going to fall through to the password
login, and for a client that has no password to fall back to (an OTP login, or one built `WithSession` from somebody
else's ticket) that's the end of the session.

If you're in that situation, renew on a timer and stay inside the window:

```go
go func() {
	t := time.NewTicker(45 * time.Minute)
	defer t.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-t.C:
			if err := client.RefreshTicket(ctx); err != nil {
				log.Printf("proxmox ticket refresh: %v", err)
			}
		}
	}
}()
```

One more wart I know about and haven't fixed yet: when two goroutines hit a 401 at the same moment, the first one renews
and the second one waits on the mutex, finds a fresh session, gets `ErrSessionExists`, and `Req` turns that into
`ErrNotAuthorized` instead of retrying with the new ticket. It's a narrow window, and the fix is a retry rather than an
error, but it's there.

### The cost of logging in lazily

While I was in here the v0.7 work added a few auth options worth knowing about. `WithOTP` stashes a single TOTP code and
zeroes it after the first login so a later re-login doesn't resend a dead code. `WithDefaultRealm("pam")` turns `root`
into `root@pam` so you stop getting 401s for forgetting the realm.

`WithEagerAuth` is the one with a story. The client has always logged in lazily: the first request goes out without auth,
gets a 401, logs in, and retries. What I didn't know in 2023 is that `pveproxy` deliberately slows down every 401:

```perl
if ($res->{status} == HTTP_UNAUTHORIZED) {
    # always delay unauthorized calls by 3 seconds
    $delay = 3 - tv_interval($reqstate->{starttime});
```

So every credential-based client has been paying three seconds on its first call since the beginning.
`WithEagerAuth` logs in inside `NewClient` instead so that cost moves to startup. It swallows the error (options can't
return one) and the lazy path still kicks in on the first request if it failed, so if you need to know at startup that
your password is wrong, call `CreateSession` yourself.

### Where API tokens stop

API tokens are still the right answer for most automation: one header, no CSRF token, no two hour clock. Except for
terminals. [#221](https://github.com/luthermonson/go-proxmox/issues/221) reported that opening a container terminal
failed with an opaque `failed to create terminal websocket: unexpected EOF`, and it turned out to be three separate bugs
in the websocket handshake ([#259](https://github.com/luthermonson/go-proxmox/pull/259)).

Two of them were mine, from the [2022 xterm.js post](/blog/2022/02/recreating-proxmox-ve-xtermjs-in-go/). The UI opens
the socket with `new WebSocket(url, "binary")` and the vncwebsocket handler hangs up on clients that don't ask for the
`binary` subprotocol, so the dialer now sends `Subprotocols: []string{"binary"}`. And the resize message is
`1:<cols>:<rows>:`, width first. I wrote it height first in 2022 (it's right there in that post's resize code) and it
took four years for anyone, me included, to notice their terminal coming out transposed.

The third is a Proxmox limitation on older releases. Through PVE 8 and early PVE 9, `termproxy` verified the VNC ticket
through `/access/ticket`, where `verify_username` rejects the `!tokenname` suffix a token user carries. The ticket's
HMAC is bound to that same name, so stripping the suffix on the client side doesn't work either. Newer PVE 9 ships a
`/access/vncticket` endpoint that accepts a token id (node shells since pve-manager 9.0.13 in November 2025, guest
terminals since this spring's qemu-server and pve-container updates), but go-proxmox doesn't detect the server version
yet, so for now the client fails fast with a sentinel instead of the mystery EOF:

```go
// ErrAPITokenWebSocketUnsupported is returned by TermWebSocket and VNCWebSocket
var ErrAPITokenWebSocketUnsupported = errors.New("proxmox does not accept API tokens for vncwebsocket; use WithCredentials or WithSession")
```

So if your tool targets PVE 8 and needs both automation and a shell, it needs a ticket for the shell, and now the ticket can stay alive.

### Links
- [#223 session race](https://github.com/luthermonson/go-proxmox/pull/223), [#239 session expiry](https://github.com/luthermonson/go-proxmox/pull/239), [#267 ticket renewal](https://github.com/luthermonson/go-proxmox/pull/267)
- [#259 websocket fixes and the API token limit](https://github.com/luthermonson/go-proxmox/pull/259)
- [#327 TLS, timeout and auth options](https://github.com/luthermonson/go-proxmox/pull/327)
- [pve-access-control, PVE/AccessControl.pm](https://git.proxmox.com/?p=pve-access-control.git;a=blob;f=src/PVE/AccessControl.pm)
- [pve-http-server, PVE/APIServer/AnyEvent.pm](https://git.proxmox.com/?p=pve-http-server.git;a=blob;f=src/PVE/APIServer/AnyEvent.pm)
- [Logging In to Proxmox From Go (2023)](/blog/2023/08/proxmox-api-auth-tickets-tokens-go/)
