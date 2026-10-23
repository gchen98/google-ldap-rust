# google-ldap-rust

A small [Rocket](https://rocket.rs) service that keeps a shared LDAP address
book in two-way sync with the Google Contacts of a group of users. When you call
`GET /sync`, it reads every contact from LDAP and from each configured Google
account, and then reconciles the two.

It talks to two backends, both expected to run locally:

| Backend | How it is reached | Purpose |
|---|---|---|
| **LDAP server** | `ldap://<host>:389`, settings in `ldap_config.toml` | Shared address book (`objectClass=mozillaAbPersonAlpha` entries under `ou=Public,dc=pioneerind,dc=com`) |
| **Go People API service** | `http://127.0.0.1:8340` | A separate service that wraps the Google People API for each user and exposes `ListContacts`, `AddContacts`, `UpdateContacts` and `DeleteContacts` |

## How syncing works

Each LDAP entry's `ou` attribute holds a list of "subscriptions" (for example
`garyc`, `family`). Each Google user in `google_config.toml` has a `filter`
list. A user receives every LDAP contact whose `ou` list shares at least one
value with that user's `filter`.

Contacts are matched across systems by **full name** (LDAP `cn` and the Google
display name).

`GET /sync` runs these phases in order, refreshing its snapshot of both sides
between phases:

1. **Purge duplicates.** If one user has several Google contacts with the same
   name, it keeps the most recently modified one and deletes the rest.
2. **Propagate edits.** For every contact modified since the last run (the time
   stored in `timestamp.txt`), the newest copy wins:
   - If LDAP is newest, the change is pushed to every subscribed user's Google
     contacts.
   - If one user's Google contact is newest, the change is written to LDAP (the
     entry is deleted and re-added) and pushed to the other subscribed users.

   Afterwards it writes the current time to `timestamp.txt`.
3. **Import new Google contacts.** Any Google contact that does not have the
   user-defined field marked `sync` is added to LDAP with `ou` set to that
   user's `subscription`. The Google contact is then marked as synced.
4. **Purge orphans.** For each user:
   - A contact that is in LDAP (and matches the user's filter) but missing from
     Google is added to Google.
   - A contact that is in Google but no longer in the user's LDAP set is
     deleted from Google.

## Configuration

The service reads its config files from its **working directory**. The real
files are git-ignored, and checked-in templates are provided:

```sh
cp ldap_config.toml.example   ldap_config.toml
cp google_config.toml.example google_config.toml
$EDITOR ldap_config.toml google_config.toml
```

### `ldap_config.toml`

```toml
host = "ldap.example.com"            # connects to ldap://<host>:389
admin_dn = "cn=admin,dc=example,dc=com"
password = "changeme"
```

### `google_config.toml`

```toml
users = [
    { username = "alice@example.com", subscription = "alice", filter = ["alice", "family"] },
    { username = "bob@example.com",   subscription = "bob",   filter = ["bob", "family", "work"] },
]
```

- `username`: the Google account, which is passed to the Go service.
- `subscription`: the `ou` value given to new contacts that this user creates in
  Google.
- `filter`: the `ou` values whose LDAP contacts this user should see.

### `timestamp.txt`

This file holds the time of the last successful edit-propagation pass as an
RFC 3339 timestamp. **It must exist before the first sync**, or the sync will
panic. To create it:

```sh
date -u +%Y-%m-%dT%H:%M:%S+00:00 > timestamp.txt
```

### Values set in code

These values are not configurable yet. To change them, edit the source:

- The Go People API URL, `http://127.0.0.1:8340`, in `src/datasource/people_api.rs`.
- The LDAP port (`389`) and the base DN `ou=Public,dc=pioneerind,dc=com`, in
  `src/datasource/ldap.rs`.

## Building and running

This project needs a recent stable Rust toolchain because it uses edition 2024.

```sh
cargo build --release
ROCKET_PORT=8081 ./target/release/google_ldap_rs
```

The default Rocket port is 8000. You can set `ROCKET_ADDRESS`, `ROCKET_PORT` and
other options through the environment or a `Rocket.toml` file.

### Endpoints

| Route | Description |
|---|---|
| `GET /` | Health check that returns `Hello, world!` |
| `GET /sync` | Runs a full sync. Progress and errors are written to stdout, and the HTTP response is empty either way. |

To trigger a sync, for example from cron:

```sh
curl -s http://localhost:8081/sync
```

### Running as a systemd service

`config/google_ldap_rs.service` is a sample unit file. Edit `User`,
`WorkingDirectory` and `ExecStart` to match your install. The config files and
`timestamp.txt` must be in `WorkingDirectory`. Then install the unit:

```sh
sudo cp config/google_ldap_rs.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now google_ldap_rs
journalctl -u google_ldap_rs -f
```

## Tests

The tests in `src/` are integration tests. They need a live LDAP server, the Go
service and real config files, and `syncer::tests::run_sync_contacts` runs a
**real sync**. Run them only against data you are willing to change:

```sh
cargo test -- --nocapture
```

## Project layout

```
src/
  main.rs                    Rocket routes (/ and /sync)
  syncer.rs                  Sync phases and orchestration
  datasource/
    ldap.rs                  LDAP search / add / delete
    people_api.rs            HTTP client for the Go People API service
    contact.rs               Google Person and LDAP Person models, field mapping
    sync_config.rs           Config structs for the two TOML files
config/google_ldap_rs.service  Sample systemd unit
*.toml.example               Config templates
```
