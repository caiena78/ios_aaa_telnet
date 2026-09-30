Telnet-transport variant of `ios_aaa` - configures local user, TACACS+ servers/group, and
AAA authentication/authorization/accounting on a Cisco IOS/IOS-XE device, and registers the
device in Cisco ISE (refreshing the TACACS shared secret if it already exists) - for devices
that only have telnet enabled and can't reach `ios_aaa` over SSH/`network_cli`.

**Use `ios_aaa` instead whenever the device has SSH.** This role exists only for devices you
genuinely cannot reach any other way. It is functionally weaker than `ios_aaa` in ways that
matter for a security-relevant config change:

## Real limitations (not implementation details - actual gaps)

- **No config diffing / idempotency.** `ios_config` compares desired lines against the running
  config and only pushes what's missing. `ansible.netcommon.telnet` has no such concept - this
  role pushes its full command list every run, whether or not the device already matches.
- **No built-in error detection.** The telnet module never fails on a bad command; it just
  returns whatever text came back. This role does a best-effort scan of the session output for
  common IOS error markers (`% Invalid input`, `% Incomplete command`, `% Bad secret`, `command
  authorization failed`) and fails the task if it sees one, but this is a substitute for real
  error handling, not equivalent to it.
- **Requires a username prompt to already exist on the VTY line** (e.g. `login local`). The
  `ansible.netcommon.telnet` module unconditionally waits for `telnet_login_prompt` before
  sending anything - if the device only has a bare password prompt on VTY (no username), this
  role will time out. That's a real gap for a genuinely virgin device: if the reason you're
  running this role is to *create* the first local admin account, and the device currently has
  no username-based login at all, this role cannot bootstrap that over telnet. You'd need to set
  up local-login manually (console access) first.
- **Local user/enable secret always use the old syntax.** The IOS releases this role targets
  (e.g. VG224 voice gateways) reject `algorithm-type scrypt` outright, so the local user is set
  with `username <user> privilege 15 password <pwd>` - stored as Type 7 when
  `service password-encryption` is configured on the device, cleartext otherwise - and the
  enable secret with `enable secret <pwd>`. There is no Type 9/scrypt path and no toggle.
- **No automatic legacy-syntax fallback for TACACS.** `ios_aaa` uses a `block`/`rescue` over
  `network_cli` to detect old IOS releases that don't support `tacacs server <name>` syntax and
  falls back automatically. This role can't detect that reliably over raw telnet, so you must
  set `ios_telnet_use_legacy_tacacs_syntax: true` explicitly per host/group if needed.
- **One telnet session per logical stage**, not per-task `ios_config` calls. The push is split
  into separate tasks - local user/enable secret, AAA new-model, TACACS servers, TACACS group,
  AAA authentication/authorization/accounting, and console/VTY lines + verify + save - so each
  stage gets its own task name and its own pass/fail. `ansible.netcommon.telnet` has no
  persistent-connection support, so each stage re-logs in and re-enters `configure terminal`;
  that costs a handful of extra logins per device in exchange for being able to see which stage
  failed instead of reading one wall of combined stdout. The `ip tacacs source-interface` /
  `ip vrf forwarding` handling
  under the group is likewise consolidated into one clean pair of commands, replacing three
  overlapping, hard-to-follow tasks in `ios_aaa` that were doing the same thing.
- **Untested against a live device.** This was built and validated by rendering the Jinja
  command-building logic against fabricated `show running-config` text (interface/VRF
  detection, stale-server removal, VTY range detection all produced correct output), but the
  actual telnet login/enable/command sequence has not been run against a real Cisco IOS/IOS-XE
  telnet session. Validate in a maintenance window with `telnet_debug: true` before trusting
  this against production devices - a bad AAA/TACACS push can lock you out of a device with no
  SSH fallback to recover it.

## What's identical to ios_aaa

The Cisco ISE device pre-check/registration block (find by IP, find by name, refresh the TACACS
shared secret on an existing device, create if not found) is copied verbatim - it's pure REST
calls via `uri` delegated to `localhost` and doesn't depend on how the device itself is reached.

## Key variables

See `defaults/main.yml`. In addition to the same vars as `ios_aaa` (`LOCAL_USER`, `LOCAL_PWD`,
`LOCAL_ENABLE`, `tacacs`, `tacacs_group_name`, `remove_undefined_servers`, `ise_*`), this role
adds:

- `telnet_port`, `telnet_login_prompt`, `telnet_password_prompt`, `telnet_timeout`,
  `telnet_pause`, `telnet_prompts` - transport settings for `ansible.netcommon.telnet`.
- `telnet_debug` - when true, prints the detected management interface/VRF, the full command
  list before it's sent, and the raw session output after.
- `ios_telnet_use_legacy_tacacs_syntax` - manual override for old IOS releases that don't
  support `tacacs server <name>` syntax (see above).

This role authenticates to the device using the same `ansible_user` / `ansible_password` as
`ios_aaa`, and enters enable mode using `ansible_become_password` - same variables, just used
directly instead of through `network_cli`'s automatic become handling.
