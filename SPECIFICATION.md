# Personal Photo Archive Specification

Status: agreed product direction; implementation has not started.

## 1. Purpose

Create a permanent, original-quality archive of an iPhone's iCloud Photos
library on a personal home server. Provide Mac tools for reviewing and
cleaning up imported media, and an iPhone app for browsing the archive with
encrypted offline storage.

Privacy is intended to protect against ordinary prying eyes, not forensic
inspection. A separate demo gallery provides a decoy unlock experience.

## 2. Deployment

- A Mac app accesses the user's Photos library through supported PhotoKit APIs.
- A single Docker service runs on a separate home server.
- Server files and its catalog persist outside the container.
- The user's existing home VPN provides remote network access.
- No Tailscale dependency, public port forwarding, or unofficial iCloud login
  automation is required.
- An iPhone companion app connects to the same service.
- Build, signing, and Apple-platform validation take place on a Mac with Xcode.

Initial technology direction:

| Component | Technology |
| --- | --- |
| Mac app | Swift, SwiftUI with AppKit where needed, PhotoKit |
| iPhone app | Swift, SwiftUI, platform cryptography and Keychain |
| Server | Go HTTP service in Docker |
| Catalog | SQLite on a local persistent server volume |
| Archive storage | Ordinary files on a persistent server volume |
| Transport | HTTPS over the existing home VPN |

## 3. Original-quality archiving

The Mac app must:

- Request full Photos library access and make incomplete authorization visible.
- Retrieve underlying asset resources, downloading cloud-only resources when
  needed instead of archiving optimized previews.
- Preserve original bytes without resizing, transcoding, or re-encoding.
- Preserve photos, videos, RAW resources, Live Photo still/video components,
  and available edited resources and adjustment data.
- Record original filenames, resource types, dates, dimensions, available
  location metadata, album associations, and source-library identifiers.
- Generate separate browsing thumbnails without modifying originals.
- Preserve smaller imported files exactly as received. Quality already lost
  before import cannot be reconstructed.

The server archive is independent of Photos:

- Source deletion never triggers deletion of archived files.
- Available changed resources are retained as additional versions.
- Identical byte content can be deduplicated without merging distinct asset
  records or losing their metadata.
- Original filenames are metadata, not unique storage keys.
- Uploads are resumable and retryable, with explicit error states.
- The server verifies resource sizes and cryptographic checksums before marking
  an asset archived. Partial uploads never count as complete.
- A complete asset requires all resources in its captured export manifest.
- Source-library identifiers are scoped to a library/device; they are not
  assumed to be universal identifiers across devices.

The app displays last successful sync, pending resources, errors, and backup
status. It reconciles the library after interruptions, not just while receiving
change notifications.

Limitations:

- The Mac must be awake, connected, and able to access Photos for ingestion.
- Files deleted before successful ingestion are not guaranteed recoverable.
- Available editing resources do not guarantee exact recreation of Apple's
  editing experience in another viewer.

## 4. Imported-media classification

Provide categories for camera originals, likely WhatsApp/imported media,
screenshots, other/unknown media, and user-defined categories.

Use available evidence such as:

- Original filename patterns associated with messaging exports.
- Camera and other embedded metadata.
- Dimensions and resource types.
- Album membership and explicit user rules.

Each inferred classification shows its evidence and confidence. Users can
correct it and filter by it. Small dimensions or missing camera metadata alone
must not establish a source or authorize deletion.

Apple does not reliably expose the app that saved every photo. Source
classification is best-effort and may remain unknown.

The application does not access WhatsApp's private database or chats. It works
with media already saved into Photos.

## 5. Mac review and cleanup

Provide a thumbnail grid and a list, with:

- Shift-click range selection.
- Command-click selection toggling.
- Select-all within the current filter.
- Filters for category, date, dimensions, size, and archive/backup status.
- Full-size preview, metadata, and classification evidence.
- Clear selection behavior when sorting or filtering changes.

Cleanup sequence:

1. User selects specific Photos assets.
2. App captures and archives every required resource.
3. Server verifies archive integrity.
4. A separate backup covering those resources and their catalog records is
   verified.
5. App rechecks that the selected source assets still match the archived
   resources; changed or unresolved assets are blocked.
6. App shows the selection, affected scope, and an explicit confirmation.
7. App requests deletion through PhotoKit and reports the actual result.

Deletion is never automatic based on classification. Failed or incomplete
archive/backup verification blocks deletion.

With iCloud Photos enabled, deletion affects iCloud and synced devices, not just
the iPhone. This warning must appear before every cleanup confirmation.
Recently Deleted remains controlled by Apple; the app does not automatically
empty it.

Photos deletion does not necessarily remove copies retained within WhatsApp.
For device-only space savings while retaining iCloud photos, recommend Apple's
Optimize iPhone Storage rather than presenting deletion as local-only.

## 6. Server access and storage

Server-side file encryption is not required for the initial release.
The vault is an access-control boundary, not an encrypted server filesystem.

- Pair each client using its own persistent, high-entropy random credential.
- Protect Mac credentials in Keychain and server credentials in restricted
  persistent configuration; never bake credentials into images or source.
- Provide explicit credential revocation and replacement.
- Device pairing alone must not unlock private gallery content.
- Enforce private-gallery authorization on all metadata, search, thumbnails,
  previews, downloads, and original-file endpoints.
- Do not expose an unauthenticated static directory containing archive files.
- Keep credentials and sensitive media details out of routine logs.
- Persist configuration and credentials across container restarts.
- Bind/expose the service only on intended home/VPN interfaces.

A configurable port is sufficient; a random port is not a security boundary.
HTTPS remains required and still uses a certificate. Device pairing credentials
do not replace TLS.

Support explicit server-certificate/public-key pinning during trusted pairing,
with a deliberate re-pairing or trusted rotation procedure. Never disable TLS
verification globally or silently accept certificate changes.

Server administrators and anyone with filesystem access to the archive or
backups can read stored files. This is an accepted limitation of the initial
design.

## 7. Real vault and decoy gallery

No Face ID or other biometric authentication is used.

- A real PIN/passphrase unlocks the private gallery.
- A distinct decoy PIN opens a separate preset demo gallery.
- Demo images are explicitly supplied or approved sample images, never
  automatically drawn from the private archive.
- The demo gallery is stable and supports ordinary browsing and previews.
- Demo sessions cannot access private originals, thumbnails, filenames,
  metadata, counts, search results, or background-sync details.
- The server enforces gallery separation, not just the client interface.
- Real and decoy credentials must differ; reject ambiguous configuration.
- Unlock attempts are rate-limited, and credential verifiers are stored using
  a suitable password-hashing mechanism rather than plaintext.
- Entering demo mode invalidates the client's real-vault session, cancels
  pending private requests, and clears decrypted private previews.
- Returning to the private gallery requires locking and authenticating again.

The demo mode protects against casual viewing only. It does not promise that
the private gallery's existence is undetectable. No forensic-evasion features
are in scope.

## 8. iPhone browsing and encrypted offline storage

The companion app supports:

- Browsing and previewing over the home VPN.
- Downloading selected items for offline viewing.
- Seeing download progress, storage usage, and explicit connection errors.
- Removing local downloads without deleting server archives or Photos assets.

All persisted local media, thumbnails, private metadata, and sensitive caches
must be encrypted. Real and demo collections use separate encryption keys.

Key requirements:

- Use authenticated encryption through established platform/library facilities.
- Protect the real collection key with the real PIN/passphrase using a reviewed
  password-based key-wrapping design and an appropriate key derivation function.
- Use Keychain protection without biometric requirements as an additional
  device-bound layer, not as a substitute for the app's real credential.
- The decoy credential must not unwrap the real collection key.
- Pairing credentials must not independently bypass the real-vault unlock.
- Store no plaintext unlock credentials or raw keys beside encrypted files.
- Retain decrypted keys only for the unlocked session as far as platform
  facilities permit.
- Recommend a longer passphrase; short numeric PINs are vulnerable to offline
  guessing even when online attempts are rate-limited.

Lifecycle and leakage controls:

- Lock when the app backgrounds and after a configured inactivity timeout.
- Obscure sensitive content in the app switcher.
- Avoid plaintext HTTP/disk caches and decrypted temporary files; where a media
  API requires a temporary file, use iOS file protection and prompt cleanup.
- Do not automatically export private files into Photos or Files.
- Warn that explicit exports fall outside the app's encryption boundary.
- Exclude recreatable encrypted downloads and device-bound secrets from app
  backups by default; downloads can be restored from the server.

Protection does not extend to an already-unlocked app, screenshots, an external
camera, or files explicitly exported by the user. No claim is made that all
decrypted memory can be securely erased on iOS.

## 9. Backups and recovery

- Maintain a backup on a separate storage target, not merely another directory
  on the same disk.
- Include original resources, manifests, and a transactionally consistent
  catalog snapshot.
- Preserve older backups so source cleanup or server mistakes do not
  immediately destroy recovery options.
- Verify backup completeness and checksums before allowing Photos cleanup.
- Provide a restore procedure and test restoration to a clean service instance.
- Treat thumbnails as regeneratable, not substitutes for backed-up originals.
- A successful upload is not a successful backup.

Credential reset/recovery requires explicit administrative authorization, not
the decoy credential. Changing a real credential must preserve access through
safe key rewrapping where possible. If a phone's local key is unrecoverable,
discard encrypted local copies and re-download after authorized re-pairing.

Server backups are not encrypted by this application in the initial release
and require appropriate filesystem permissions.

## 10. Delivery phases

1. Validate PhotoKit export on the Mac using representative real assets.
2. Implement the Docker server, persistent catalog, verified ingestion, and
   independent backups.
3. Implement Mac synchronization, review, classification, and range selection.
4. Implement server-enforced real/demo access and credential lifecycle.
5. Implement iPhone browsing and encrypted offline storage without biometrics.
6. Enable Photos cleanup only after archive, backup, and recovery tests pass.

## 11. Acceptance criteria

- Exported original resource checksums equal checksums of the corresponding
  stored and restored resources.
- Test HEIC, JPEG, RAW, video, Live Photos, edited assets, and cloud-only assets.
- Interrupted/repeated uploads do not corrupt files or duplicate asset records.
- Deleting a successfully archived source asset leaves its archive and backups
  intact.
- Missing resources, failed backups, changed source assets, and canceled
  PhotoKit requests never produce a success-shaped cleanup result.
- Shift-click selects the intended inclusive range in the current visible
  ordering; Command-click and filter changes behave consistently.
- Unknown sources remain unknown instead of being falsely labeled WhatsApp.
- Demo credentials cannot access any real-gallery API or local encrypted
  collection, including after previously unlocking the real gallery.
- Backgrounding and switching modes do not leave private previews visible.
- Inspecting persisted iPhone app data finds no plaintext private media,
  thumbnails, or private metadata.
- Wrong credentials fail explicitly; rate limits and real credential changes
  work without biometrics.
- Container restarts preserve archives, catalog, and pairing configuration.
- Unexpected HTTPS certificate changes block connections until explicitly
  resolved.
- A clean restore recovers original resources and their catalog associations.

## 12. Items to confirm during implementation

- Server OS, storage layout, separate backup destination, and available space.
- Supported macOS/iOS versions and signing/distribution method.
- Library size and acceptable initial synchronization duration.
- Initial demo-gallery images.
- Real credential policy and inactivity timeout.
- Exact certificate provisioning and rotation workflow for the home VPN.

These details do not change the agreed architecture, but must be confirmed
before deployment or enabling destructive cleanup.
