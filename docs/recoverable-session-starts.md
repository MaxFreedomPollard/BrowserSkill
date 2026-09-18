# Recoverable browser session starts

The DSH plugin used to register a browser session only after receiving a successful CLI result. Cancellation, a lost reply, or a host crash could leave an Agent Window that the plugin could neither identify nor close (issue #245). A second leak occurred when initial navigation failed and a failed compensating stop caused the plugin to discard ownership.

## Lifecycle contract

Managed starts now have a caller-generated request token before any browser side effect. The plugin writes the token and DSH ownership lineage to an atomic, synced journal first. It prepares the request on the daemon, then starts it. The separate `session.start_tracked` RPC fails closed on an older daemon. A start without a prepared request also fails: a delayed CLI cannot recreate a cancelled window after a daemon restart.

The CLI flow is:

```text
bsk session request <token> --prepare --json
bsk session start --request-id <token> [window options] --json
bsk session request <token> --claim --json
bsk session request <token> --json
bsk session request <token> --cancel --json
```

A token is `<expiry-unix-ms>:<UUID>`, generated with a random UUID. Admission deadlines must be in the next ten minutes; the plugin uses five minutes. Treat tokens as ownership handles: they are not labels, task names, or short session IDs. Ordinary `session start` remains unchanged and does not require these calls. Managed starts do not sync unrelated CLI harness skills; the plugin carries its own instructions.

Preparation creates no browser window. Start transitions from prepared to starting, then ready. Claim happens after the plugin durably records the returned session and finishes initial navigation/emulation; it changes ready to active. A repeated start with the same token and parameters returns the existing result instead of opening another window; different parameters are rejected. Expired tokens cannot start again.

Cancellation is monotonic, including cancellation **before** prepare/start. An early cancellation leaves a terminal tombstone. Cancelling a starting operation preserves its original extension waiter until creation settles; a premature `session not found` cannot prove that a delayed `chrome.windows.create` has finished. These reservations are excluded from idle cleanup and ordinary stop until creation settles. The caller's wait remains bounded, while daemon-owned reconciliation may continue.

Cleanup failures retain the exact session/window identity and can be retried. The extension also retains a window whose startup compensation failed, so a later stop can actually close it. Cleanup uses the request's original session identity and verifies that a reused short session ID does not belong to another window.

The existing daemon reaper cancels unclaimed requests after their admission deadline (normally within the next 30-second tick), retries failed cleanup, and removes expired terminal tombstones. Claimed sessions retain the existing session idle policy. Browser disconnection still follows the existing session teardown behavior. An unresponsive browser can delay confirmed cleanup; that state remains owned and visible as pending rather than being reported as closed.

## Plugin recovery

The journal defaults to `$BSK_HOME/dsh-starts/<scope>` (or `~/.bsk/dsh-starts/<scope>`). The scope includes the working directory and configured CLI path. `sessionStateDirectory` can assign a dedicated durable directory to a host/profile. Each live plugin instance owns a separate process/UUID directory; recovery never takes another live instance's records. A reused PID is treated conservatively as live; daemon leases and idle cleanup still apply.

On reload/startup, abandoned journal directories are claimed by atomic rename and their requests are cancelled. Interrupted recovery remains recoverable, including directories moved just before a crash. Failed cleanup records remain on disk and are retried by the plugin, list/stop actions, and the daemon. Unload and conversation archival cancel pending starts as well as completed sessions. New starts are refused during unload or from an archived conversation.

The model-facing list retains its owned-session view and adds `pendingCleanup`. It never discovers ownership by diffing the daemon's global session list. A stop with no current session can retry pending startup cleanup. Explicit and overlay stops use the same request handle for managed sessions.

CLI and daemon must both support the new request protocol. Preparation detects unsupported versions before opening a window. Update the extension as well to obtain retryable startup-compensation failures. No persistent daemon service, external database, or new package dependency is introduced.

## Regression coverage

- Plugin: lost/aborted/timed-out successful replies; failed navigation and failed cleanup; archive/unload during an in-flight start; write-ahead persistence failure; pending capacity limits; restart recovery with another live instance; unsupported preparation; stop without a current session.
- Daemon: cancellation before prepare/start; expired tokens and tombstone collection; restart admission; unclaimed vs. claimed leases; reused short IDs; concurrent duplicate requests; delayed creation beyond the old cancellation grace; retryable close failures; foreign session isolation.
- Integration: actual CLI child killed after dispatch and before its successful response, followed by recovery through a new CLI process. IPC and WebSocket tests use a real daemon and a controlled extension peer, without operating the user's browser.
- Extension: failed startup compensation retains the window until a subsequent stop succeeds.

These checks complement the existing session, cancellation, queue, plugin, and extension tests. They do not claim a Windows/DSH/Chrome GUI end-to-end run.
