# Platform services and integrity

The menu interfaces with several external services: identity, version information, social features, reports, and appeals. A method that returns locally works differently than a service that accepts a request. This distinction matters for this code, but the UI tends to overlook it.

The platform layer connects the game to the player's account and external services. It handles statistics and achievements, rich presence, communication permissions, and platform storage. The user service manages authentication and reporting, while the game's UI displays sanctions and appeals. A version service provides compatibility and cross-platform configuration. A checksum-based mod-state path tells the game when selected installed files differ from their reference state. These responsibilities converge at the menu, but treating them as equivalent measures of trust or connectivity is incorrect.

## Identity and initialization

`PlatformManager` initializes the selected platform at Unity lifecycle boundaries, delegating to the platform implementation. In the Steam-oriented user implementation examined here, `UserManager.Initialize` picks a Steam identity provider and an Epic online-service implementation. `LoginAsync` retrieves platform identity information, passes it to the online service, and stores the returned user. Authentication renewal happens via `OnUserAuthenticationAboutToExpire` and `RenewAuthAsync`.

A successful login needs a success message. It should never print the credential itself.

`UserManager.LoginAsync` logs the returned user's authentication token in the successful-login branch. This sends a credential into the standard diagnostic stream alongside routine success messages. The logging is part of the success path, not confined to an exception handler.

The immediate recipient is Unity's logging system. A diagnostic record meant only to confirm that login succeeded instead includes the credential. Any subsequent handling of that log output must therefore treat it as sensitive. This chapter does not reproduce a credential, identify a live session, or explain how to extract one.

Logging an authentication token represents a serious secret-handling flaw. The success path needs a result and, optionally, a nonsecret correlation reference. There is no justification for copying the credential into routine diagnostics. Redacting the secret at every subsequent log consumer cannot replace excluding it at the source. `UserManager.LoginAsync` expands the number of places that must protect the credential without improving its success report in any meaningful way.

## Gameplay consumers

The gameplay code accesses these services through various consumers:

- `AchievementManager` requests and stores platform statistics; its progression role is addressed separately.
- `CharacterVoiceHandler` verifies communication permission before allowing transmission.
- `ReportPlayerPopup.OnReportPlayer` checks the selected report category and sends the report to `UserManager.ReportPlayer`.
- `BlocklistTable` merges locally saved entries with platform entries and alerts listeners when block state changes.
- `NetworkConnector.OnSanctionsChanged` and `MainMenuMainPage` evaluate sanction state to determine connection behavior and user-facing options.

## Version information

`CloudAPI.CheckVersion` retrieves current compatibility information and calls the supplied response callback when successful. `MainMenuPageHandler.Start` uses it to refresh next-level information and load the appropriate map bundle. A separate coroutine waits for `CloudAPI.CrossplaySettings.IsValid` before applying server settings. `CloudAPI.OnResponse` populates that configuration from the response.

The weaker point is asynchronous completion. Several methods conceal an external result behind logging or rely on callbacks that may never execute. The UI then guesses whether an operation finished by examining local state that describes something unrelated.

`CloudAPI.CheckVersion` clears its querying status when the request finishes. On success it parses the response and triggers the callback. On error it logs the problem, but the connection-error branch skips the callback entirely. Other error types produce a response marking the version unacceptable.

`MainMenuPageHandler.Start` has two users of this operation. The response callback updates next-level and map-bundle information, while `AssignCrossplaySettings` waits for the configuration to become valid. `CloudAPI.OnResponse` populates that configuration alongside the supplied callback. When a connection fails initially, neither user receives a completed result, and the configuration wait has no local path to failure.

Clearing the querying status does not solve this dependency. It signals request completion; it does not fill the configuration or tell the callback consumer that no configuration will come from this attempt. The caller lacks a typed outcome that separates temporary unavailability from version rejection.

`CloudAPI.CheckVersion` leaves callers stranded on the connection-error path. Finishing the request without invoking the callback leaves dependent work waiting for a success that will never arrive. Use a typed outcome covering success, transport failure, and rejection, and make the configuration wait observe it. The current interface passes its missing failure handling to consumers without giving them the information they need to recover.

## Presence and integrity

Rich presence has narrower scope. `RichPresenceService` delegates through `IRichPresence`, refreshes when room membership changes, and accepts status updates from the main menu, airport, and mountain progression. `SteamRichPresence.SetState` publishes the current activity and party size via Steam's presence interface. This is a defined social feature, not evidence of an unspecified gameplay analytics system.

Platform interfaces prevent most gameplay consumers from handling provider-specific operations directly. Rich presence has a particularly simple interface: gameplay picks an activity, and the provider publishes it.

`ModUtility.InitialiseAsync` calculates checksums for a chosen set of installed files and compares them against reference information. The resulting mod-state flag is used by the main-menu warning path and room setup. This creates a compatibility-related signal. It does not confirm the presence, absence, or completeness of a separate anti-cheat system, and this chapter does not explain how to change the signal.

## Reports and appeals

Both the reporting UI and the user-service implementation were examined. This distinction matters because closing a popup is local, while a successful report is external. These two outcomes do not share a completion result in this implementation.

Finding out whether a report was received is much harder than making the popup disappear.

`ReportPlayerPopup.OnReportPlayer` calls `UserManager.ReportPlayer`, fires its local `OnReport` event, and closes the popup. `UserManager.ReportPlayer` is an asynchronous void method. It awaits the external report operation internally but gives the popup neither a task nor a completion callback. Failures are logged; unauthenticated calls return without submitting.

The popup cannot tell a successful report from a failed attempt when deciding whether to close. Its close path resets the form. Users receive feedback based on dispatching the call, not on receiving a result from the service.

This does not mean reports usually fail. It means failure and success look the same to the UI that gathered the report. Users cannot get reliable delivery confirmation from this path, and the form disappears before any asynchronous failure gets logged.

The reporting flow provides no reliable delivery feedback to users. `UserManager.ReportPlayer` hides the result in asynchronous void execution while the popup closes and clears. Provide an awaitable outcome, show a pending state, and keep the text if submission fails. Sending a request is not the same as the service accepting it. Treating a user-authored report as optional fire-and-forget telemetry is a poor interface choice with an obvious cost when submission goes wrong.

`UserManager.SendAppeal` sets `HasSentAppealThisSession` before it awaits the external appeal operation. Failed results and any caught exceptions get logged, but the flag itself is never cleared. When `MainMenuMainPage` checks this flag to decide whether to show the appeal path, it relies on that setting.

The flag's consumer therefore treats any attempted submission as if the appeal was already sent. A network or service failure leaves the session in a state that blocks another submission through that path, even though this is just a local lifecycle decision. It does not mean the server accepted an appeal or that any submission limit was enforced.

The problem is that `HasSentAppealThisSession` conflates two different things: an attempt and a successful submission. While an in-flight flag can prevent simultaneous requests, it needs to clear when the attempt finishes. A separate flag could track successful results instead. Right now the session stays marked as sent even after failure, which causes the UI to suppress any recovery from a request that never succeeded. This lockout is something the client creates through its own state management.

## Reading and changing this code

When working with this code, separate platform initialization, user authentication, statistics readiness, and room connectivity as independent states. Features that depend on one of these should check that specific dependency directly rather than inferring it from whether another service is available. The achievement and voice chapters demonstrate how different parts of the system consume these states.

Keep compatibility signals and account restrictions distinct from visual customization. This analysis describes what each one does without offering specific changes. Before sharing diagnostic material, review it for sensitive content. This matters especially because the login path being examined logs more information than just a success indicator.

## Open questions

No requests, reports, appeals, or account actions were sent during this analysis. Delivery rates, service-side validation, retention policy, and live configuration are not established by the local call paths. Authentication failure and version-query behavior have not been exercised against the official services here.

The inspected managed user implementation contains a sanction-poll method that updates its timestamp without retrieving sanction data. Its presence and the game-side sanction consumers do not establish how sanctions are populated across every provider or shipped configuration. Broader enforcement conclusions require additional provider and native evidence.

The pass identified explicit version, identity, social, statistics, and reporting functions. It did not establish a complete inventory of SDK telemetry or native integrity behavior. A filename search cannot prove the absence of such behavior, and the checksum mechanism alone is not a complete security assessment.
