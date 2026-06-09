# Hubspace / Afero API — Reverse-Engineering Notes

All knowledge here was obtained by reading the plugin source code and running live HTTP probes
against the API.  There are no official Hubspace or Afero docs.

---

## Authentication

### Endpoint
```
POST https://accounts.hubspaceconnect.com/auth/realms/thd/protocol/openid-connect/token
```

### Password grant (what the plugin uses)
```
Content-Type: application/x-www-form-urlencoded

grant_type=password
client_id=hubspace_android
username=<email>
password=<password>
```

### Token refresh
```
grant_type=refresh_token
client_id=hubspace_android
refresh_token=<refresh_token>
```

### Response (`TokenResponse`)
```json
{
  "access_token": "<JWT>",
  "expires_in": 3600,
  "refresh_token": "<string>",
  "refresh_expires_in": 86400,
  "token_type": "bearer"
}
```

The plugin persists the refresh token to `<homebridge-storage>/.hubspace-token.json` so it
survives restarts without re-entering credentials.

### Other known clients (all rejected password grant in testing)
| client_id | Result |
|---|---|
| `hubspace_android` | **Works** |
| `hubspace_ios` | `Client not allowed for direct access grants` — uses browser OAuth only |
| `hubspace_web` | Invalid client |
| `thd_customer` | Invalid client |

---

## Base URLs

| Purpose | URL |
|---|---|
| Auth / token | `https://accounts.hubspaceconnect.com/auth/realms/thd` |
| Device API (plugin uses) | `https://api2.afero.net/v1/` |
| Metadevice queries | Same URL but add header `Host: semantics2.afero.net` |
| `semantics.afero.net` (no 2) | Also resolves; returns richer fields (`locale`, `image`, `tag`) |
| `api3.afero.net` | Exists; TLS works; returns same data as api2 |
| `api.afero.io` | Exists but rejects our token (401) |
| `conclave-api.afero.io` | Exists but no matching routes (404) |
| `api.afero.net` (no number) | DNS not found |

All authenticated requests require:
```
Authorization: Bearer <access_token>
```

---

## Endpoints

### 1. Resolve account ID
```
GET /v1/users/me
```
Response (`AccountResponse`):
```json
{
  "accountAccess": [
    {
      "account": {
        "accountId": "<uuid>"
      }
    }
  ]
}
```
The plugin takes `accountAccess[0].account.accountId`.

---

### 2. List all metadevices (device discovery)
```
GET /v1/accounts/{accountId}/metadevices
Host: semantics2.afero.net
Authorization: Bearer <token>
```

Returns an array of `DeviceResponse` objects. The plugin filters to
`typeId === 'metadevice.device'` **and** `children.length === 0` (leaf devices only).

#### Full metadevice object structure
```json
{
  "id": "<uuid>",
  "createdTimestampMs": 1750115666365,
  "updatedTimestampMs": 1751316663684,
  "version": 1,
  "typeId": "metadevice.device",
  "friendlyName": "Watering Timer",
  "friendlyDescription": "",
  "locale": "en_US",
  "image": "",
  "tag": "",
  "deviceId": "<hex-device-id>",
  "children": [],
  "description": {
    "id": "<uuid>",
    "createdTimestampMs": 1729188087792,
    "updatedTimestampMs": 1729188087792,
    "version": 1,
    "device": {
      "defaultName": "Watering Timer",
      "manufacturerName": "Husky",
      "model": "TBD",
      "type": "device",
      "deviceClass": "water-timer",
      "profileId": "<uuid>"
    },
    "defaultImage": "2-spigot-watering-timer-icon",
    "hints": ["outdoor", "periodic-schedules-v2"],
    "descriptions": [],
    "functions": [ ... ]
  }
}
```

#### `typeId` values observed
| typeId | Meaning |
|---|---|
| `metadevice.device` | Actual device (leaf) |
| `metadevice.room` | Room grouping |
| `metadevice.group` | User-defined group |
| `metadevice.home` | Top-level home container |

#### `deviceClass` values and plugin mapping
| deviceClass | Plugin DeviceType |
|---|---|
| `light` | Light |
| `fan` | Fan |
| `power-outlet` | Outlet |
| `water-timer` | Sprinkler |

---

### 3. Functions (capabilities) inside a metadevice

Each metadevice's `description.functions` array describes what the device can do.
Every function maps to one or more **device attributes** (identified by numeric string `key`).

#### Function object structure
```json
{
  "id": "<uuid>",
  "functionClass": "toggle",
  "functionInstance": "spigot-1",
  "type": "category",
  "schedulable": true,
  "values": [
    {
      "name": "on",
      "deviceValues": [
        {
          "type": "attribute",
          "key": "2081",
          "value": "1"
        }
      ]
    },
    {
      "name": "off",
      "deviceValues": [
        {
          "type": "attribute",
          "key": "2081",
          "value": "0"
        }
      ]
    }
  ]
}
```

#### Known `functionClass` values
| functionClass | functionInstance | Description |
|---|---|---|
| `toggle` | (none / `spigot-1` / `spigot-2`) | Power on/off |
| `brightness` | — | Dimmer level 0–100 |
| `color-rgb` | — | RGB color |
| `color-temperature` | — | Color temp in Kelvin |
| `fan-speed` | — | Fan speed 0–100 |
| `max-on-time` | `spigot-1` / `spigot-2` | Timer max-on minutes (1–360) |
| `timer` | `spigot-1` / `spigot-2` | Countdown timer in minutes |
| `battery-level` | — | Battery % (0–100) |
| `schedule-pause` | `active` / `rain-delay` | Schedule pause state |

#### Known attribute keys (water timer)
| Attribute key | Meaning |
|---|---|
| `2081` | Spigot 1 toggle |
| `2082` | Spigot 2 toggle |
| `101` | Spigot 1 max-on-time (minutes) |
| `102` | Spigot 2 max-on-time (minutes) |
| `51` | Spigot 1 timer (minutes) |
| `52` | Spigot 2 timer (minutes) |
| `65079` | Battery level (%) |
| `49998` | Schedule-pause active flag |
| `49999` | Schedule-pause time array |

---

### 4. Read device state (attributes)
```
GET /v1/accounts/{accountId}/devices/{deviceId}?expansions=attributes,state
Authorization: Bearer <token>
```

`deviceId` is the **hex hardware ID** (`deviceId` field on the metadevice), not the metadevice UUID.

#### Response (`DeviceStatusResponse`)
```json
{
  "deviceId": "<hex>",
  "deviceState": {
    "available": true,
    "visible": true,
    "connected": false,
    "dirty": false
  },
  "attributes": [
    {
      "id": 2081,
      "value": "1",
      "updatedTimestamp": 1751316000000
    }
  ]
}
```

- `deviceState.available === false` → device is offline; plugin returns `undefined`.
- Attribute `id` is numeric but matched as a string (`id.toString() === attributeKey`).
- `updatedTimestamp` is a Unix ms timestamp — the plugin uses this to compute
  remaining valve run time.

---

### 5. Write a device attribute
```
POST /v1/accounts/{accountId}/devices/{deviceId}/actions
Authorization: Bearer <token>
Content-Type: application/json

{
  "type": "attribute_write",
  "attrId": "<attributeKey>",
  "data": "<encoded-value>"
}
```

#### Data encoding rules
| TypeScript type | Encoding | Example |
|---|---|---|
| `boolean` true | `"01"` | power on |
| `boolean` false | `"00"` | power off |
| `number` | hex bytes, little-endian reversed | `5` → `"05000000"` |
| `string` | passed through as-is | |

The `convertNumberToHexReverse` utility in `src/utils.ts` handles number encoding.

A `200` response means success.

---

### 6. Schedules
```
GET /v1/accounts/{accountId}/metadevices/{metadeviceId}/schedules
Authorization: Bearer <token>
```

Returns schedule objects. Each schedule event's `semanticValues` array references
`functionClass` + `functionInstance` (e.g. `timer` + `spigot-1`) with a `value` in minutes.
Schedules use the metadevice UUID (not deviceId).

Example schedule event:
```json
{
  "functionClass": "timer",
  "functionInstance": "spigot-1",
  "value": 20
}
```

---

## Data Flow (plugin initialization)

```
1. POST /protocol/openid-connect/token     → access_token, refresh_token
2. GET  /v1/users/me                       → accountId
3. GET  /v1/accounts/{id}/metadevices      → array of all metadevices
      filter: typeId=metadevice.device, children=[]
      map:    friendlyName → device.name
              deviceId     → device.deviceId  (hex, used for attribute R/W)
              id           → device.id        (UUID, used for schedules)
              deviceClass  → DeviceType enum
              functions[]  → DeviceFunction capabilities
4. Per accessory, on HomeKit GET:
      GET /v1/accounts/{id}/devices/{deviceId}?expansions=attributes,state
      → read attributes[key].value
5. Per accessory, on HomeKit SET:
      POST /v1/accounts/{id}/devices/{deviceId}/actions
      → write attribute
```

---

## What Was NOT Found

These were probed and confirmed absent or inaccessible:

| What we looked for | Result |
|---|---|
| Per-spigot friendly names ("Pansies", "Backyard Sprinkler") | **Not in any API response** — likely stored locally in the Hubspace app |
| `description.descriptions` array populated | Always `[]` — structurally the right place but empty |
| `PUT` to update descriptions | HTTP 200 accepted but GET still returns `[]` |
| `GET /v1/accounts/{id}/metadevices/{id}/names` | 404 |
| `GET /v1/accounts/{id}/metadevices/{id}/tags` | 404 |
| `GET /v1/accounts/{id}/names` | 404 |
| `GET /v1/descriptions/{descriptionId}` | 404 |
| `GET /v1/profiles/{profileId}` | 404 |
| `GET /v1/accounts/{id}/devices/{id}/functions` | 404 |
| `GET /v1/accounts/{id}/devices/{id}/zones` | 404 |
| `GET ?expansions=tags` on metadevices list | 500 Internal Server Error |
| GraphQL (`/graphql` on api2 or semantics2) | 404 / 403 |
| `api2.afero.net` without `Host: semantics2` | 404 for metadevice routes |
| `api.afero.io` (different TLD) | 401 — our token not accepted |

---

## Firebase

The Hubspace iOS app connects to:
```
hubspace-9f606.firebaseio.com  (WebSocket/TLS)
```
Firebase API key: `AIzaSyCd-lNsr29RBic-DIdS97-aeCYZOwmtnh8`  
Project number: `1006144984876`  
Bundle: `io.afero.partner.hubspace`

The iOS app uses SSL/certificate pinning on this connection — mitmproxy intercepts the TLS
handshake but the client disconnects immediately because it doesn't trust the proxy CA.
The plugin does **not** use Firebase; it's unknown what the app uses it for (possibly
real-time push for device state updates).

---

## Plugin Model Mapping

```
Afero API field          → Plugin model field
──────────────────────────────────────────────
metadevice.id            → device.id       (metadevice UUID)
metadevice.deviceId      → device.deviceId (hex hardware ID — used for attribute R/W)
hap.uuid.generate(id)    → device.uuid     (Homebridge accessory UUID)
metadevice.friendlyName  → device.name
description.device.manufacturerName → device.manufacturer
description.device.model → device.model[]
description.functions[]  → device.functions[]  (filtered to known DeviceFunctions)
```

The `device.uuid` (used to match cached accessories) is derived by running the Afero
metadevice UUID through `hap.uuid.generate()`, not by using any field from the API directly.
