# Openkoda Authenticated Cross-Tenant Private File Disclosure

## Executive Summary

An ordinary authenticated user in one Openkoda organization can retrieve a private file belonging to another organization through the HTML frontend-asset handling path when the file identifier is known or guessed. The failure is an authorization and organization-boundary bypass: the request is authenticated, but the lookup used by this path is not constrained by the requesting user's organization or the file's private status. This assessment does not demonstrate file-ID enumeration, file modification, deletion or unauthenticated access.

The earliest released version I could verify as affected is 1.5.0. The behaviour was verified in releases 1.5.0, 1.5.1 and 1.7.1. The exact authenticated branch described here was not present in the inspected 1.4.3 code. No fixed release was identified in the inspected history.

The affected path requires a valid ordinary account in a different organization from the file owner and a known or guessed numeric file identifier. The demonstrated impact is cross-organization disclosure of a private file through an authenticated request.

I reviewed the supplied private assessment, its source references and its recorded disposable-lab results; I did not rerun the trigger while preparing this public-safe reference. The raw request, file identifier and target-specific response are intentionally omitted here and can be supplied privately to the vendor or CNA.

## Background

Openkoda serves uploaded files through more than one controller path. The ordinary file-content path is expected to enforce organization scope and file-read authorization before returning a private file. Frontend assets have a separate request path, so it must apply the same authorization policy before streaming content.

In the validation scenario, Alice owned a private file in one organization and Mallory had a normal account in a different organization. Mallory did not need Alice's session or administrative privileges. The security boundary that should have stopped her was the file's organization ownership and private-read policy.

## Vulnerability Details

The relevant source flow in the assessed Openkoda code is:

1. `openkoda/src/main/java/com/openkoda/controller/frontendresource/FrontendResourceAssetController.java`, `getFrontendResourceAsset`, handles the HTML frontend-asset branch and uses `repositories.unsecure.file.findOne` to resolve the requested file.
2. That lookup is not scoped to the requesting user's organizations and does not require the file to be public before the controller streams the content.
3. `openkoda/src/main/java/com/openkoda/controller/file/FileControllerHtml.java`, `FileControllerHtml.content`, takes the ordinary file-content path and uses `secureFileRepository.findOne`, which applies the normal organization and read-privilege rules.

The security failure is therefore specific to the frontend-asset branch: it checks that the caller is an authenticated user but does not carry the ordinary ownership and privacy checks through to the file lookup. A private file belonging to Alice can be returned to Mallory when Mallory supplies its numeric identifier through that branch.

The recorded validation used a disposable local Openkoda 1.7.1 deployment. Alice's private file was not returned through the ordinary organization-scoped path when requested by Mallory, but the frontend-asset request returned the private file body with a successful response. An unauthenticated request did not return the file. These controls distinguish the issue from public-file access and from a missing login requirement.

The earliest verified affected release is 1.5.0, with the same behaviour verified in 1.5.1 and 1.7.1. The inspected 1.4.3 code did not contain this exact authenticated branch. No fixing release was identified in the inspected history.

## Exploitability Analysis

Mallory needs a valid ordinary Openkoda account in a different organization and a known or guessed numeric file identifier. She reuses her own session; she does not need to steal Alice's credentials, break transport security or become an administrator.

The ordinary secure file path denied Mallory's cross-organization request, while the frontend-asset path returned Alice's private content. The unauthenticated control also failed, showing that authentication still gates the route. The evidence supports a targeted cross-organization confidentiality loss for an identified private file. It does not establish file-ID enumeration, bulk disclosure, write access, deletion or broader account compromise.

## Proof of Concept

The public reference intentionally does not include the frontend-asset URL shape, a file identifier, session material, a ready-to-run command or captured file contents. Those details would enable direct retrieval from an unpatched deployment while the source-level authorization gap and impact are sufficient for CNA review.

For safe confirmation, use only a disposable Openkoda instance owned by the tester. Create a private file for Alice, place Mallory in a separate organization, verify that the ordinary organization-scoped download is denied, and then compare the separate frontend-asset path while using a test file identifier. The private assessment record contains the exact request and observed response from Openkoda 1.7.1 and can be shared with the vendor or CNA under coordinated disclosure.

## Remediation

The frontend-asset branch must use the same organization and file-read authorization as the ordinary file-content endpoint before it resolves or streams a file. The preferred fix is to replace the unscoped repository lookup with an authorization-enforcing lookup. If a separate query is required, it must constrain the file to an organization allowed for the current user, require the intended read privilege and enforce the file's private/public policy.

Regression tests should cover an allowed same-organization read, a denied cross-organization read, a denied private-file read by an unauthenticated user, a frontend-asset request for a private file and any intentionally supported administrator behaviour. The proposed remediation is not a claim that a vendor fix has shipped.

No fixed release was identified in the inspected history.

## Summary

In releases 1.5.0, 1.5.1 and 1.7.1, an authenticated ordinary user from one organization can retrieve another organization's private file through a frontend-asset path when the numeric file identifier is known or guessed. The ordinary secure path enforces the boundary, but this branch uses an unscoped lookup. The public report omits the exact request and identifier; those remain available for private coordinated disclosure.
