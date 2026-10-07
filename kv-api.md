# kv-api: in-memory key/value HTTP service

## Purpose

A small HTTP service that stores string values by key in memory. It exists to exercise the
spec → generated code → attestable AMI pipeline, so it must be simple, self-contained and fully tested.

## Language and runtime

Go, standard library only, compiled to a single static binary (`CGO_ENABLED=0`). No runtime
packages are needed on the target machine.

## HTTP API

JSON everywhere. Listen on `0.0.0.0`, port from the environment variable `PORT`, default `8080`.

| Method | Path | Request | Responses |
|---|---|---|---|
| GET | `/healthz` | – | `200 {"status":"ok"}` once the server accepts requests |
| PUT | `/kv/{key}` | `{"value":"<string>"}` | `201` when the key is new, `200` when overwritten, body `{"key":"…","value":"…"}` |
| GET | `/kv/{key}` | – | `200 {"key":"…","value":"…"}`; `404` if missing |
| DELETE | `/kv/{key}` | – | `204` with no body; `404` if missing |
| GET | `/kv` | – | `200 {"keys":["a","b",…]}`, keys sorted ascending; empty list when the store is empty |

Rules:

- A key is 1 to 64 characters from `[A-Za-z0-9_.-]`; anything else is `400`.
- A value is at most 4096 bytes; larger is `413`. A malformed or missing JSON body on PUT is `400`.
- At most 10000 keys; a PUT that would add a new key beyond that is `507`.
- Any other path or method is `404` / `405`.
- Every error response is `{"error":"<human readable message>"}` with the matching status code.

## Behaviour

- Data lives in memory only and is lost on restart. Concurrent requests must be safe.
- Log one line per request to stdout: method, path, status code, duration.
- Shut down gracefully on SIGTERM and SIGINT (stop accepting, finish in-flight requests, exit 0).
- No configuration files, no external services, no authentication.

## Tests

`go test ./...` must cover every endpoint and every error case listed above, including the key
limit and the value size limit, without network access or external tools.
