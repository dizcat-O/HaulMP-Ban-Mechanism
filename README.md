# HaulMP Ban Mechanism

Technical analysis of the ban detection and error-reporting mechanism used by the **HaulMP launcher and updater**.

This repository documents the observed client-side execution flow when the server rejects an update request because the PC or Steam library is banned.

## Overview

When the HaulMP server rejects a request from a banned client, it returns an HTTP error response. The updater propagates this error through its exception-handling and reporting pipeline until the launcher displays the resulting message to the user.

The observed ban response contains the following reason phrase:

> This PC or Steam library is banned from HaulMP.

## Ban Detection Flow

```text
HaulMP Server
     |
     | HTTP 403 Forbidden
     | Reason: PC or Steam library is banned
     v
<Get>d__26::MoveNext
     |
     | EnsureSuccessStatusCode()
     | Throws HttpRequestException
     v
<Run>d__27::MoveNext
     |
     | Catch exception
     | status.Text = ex.Message
     | Finish(failed = true)
     v
Updater::Report
     |
     | Serializes error information
     v
{
  "error": true,
  "message": "<exception message>"
}
     |
     v
HaulMP.Launcher.exe
     |
     | UpdateReport.Error == true
     | Throws IOException(UpdateReport.Message)
     v
Launcher UI
     |
     v
Displays the ban-related error message
```

## HTTP Request Analysis

Based on the examined client implementation, the updater initializes its HTTP client using:

```csharp
new HttpClient(
    new HttpClientHandler
    {
        AllowAutoRedirect = false
    }
);
```

The client also configures a timeout of approximately ten minutes.

The update URL includes a cache-busting query parameter:

```text
?check=<UtcNow.Ticks>
```

### Observed request characteristics

- **Authentication headers:** No authentication headers were observed in the examined client setup.
- **Steam credentials:** No Steam token was observed being explicitly attached to the request.
- **HWID:** No hardware identifier was observed being explicitly attached to the request.
- **Query parameters:** The observed `check` parameter appears to serve as a cache-buster.
- **Redirect handling:** Automatic HTTP redirects are disabled in the configured handler.

These findings describe the examined client code. They do not establish that the complete application never transmits identifying information through another component or request.

## How the Ban Is Identified

The client-side implementation examined here does not reveal an explicit hardware ID, Steam authentication token, or dedicated ban identifier being attached to the relevant HTTP request.

Consequently, the server may identify the client using its source IP address or information maintained server-side. However, the exact server-side identification method cannot be conclusively established from the client implementation alone.

Possible mechanisms include:

- Source IP address matching.
- Server-side records associated with previous requests.
- Identification performed by another component not covered by the examined code.

**Important:** IP-based identification is a hypothesis based on the observed request, not a confirmed description of HaulMP's server implementation.

## Error Propagation

The observed error-handling process consists of four stages:

1. **HTTP rejection:** The server returns an unsuccessful HTTP response.
2. **Exception generation:** `EnsureSuccessStatusCode()` throws an `HttpRequestException`.
3. **Updater reporting:** The exception message is assigned to the status text and serialized into a JSON report.
4. **Launcher display:** The launcher reads the report and raises an `IOException` containing the reported message.

This allows the server's rejection message to propagate through the updater and appear in the launcher UI.

## Scope and Limitations

This repository documents the observed client-side behavior and the conclusions that can reasonably be drawn from it.

It does not include a verified analysis of the server's internal ban database, IP matching rules, Steam library validation, or other server-side enforcement logic.

All conclusions should be interpreted in the context of the examined client implementation.

## Disclaimer

This repository is intended for software analysis, documentation, and educational reverse engineering. It does not claim access to HaulMP's internal server implementation or private infrastructure.
