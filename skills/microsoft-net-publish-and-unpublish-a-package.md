---
name: Publish and unpublish a NuGet package
description: Push a package to nuget.org, and understand what can and cannot be taken back afterwards — including why a push is effectively permanent.
api: openapi/microsoft-net-serviceindex-api-openapi.yml
operations: [getServiceIndex]
---

# Publish and unpublish a NuGet package

## Read this before you push

**nuget.org does not support permanent deletion of packages.** Once a package ID and version are
published, that version stays downloadable forever. Unlisting hides it from search and from the web
UI, but it remains installable by exact version, still resolves under floating version constraints
(`1.0.0-*`), and still appears in the catalog change feed. Deletion happens only by manual NuGet-team
intervention for copyright infringement or harmful content.

Treat a push as irreversible. Rehearse first.

## 0. Rehearse against the test gallery

There is a full parallel pre-production gallery with its own accounts and its own API keys:

- Gallery: `https://int.nugettest.org/`
- Source:  `https://apiint.nugettest.org/v3/index.json`

nuget.org credentials do not work there and nothing you push there can reach production.

A NuGet API key carries **no visible test/live prefix**. Nothing about the key tells you which gallery
it belongs to. The `--source` argument is the entire safety boundary on this API — check it before
every push.

## 1. Resolve the publish endpoint

Call `getServiceIndex` — `GET https://api.nuget.org/v3/index.json` — and read the `@id` of the
`PackagePublish/2.0.0` resource. On nuget.org that is `https://www.nuget.org/api/v2/package`.

## 2. Push

`PUT` to the publish resource with `Content-Type: multipart/form-data`, the raw `.nupkg` bytes as the
first item in the body, and the header `X-NuGet-ApiKey: {YOUR_API_KEY}`.

Or, from the CLI:

```
dotnet pack
dotnet nuget push <pkg>.nupkg --api-key <KEY> --source https://api.nuget.org/v3/index.json
```

Responses:

- `201` or `202` — pushed. Implementations vary on which success code they return; accept both.
- `400` — the package is invalid.
- `409` — a package with that ID and version already exists. nuget.org will not replace it.

There is **no idempotency key** on this API. Retry safety comes from version immutability alone: a
retried push cannot create a duplicate because the second attempt returns 409. But a 409 does not tell
you whether your own earlier attempt succeeded or someone else published that version first — check
the registration index if you need to know.

## 3. Unlist (the closest thing to an undo)

`DELETE {publish-resource}/{ID}/{VERSION}` with the same `X-NuGet-ApiKey` header.

- `204` — unlisted.
- `404` — no such ID and version.

On nuget.org this is an unlist, not a delete. On other NuGet server implementations the same request
may be a hard delete — the reversibility of this call depends on which source you are pointed at.

## 4. Relist (the actual reversal)

`POST {publish-resource}/{ID}/{VERSION}` — same URL as the unlist, different method.

- `200` — the version is listed again.
- If the package is already listed, the request still succeeds. This operation is idempotent.

No time window is published for how long a version can be relisted.

## 5. Deprecate instead, where you can

Deprecating a version attaches an advisory message without hiding it. It is the gentler signal to
downstream consumers than unlisting, and it is reversible. It is done in the nuget.org web UI; no
public API endpoint is documented for it.

See `conventions/microsoft-net-conventions.yml` for the full reversibility analysis and
`sandbox/microsoft-net-sandbox.yml` for the test environment.
