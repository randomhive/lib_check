# lib_check

Check_MK piggyback plugin for IBM TS4500 tape library monitoring via the REST API.

Related TSM monitoring: [`tsm_check`](https://github.com/randomhive/tsm_check) (Spectrum Protect / TSM local checks).

REST API reference: [IBM tape-automation `roecurl.md`](https://github.com/IBM/tape-automation/blob/master/rest-over-ethernet/roecurl.md)

## Prerequisites

`lib_check` uses **HTTPS only** (`https://HOST/web/api/v1/...`). REST login and all API calls fail if the library does not accept HTTPS.

On the TS4500, enable **Secure Communications** in the management GUI (IMC / web UI). With secure communications active, the library serves the REST API over HTTPS. If it is disabled, HTTP may still respond but redirects or rejects API traffic — `lib_check` will not follow HTTP or downgrade to cleartext.

Use `SSL_SKIP=1` in `lib_check.cfg` when the library uses a self-signed certificate (typical with secure communications enabled but no corporate CA). Use `SSL_SKIP=0` only when the management certificate is signed by a trusted CA.

Requires `python3` and `curl` on the Check_MK agent host.

## Configuration

Copy `lib_check.cfg.example` to `lib_check.cfg` — one library per line:

```
NAME:FQDN:HOST:ALT_HOSTS:CREDENTIALS:SSL_SKIP
```

| Field | Description |
|-------|-------------|
| `NAME` | Short identifier (logging, temp files) |
| `FQDN` | Library hostname in Check_MK (piggyback target) |
| `HOST` | Primary management FQDN or IP for HTTPS REST (active LCC) |
| `ALT_HOSTS` | Optional comma-separated backup management hosts (other LCCs); empty if none |
| `CREDENTIALS` | Path to credentials file (`USER` / `PASSWORD`) |
| `SSL_SKIP` | `1` = skip TLS verification (`curl -k`), `0` = verify certificate |

Connection tries `HOST` first, then each entry in `ALT_HOSTS` until login succeeds.

Copy `lib_check.credentials.example` to a credentials file (e.g. `lib_check.credentials`) and restrict permissions (`chmod 600`).

## Usage

Install `lib_check` and `lib_check.cfg` on the Check_MK agent host. Output is piggyback data for each library `FQDN`:

```
<<<<library.example.org>>>>
0 Library ...
0 Accessor_Aa ...
0 Drive_F1C4R2 ...
<<<<>>>>
```

Run with `DEBUG=1` to log resolved `curl` commands to stderr.

On login or credential errors, a single `Connection` service is emitted for that library. Individual check failures (HTTP/curl errors) emit one error line for the affected service and continue with the remaining checks.

## Checks

All checks run for every configured library after a successful login. Per-resource services use API field names in the check output where possible; null or missing JSON fields are omitted rather than shown as placeholders.

| Service | REST endpoint | Description |
|---------|---------------|-------------|
| `Connection` | `POST /login` | Emitted only on credential or login failure |
| `Library` | `GET /library` | Overall library status, cartridge access, licensed capacity utilization (WARN ≥90%, CRIT ≥98%) |
| `Accessor_*` | `GET /accessors` | One service per accessor (e.g. `Accessor_Aa`). State from accessor `state`, `driveAccess`, `cartridgeAccess` |
| `Cleaning` | `GET /cleaningCartridges` | Cleaning cartridge count and total remaining cleans (WARN ≤100, CRIT ≤50) |
| `Slots` | `GET /slots` | Cartridge count vs. tier capacity across all slots |
| `Drive_*` | `GET /drives` | One service per drive (e.g. `Drive_F1C2R2`). Drive fields from `/drives`; matching FC port fields from `/fcPorts` appended when available |
| `FCPorts` | `GET /fcPorts` | Summary: total FC ports and count in bad state |
| `Events` | `GET /events` | Active (non-silenced) warnings and errors |
| `EthernetPort_*` | `GET /ethernetPorts` | One service per port with a configured IPv4 address (not `disabled`). Optional `state` when reported by the API |
| `EthernetPorts` | `GET /ethernetPorts` | Summary of active Ethernet ports |
| `NodeCard_*` | `GET /nodeCards` | One service per node card (e.g. `NodeCard_LCC_F1`, `NodeCard_MDA_Aa`). LCC cards include `reportingLCC`, `primaryLCC`, `lastRestart`, `cfVendor` when present |
| `NodeCards` | `GET /nodeCards` | Summary: total node cards and count in bad state |
| `IOStation_*` | `GET /ioStations` | One service per I/O station (e.g. `IOStation_F1IOu`). Fields from station, door, and magazine objects only |
| `IOstation` | `GET /ioStations` | Total occupied magazine slots (`contentsVolser` non-null); WARN ≥200, CRIT ≥244 |
| `UnassignedCartridges` | `GET /dataCartridges` | Data cartridges with no logical library assignment |
| `LogicalLib_*` | `GET /logicalLibraries` | One service per logical library (e.g. `LogicalLib_dp-backup`). Uses API fields `cartridges`, `virtualSlots`, `drives`, etc.; slot utilization WARN ≥90%, CRIT ≥98% |

### State mapping (selected)

| Component | OK | WARN | CRIT |
|-----------|----|------|------|
| Accessor | `onlineActive`, `onlineStandby` | `scannerFailed`, `gripper*Failed`, `calibrating`, … | `bothGrippersFailed`, `noMotorPower`, `inServiceMode`, `unknown` |
| Drive | `online`, `cleaning` | `initializing`, `restarting`, `unreachable`, … | `inServiceMode`, `unknown` |
| FC port | `communicationEstablished` | (other) | `failed`, `unreachable` |
| Node card | `online` | `restarting`, `unknown` | `inServiceMode`, `unreachable`, `noEthernet`, `noCAN` |
| I/O station | `normal`, `closedNoMagazine` | open door | other / failed states |

## Files

| File | Purpose |
|------|---------|
| `lib_check` | Main script (shell + embedded Python) |
| `lib_check.cfg` | Local library connection definitions (not in repo; copy from example) |
| `lib_check.cfg.example` | Config file template |
| `lib_check.credentials.example` | Credentials file template |
