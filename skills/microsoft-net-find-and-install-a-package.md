---
name: Find and install a NuGet package
description: Search nuget.org for a package, read its metadata, and resolve the exact version and download URL — using the service index rather than hardcoded hosts.
api: openapi/microsoft-net-search-api-openapi.yml
operations: [getServiceIndex, searchPackages, getRegistrationIndex, listPackageVersions]
---

# Find and install a NuGet package

All operations here are anonymous reads. No API key is needed and none should be sent.

## 1. Resolve the source

Call `getServiceIndex` — `GET https://api.nuget.org/v3/index.json`.

Do not hardcode resource hosts. Read `resources[]` and pick the `@id` whose `@type` matches the
resource you need, choosing the highest version you understand:

- `SearchQueryService/3.5.0` for step 2
- `RegistrationsBaseUrl/3.6.0` for step 3
- `PackageBaseAddress/3.0.0` for step 4

This is what makes the same code work against nuget.org, Azure Artifacts, GitHub Packages, Artifactory
and Nexus unchanged.

## 2. Search

Call `searchPackages` against the `SearchQueryService` `@id` — `GET /query`.

Parameters: `q` (search terms), `skip` (default 0), `take` (default 20), `prerelease` (default false),
`semVerLevel` (send `2.0.0`), `packageType`.

Read `totalHits` and `data[]`. Each `SearchResult` carries `id`, `version`, `description`, `authors`,
`totalDownloads`, `verified`, and — most usefully — `registration`, an absolute URL straight to that
package's registration index. Prefer following `registration` over constructing the URL yourself.

Pagination is offset-based: increment `skip` by `take`. There is no cursor.

## 3. Read package metadata

Call `getRegistrationIndex` — `GET /registration5-semver1/{id}/index.json`.

**Lowercase the package ID in the path.** This is the single most common cause of a spurious 404 on
this API. For one specific version, call `getRegistrationLeaf` —
`GET /registration5-semver1/{id}/{version}.json` — with both `id` and `version` lowercased and the
version normalized.

## 4. Resolve versions and content

Call `listPackageVersions` — `GET /flatcontainer/{id}/index.json` — for the authoritative
`versions[]` array. Then `downloadPackage` —
`GET /flatcontainer/{id}/{version}/{id_lower}.{version_lower}.nupkg` — for the bytes, or
`downloadNuspec` — `GET /flatcontainer/{id}/{version}/{id_lower}.nuspec` — for the manifest alone.

## 5. Install

For a project, prefer the CLI over calling the API directly:

```
dotnet add package <Id> --version <Version>
dotnet restore
```

## Errors

- `404` — almost always a casing or version-normalization mistake in the path. Lowercase and retry once.
- `429` — throttled. There is **no** `Retry-After` header; the wait is prose inside the JSON body,
  e.g. `{"statusCode": 429, "message": "Rate limit is exceeded. Try again in 56 seconds."}`. Parse the
  seconds out of `message` and back off.
- `403` `{"statusCode": 403, "message": "Quota exceeded."}` — a longer-window cap, not a throttle. Waiting
  a few seconds will not clear it.
- `400` with code `NuGet.V2.Deprecated` — you are on the legacy V2 OData surface. Move to V3.

Full catalogue: `errors/microsoft-net-problem-types.yml`. Published limits:
`rate-limits/microsoft-net-rate-limits.yml`.
