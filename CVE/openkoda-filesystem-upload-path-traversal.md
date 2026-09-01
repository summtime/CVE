# Openkoda Authenticated Filesystem Upload Path Traversal

## Executive Summary

An ordinary authenticated Openkoda user can cause uploaded content to be written outside the configured filesystem storage root when the application uses `file.storage.type=filesystem`. The security boundary crossed is the application-managed storage directory; the operating-system permissions of the Openkoda process still limit where the write can succeed. This assessment does not demonstrate remote code execution or privilege escalation.

The earliest released version I could verify as affected is 1.4.1. The same implementation is present in the Openkoda 1.4.0 release commit, but a public 1.4.0 tag was not available in the inspected repository. The behaviour was verified in releases 1.4.1, 1.4.3, 1.5.0, 1.5.1 and 1.7.1. No fixed release was identified in the inspected history.

The affected path requires an authenticated ordinary user, filesystem storage mode and a destination that the application process can write to. The default database storage mode does not reach this filesystem sink. The demonstrated impact is authenticated arbitrary file creation or overwrite outside the configured storage root, subject to operating-system permissions.

I reviewed the supplied private assessment, its source references and its recorded disposable-lab results; I did not rerun the trigger while preparing this public-safe reference. The raw request, exact attacker-controlled values and target-specific output are intentionally omitted here and can be supplied privately to the vendor or CNA.

## Background

Openkoda accepts file uploads through its HTML file controller and stores them using a configured storage backend. In filesystem mode, the application should treat upload metadata as data, resolve the final destination beneath the configured storage root and reject any normalized path that escapes that root.

The attacker does not need an administrator account. A normal authenticated account can submit an upload through the ordinary application flow. The affected behaviour depends on filesystem storage being enabled; it is not claimed for the database storage path.

## Vulnerability Details

The relevant source flow in the assessed Openkoda code is:

1. `openkoda/src/main/java/com/openkoda/controller/file/FileControllerHtml.java`, `FileControllerHtml.upload`, accepts upload metadata supplied by the client.
2. `openkoda/src/main/java/com/openkoda/core/service/FileService.java`, `FileService.saveAndPrepareFileEntity` and `prepareStoredFileName`, incorporates that metadata into the stored filename and passes it through `FilenameUtils.concat`.
3. `FileService.handleFilesystemWrite` passes the resulting path to the filesystem output sink.

The missing security decision is a containment check after path normalization. The implementation does not establish that the normalized destination remains below the configured storage root before writing the uploaded bytes. As a result, traversal components in client-controlled upload metadata can change the final destination rather than merely naming a file within the storage directory.

The recorded validation used a disposable local Openkoda 1.7.1 deployment, filesystem storage and a normal authenticated user. A control upload remained beneath the configured root, while the traversal case created the uploaded content at a location outside that root. The observed result was file creation or overwrite within the permissions of the application process.

The earliest verified affected release is 1.4.1. The inspected release history also contains the vulnerable implementation in 1.4.3, 1.5.0, 1.5.1 and 1.7.1. Because the public 1.4.0 tag was unavailable and no fixing release was identified in the inspected history, those facts should not be read as a definitive first-affected or fixed-version boundary beyond the releases stated above.

## Exploitability Analysis

Mallory needs only a valid ordinary Openkoda account and a deployment using filesystem storage. She does not need another user's session or administrative privileges. The write succeeds only where the operating system grants the Openkoda process permission, and the destination's parent directory must be usable by that process.

The database storage mode does not use the vulnerable filesystem sink. The normal in-root upload control shows that ordinary upload handling works as intended for a safe destination; the out-of-root result isolates the failure to path containment. The assessment did not demonstrate remote code execution, privilege escalation, access to arbitrary hosts or a bypass of operating-system permissions.

## Proof of Concept

The public reference intentionally does not include a ready-to-run request, traversal string, session material, file identifiers or target-specific output. That detail would make exploitation against an unpatched deployment unnecessarily easy while adding little to CNA's ability to classify the underlying flaw.

For safe confirmation, use only a disposable Openkoda instance owned by the tester, enable filesystem storage with a temporary root, authenticate as an ordinary user and compare a normal upload with a separately controlled upload whose final normalized destination is outside that root. The private assessment record contains the exact request and observed result from Openkoda 1.7.1 and can be shared with the vendor or CNA under coordinated disclosure.

## Remediation

The filesystem storage code should not allow client-controlled metadata to define directory structure. It should validate the UUID as a safe token, require the filename to be a single path component, resolve the candidate destination against a normalized storage root and reject it unless the resolved path is contained by that root using a path-boundary-safe comparison.

The same validation should cover every filesystem and failover storage path. File creation should also account for symbolic-link races where the deployment threat model requires it. Regression tests should cover traversal in each accepted metadata field, absolute and alternate path forms, safe in-root uploads, existing-file handling and the database storage mode.

No fixed release was identified in the inspected history; the remediation above is a proposed defensive change, not a claim that a vendor fix has shipped.

## Summary

With filesystem storage enabled, an authenticated ordinary Openkoda user can exploit missing post-normalization path containment to create or overwrite uploaded content outside the configured storage root, subject to the Openkoda process's operating-system permissions. The earliest released version verified as affected is 1.4.1, with the same behaviour verified in 1.4.3, 1.5.0, 1.5.1 and 1.7.1. The public report deliberately omits the exact reproduction details; those remain available for private coordinated disclosure.
