# Platform services and integrity

The menu sits in front of several external services: identity, version information, social features, reports, and appeals. A local method returning is not the same thing as a service accepting a request. This ought to be a basic distinction for this code. The UI repeatedly has to live without it.

The platform layer connects the game to the player's account and external services. It supplies statistics and achievements, rich presence, communication permissions, and platform storage. The user service supports authentication and reporting, while game-side UI presents sanctions and appeals. A version service supplies compatibility and cross-platform configuration. A checksum-based mod-state path informs the game that selected installed files differ from its reference state. These responsibilities overlap at the menu, but they are not interchangeable measures of trust or connectivity.

## Identity and initialization

`PlatformManager` initializes the selected platform at Unity lifecycle boundaries, delegating the work to the platform implementation. In the inspected Steam-oriented user implementation, `UserManager.Initialize` selects a Steam identity provider and an Epic online-service implementation. `LoginAsync` obtains platform identity information and passes it to the online service, then retains the returned user. Authentication renewal is attached through `OnUserAuthenticationAboutToExpire` and `RenewAuthAsync`.

A successful login needs a success message. It does not need to print the credential.

`UserManager.LoginAsync` logs the returned user's authentication token on the successful-login branch. This puts a credential into the ordinary diagnostic stream alongside routine success messages. The logging is part of the successful path, not restricted to an exception where extra information was being gathered.

The immediate consumer is Unity's logging system. A diagnostic record that should establish only that login succeeded is instead given the credential itself. Any later handling of that diagnostic output must therefore treat it as sensitive. This chapter does not reproduce a credential, identify a live session, or provide instructions for extracting one.

Logging an authentication token is a serious secret-handling defect. The success path needs a result and, if useful, a nonsecret correlation reference. It has no business copying the credential into ordinary diagnostics. Redaction by every later log consumer is not a substitute for excluding the secret at the call site. `UserManager.LoginAsync` expands the places that must protect the credential without making its success report meaningfully better.

## Gameplay consumers

The gameplay-facing code accesses those services through different consumers:

- `AchievementManager` requests and stores platform statistics; its progression responsibilities are covered separately.
- `CharacterVoiceHandler` checks communication permission before allowing transmission.
- `ReportPlayerPopup.OnReportPlayer` validates the selected report category and forwards the report to `UserManager.ReportPlayer`.
- `BlocklistTable` combines locally saved entries with platform entries and notifies listeners when block state changes.
- `NetworkConnector.OnSanctionsChanged` and `MainMenuMainPage` interpret sanction state for connection behavior and user-facing options.

## Version information

`CloudAPI.CheckVersion` requests current compatibility information and invokes the supplied response callback on a successful response. `MainMenuPageHandler.Start` consumes it to update next-level information and prepare the relevant map bundle. A separate coroutine waits for `CloudAPI.CrossplaySettings.IsValid` before applying server settings. `CloudAPI.OnResponse` fills that configuration from the received response.

The weaker boundary is asynchronous completion. Several methods hide an external result behind logging or use a callback that is not guaranteed to run. The UI then has to infer whether an attempt has finished from local state that describes something else.

`CloudAPI.CheckVersion` clears its querying status when the request completes. On success it parses the response and invokes the callback. On an error it logs the error, but the connection-error branch does not invoke the callback at all. Other error categories produce a response that marks the version unacceptable.

`MainMenuPageHandler.Start` has two consumers of this operation. Its response callback updates next-level and map-bundle information, while `AssignCrossplaySettings` waits until the configuration becomes valid. That configuration is populated through `CloudAPI.OnResponse`, which is combined with the supplied callback. On an initial connection failure, neither consumer gets a completed result and the configuration wait has no local failure exit.

The querying status being cleared does not solve that dependency. It describes request completion; it does not populate the configuration or notify the callback consumer that no configuration will arrive from this attempt. The caller has no typed outcome distinguishing temporary unavailability from version rejection.

`CloudAPI.CheckVersion` fails its callers on the connection-error path. Completing the request without completing the callback leaves dependent work waiting for a success state that this attempt will never supply. Return a typed outcome for success, transport failure, and rejection, and make the configuration wait observe it. The present interface offloads its missing failure handling onto consumers that were never given the information to recover.

## Presence and integrity

Rich presence is more narrowly scoped. `RichPresenceService` delegates through `IRichPresence`, refreshes on room membership changes, and accepts status updates from the main menu, airport, and mountain progression. `SteamRichPresence.SetState` publishes the current activity and party size using Steam's presence interface. This is an identified social feature, not evidence of an unspecified gameplay analytics system.

Platform interfaces keep most gameplay consumers from handling provider-specific operations directly. Rich presence has a particularly small interface: gameplay selects an activity, and the provider publishes it.

`ModUtility.InitialiseAsync` computes checksums for a selected collection of installed files and compares them with reference information. The resulting mod-state flag is consumed by the main-menu warning path and room setup. That establishes a compatibility-related signal. It does not establish the existence, absence, or completeness of a separate anti-cheat system, and this chapter does not describe how to alter the signal.

## Reports and appeals

The reporting UI and the user-service implementation were both inspected. That matters because a popup disappearing is a local action, while a successful report is an external operation. The two do not share a completion result in this implementation.

Closing the form is much easier than finding out whether the report arrived.

`ReportPlayerPopup.OnReportPlayer` calls `UserManager.ReportPlayer`, invokes its local `OnReport` event, and hides the popup. `UserManager.ReportPlayer` is an asynchronous void method. It awaits the external report operation internally, but exposes neither a task nor a completion callback to the popup. A failure is logged; an unauthenticated call returns without submission.

The popup therefore cannot distinguish a successfully accepted report from a failed attempt when deciding to close. Its hide path resets the form's state. The user's feedback is based on having dispatched the call rather than having received a result from the service.

This does not mean that reports normally fail. It means that failure and success are indistinguishable to the UI that collected the report. The user cannot get delivery feedback from this call path, and the form has already been cleared when an asynchronous failure is logged.

The reporting flow gives the user no trustworthy delivery feedback. `ReportPlayer` hides the result behind asynchronous void execution while the popup closes and clears the form. Return an awaitable outcome, show pending state, and retain the text on failure. Dispatch is not acceptance. Treating a user-authored report like optional fire-and-forget telemetry is a bad interface decision with an obvious cost when submission fails.

`UserManager.SendAppeal` sets `HasSentAppealThisSession` before awaiting the external appeal operation. Failed results and caught exceptions are logged without clearing that flag. `MainMenuMainPage` checks the flag when presenting the appeal path.

The variable's consumer therefore treats an attempted submission as an already sent appeal. A network or service failure still leaves the session in the state used to suppress another submission through that path. This is a local lifecycle decision, not evidence that the server accepted an appeal or imposed a submission limit.

`HasSentAppealThisSession` is incorrectly used for both an attempt and a successful submission. An in-flight flag can block simultaneous requests, but it must clear when the attempt ends. A separate accepted-result flag can track success. The current code leaves the session marked sent after failure, so the UI suppresses recovery from a request that did not succeed. That is an avoidable lockout created by the client's own bookkeeping.

## Reading and changing this code

Treat platform initialization, user authentication, statistics readiness, and room connectivity as different states. A feature that needs one of them should inspect that dependency rather than infer it from another service being available. The achievement and voice chapters show distinct consumers of those states.

Keep compatibility signals and account restrictions separate from visual customization. This analysis describes their responsibilities without supplying alteration instructions. Diagnostic material should be reviewed for sensitive content before it is shared, particularly because the inspected login path logs more than a success indication.

## Open questions

No requests, reports, appeals, or account actions were sent during this analysis. Delivery rates, service-side validation, retention policy, and live configuration are not established by the local call paths. Authentication failure and version-query behavior have not been exercised against the official services here.

The inspected managed user implementation contains a sanction-poll method that updates its timestamp without retrieving sanction data. Its presence and the game-side sanction consumers do not establish how sanctions are populated across every provider or shipped configuration. Broader enforcement conclusions require additional provider and native evidence.

The pass identified explicit version, identity, social, statistics, and reporting functions. It did not establish a complete inventory of SDK telemetry or native integrity behavior. A filename search cannot prove the absence of such behavior, and the checksum mechanism alone is not a complete security assessment.
