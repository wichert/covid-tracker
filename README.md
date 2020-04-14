# Covid tracker backend

## Data

### Observation data

Observation data is stored as a collection of files: one for every device, as identified
by an authorisation code. Each file is a CSV file with the following fields:

| Field | Type | Description |
| -- | -- | -- |
| processed_at | integer | UNIX timestamp this record was processed and stored |
| day_number | integer | Day Number the Daily Tracing Key was used |
| key | string | BASE64 encoded Daily Tracing Key |

The filename is `<authorisation code>.csv`. This allows for very efficient deletion of
expired authorisation codes.

## Authorisation codes

Valid authorisation codes are stored as a collection of files, one for each authorisation
code. The filesystem modification timestamp is used as code creation timestamp. This
approach is chosen for two reasons:

1. The Go runtime does not expose the creation timestamp in
   [FileInfo](https://golang.org/pkg/os/#FileInfo), so we use the modification timestamp
   as a proxy.
2. This allows is to efficiently expire codes without having to open and read many files.

In order to allow validation the creation timestamp is also stored inside the file as a
UNIX timestamp in text format.

## Deployment

The first step is to prepare some directories where data will be stored

| Directory | Description |
| -- | -- |
| `<data-dir>/records` | Storage for daily tracing key data |
| `<data-dir>/tokens` | Storage for daily tracing authorisation tokens |

The default data directory is `/var/lib/covid-tracker`, but you can specify a different
location with the `-data` option to all commands.

### Health API server

The health API server must be deployed as part of a chain of HTTP servers:

```plain
+--------------+      +--------------+      +---------------+
|              |      |              |      |               |
|  SIAM proxy  +----->+  Health API  | +--> | Front-end app |
|              |      |  server      |      |               |
+--------------+      +--------------+      +---------------+
```

The SIAM proxy will handle authentication and authorisation. Authorised requests
will be forwarded to the *Health API server*. This will handle API calls internally,
and hand of all other (GET) requests to the front-end app. It can do this in two ways:
either by proxying to another webserver (like webpack-dev-server during development),
or by serving files from the filesystem.

## Definitions

- **day number**: the number of seconds since Unix Epoch Time divided by 86400.
  Please note that this means a day in this context *does not match a day in local time* (or in
  any other timezone). This definition is taken from the Apple/Google specification.

- **daily tracing key**: a 16-byte number used to generate Rolling Proximity Identifiers,
  as broadcast over BLE. This definition is taken from the Apple/Google specification.

- **medical practitioner**: a person or organisation who is legally allowed to run Covid tests.

- **tracing authorisation token**: an access token that must be used to submit daily tracing keys.
