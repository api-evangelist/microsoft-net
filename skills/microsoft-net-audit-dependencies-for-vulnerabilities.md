---
name: Audit .NET dependencies for known vulnerabilities
description: Use the NuGet vulnerability feed and the dotnet CLI to find and remediate vulnerable packages, including transitive ones.
api: openapi/microsoft-net-serviceindex-api-openapi.yml
operations: [getServiceIndex, getRegistrationIndex, listPackageVersions]
---

# Audit .NET dependencies for known vulnerabilities

## 1. Find the advisory feed

Call `getServiceIndex` — `GET https://api.nuget.org/v3/index.json` — and read the `@id` of the
`VulnerabilityInfo/6.7.0` resource. On nuget.org that is
`https://api.nuget.org/v3/vulnerabilities/index.json`.

This is a machine-readable advisory feed, published by the same source that serves the packages. It
is the data behind NuGet Audit, and it is anonymous.

## 2. Prefer the CLI for a real project

```
dotnet list package --vulnerable --include-transitive
```

`--include-transitive` matters. Most vulnerable dependencies in a .NET project are not the ones in the
`.csproj`.

## 3. Resolve a safe upgrade

For each flagged package, call `listPackageVersions` — `GET /flatcontainer/{id}/index.json`, ID
lowercased — for the authoritative version list, then `getRegistrationIndex` —
`GET /registration5-semver1/{id}/index.json` — to read each version's metadata and dependency ranges.

Pick the lowest version that clears the advisory and still satisfies your target framework. Upgrading
further than necessary is how an audit fix becomes a breaking change.

```
dotnet add package <Id> --version <SafeVersion>
dotnet restore
dotnet build
```

## 4. Check the platform itself, not just the packages

.NET servicing releases are frequently security releases. The machine-readable channel feed at
`https://raw.githubusercontent.com/dotnet/core/main/release-notes/releases-index.json` gives every
channel's latest release, support phase and EOL date; each channel's `releases.json` lists every
servicing release with its CVE list.

Confirm your channel is still supported before you spend time on package CVEs — a project on an
out-of-support channel is receiving no patches at all. See `lifecycle/microsoft-net-lifecycle.yml`.

## 5. Harden the supply chain

nuget.org repository-signs every package it serves (`RepositorySignatures/5.0.0` in the service index)
and the client validates the signature. Beyond that, enable Central Package Management, NuGet Audit,
and Package Source Mapping — the first-party NuGet MCP server will review a repository's posture on
exactly those four points and recommend hardening steps (`dnx NuGet.Mcp.Server --yes`, requires a
.NET 10 SDK).

## Errors

`404` on any path here is almost always an uppercased package ID. Lowercase and retry once. `429`
carries its wait time as prose in the response body, not in a `Retry-After` header.
