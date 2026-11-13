# How Blueshift works

[← Back to the README](../README.md)

- [Verdicts and readiness](#verdicts-and-readiness)
- [What it handles](#what-it-handles)
- [One integration, through the Jamf API gateway](#one-integration-through-the-jamf-api-gateway)
- [The screens](#the-screens)
- [Scope: comparison, smart groups and "All" scopes](#scope-comparison-smart-groups-and-all-scopes)
- [The blueprint's description](#the-blueprints-description)
- [Keeping a profile as installed](#keeping-a-profile-as-installed)
- [Retiring the original](#retiring-the-original)
- [Working as a team](#working-as-a-team)
- [Where legacy and declarative genuinely disagree](#where-legacy-and-declarative-genuinely-disagree)
- [The registry corrects itself](#the-registry-corrects-itself)

## Verdicts and readiness

The blueprints API does not accept every Apple MDM payload. It keys on a fixed
registry, and a payload type outside it comes back as:

```json
{"httpStatus":400,"errors":[{"code":"VALIDATION_FAILURE",
  "field":"steps[0].components[2].configuration",
  "description":"Failed to validate configuration."}]}
```

No payload type, no key, no reason — byte for byte what a wholly invented
payload type returns. Blueshift answers the question up front. Every payload
gets a verdict before anything is sent:

| Verdict | Meaning |
|---|---|
| **Native DDM** | Promoted to a declarative component — applies without a profile install, reports status back |
| **Supported** | In the registry; passes through inside a configuration-profile component |
| **Wrapped** | Not in the registry; delivered as managed preferences instead, settings preserved. Apple payloads that aren't preference domains (accounts, say) are never wrapped: they move as declarations or are blocked |
| **Blocked** | Explicitly refused. It would reject the *entire* blueprint, so Blueshift leaves it out and says so |

The verdicts roll up into one readiness per profile, which the Overview counts
and the Profiles list sorts by:

| Readiness | Meaning |
|---|---|
| **Declarative upgrade** | Something becomes a native declarative component |
| **Clean migration** | Every payload is accepted; nothing lost, nothing promoted |
| **Blueshift-editable** | Migrates and deploys, but Jamf Pro's blueprint editor cannot open the values — Blueshift's Edit Values (or the API) can |
| **Partly blocked** | Some payloads are refused and left out; the rest migrate |
| **Cannot migrate** | Every payload is refused |
| **Unreadable** | Signed, empty, or not a property list |

## What it handles

- **Any source.** Jamf Pro, or a `.mobileconfig` dragged in from disk —
  exported from another MDM or hand-written.
- **Mac and mobile.** macOS, iOS and iPadOS profiles go through the same
  registry and converters.
- **74 supported and 22 blocked payload types**, probed against the API rather
  than taken from the documentation.
- **Unknown types are wrapped, never dropped.**
- **The one-character failure.** Jamf Pro writes User Preferences as
  `com.apple.preferences.users`; the API only accepts
  `com.apple.preference.users`. Blueshift rewrites it.
- **Declarative promotion** for passcode policies, Safari settings,
  software-update deferrals and Rapid Security Response.

## One integration, through the Jamf API gateway

Everything goes through the Jamf API gateway at
`{us|eu|apac}.api.jamfcloud.com` with one Jamf Account integration created at
the Platform environment level:

| | Served by | Path on the gateway |
|---|---|---|
| Blueprints, device groups, devices, declaration reports | **Jamf Platform API** | `blueprints/v1/…`, `device-groups/v1/…`, `devices/v1/…`, `ddm/report/v1/…` |
| Configuration profiles and categories | Jamf Pro **Classic API**, routed by the gateway | `proclassic/…` |
| Jamf Pro version, server URL, inventory | **Jamf Pro API**, routed by the gateway | `pro/v1/…` |

Every request carries the environment ID — not the Jamf Pro tenant ID — in an
`X-Environment-Id` header. Profiles read through the gateway are byte for byte
what the tenant's own URL returns, and the cleanup writes behave identically.
The Jamf Pro host, used for **Open in Jamf Pro** links, is read from the tenant
on every connect.

A client belongs to one region, and the wrong region answers `invalid_client` —
the same error as a mistyped secret — so Blueshift tries the other two and
switches to the one that accepts the credentials.

Migration history is kept per Jamf environment, because profile IDs are only
unique within a tenant.

## The screens

| Section | What it is for |
|---|---|
| **Overview** | Where the migration stands. Every number opens the rows behind it |
| **Profiles** | Every configuration profile, its scope, and what each payload becomes |
| **Blueprints** | What exists on the Platform, its scope and its rollout |
| **Migrations** | What moved, when, into which blueprint, and what is left to retire |
| **Devices** | What every blueprint is doing on one machine |
| **Activity** | Everything the app has done, from the unified log |
| **Payload Registry** | Which payload types this tenant accepts |
| **Help** (⌘?) | Getting started, every screen, concepts and troubleshooting |

**Migrating many at once.** **Select Ready** above the Profiles list selects
every Declarative upgrade, Clean migration and Blueshift-editable profile not
yet migrated. The batch sheet names the blueprints, carries each profile's scope
over (or applies one set of groups to all), creates them one at a time, and ends
with **Retry N Failed** and **Clean Up Originals…**. Carrying scope over only
takes scopes that carry exactly; a profile with exclusions, limitations or
entries a blueprint cannot express is skipped, to be migrated on its own.

**Migrate for all, edit for selective.** Blueshift edits only the values Jamf
Pro's blueprint editor cannot open (managed preferences such as Jamf Connect,
and payload types Jamf Pro does not render), on any blueprint; everything else
shows **Edit in Jamf Pro**, so each setting has one place to be edited. A
Declarative upgrade or Clean migration is edited in Jamf Pro, a
Blueshift-editable one in Blueshift, and a Partly blocked one payload by
payload. Scope and condition stay editable in Blueshift on blueprints it
migrated.

**Editing values as plists.** Edit Values shows preference domains as a form or
as a plist: import a `.plist` or `.mobileconfig` (replace or merge), edit XML,
copy it out. **Review & Save** lists every change before it is written.

**After a migration.** **Show Blueprint** opens it; **Undo this migration**
removes the blueprint just created and leaves the profile untouched.

**Readiness report.** File ▸ Export Readiness Report (⇧⌘E) saves one CSV row
per profile.

**Keyboard.** ⌘1–⌘7 move between sections and the arrow keys through every list.
The Dock icon counts migrations still waiting for cleanup.

Blueshift does not deploy a migration. Deploying is done in Jamf Pro, where it
is recorded against your change process.

## Scope: comparison, smart groups and "All" scopes

Blueprint scope is a list of device groups and nothing else. A Jamf Pro scope
can also name individual devices, departments, buildings and users, narrow the
result with limitations, and subtract exclusions. Dropping any of that fails
silently — the blueprint reaches machines the profile never did, or misses ones
it did.

**Create Blueprint compares the scope line by line.** A table lists every entry
— target, limitation or exclusion — beside what it becomes in the blueprint as
is and in a suggested smart group: ✓ exact, ≈ approximate, ✕ not possible.
A summary counts each, and one recommendation follows: use the matching groups
when everything carries, or build a smart group when it does not. Groups are
pre-selected only for an exact carry — the matching groups of a scope with
exclusions would reach devices the profile never did.

**Build a smart group for this scope** proposes one group whose criteria say
what the whole scope said:

```
( Computer Group member of "Finance Macs" or Serial Number is "DEMO00000000" or Department is "Operations" )
and Computer Group not member of "Loaners"
```

Each criterion is shown before anything is created. User limitations become
the user assigned in inventory and are marked approximate; network segments,
iBeacons and user groups are listed as impossible. The group is created on the
Platform only when you click **Create Smart Group** (Device Groups — Create),
then selected as the blueprint's scope. Individual devices are written as their
serial number when the name matches exactly one device. Mobile profiles get a
mobile device group with the equivalent criteria.

**"All" scopes.**

| Profile scope | Carried as |
|---|---|
| All Computers | Jamf Pro's built-in **All Managed Computers** (no criteria — the same devices), or **All Managed Clients**, noted as leaving out macOS Server. Blueshift reads the group's criteria and warns if it has been edited |
| All Mobile Devices | The built-in **All Managed iPads + iPhones + iPod touches + Apple TVs + Vision Pros + Watches** together — blueprint scope is a union |
| All Users | "Username is not empty" — devices with an assigned user, marked approximate |

**Nesting is measured before anything is created.** A "member of" criterion
nests the referenced group, and Jamf Pro evaluates every level on each
inventory update. The builder follows each referenced group's criteria through
the Platform, shows the deepest chain and how many smart and static groups it
relies on, warns from four levels, flags groups the Platform does not know, and
refuses a name that already exists. Jamf recommends nesting smart groups only
when necessary: while groups recalculate, devices can briefly drop out and back
in, which can remove and reinstall what is scoped — most noticeably on mobile
devices.

## The blueprint's description

**Settings ▸ Migration Record** chooses what every migrated blueprint's
description says — a checkbox for each fact (original name and ID, category,
date, time, who migrated it, email, department, Mac and serial number, API
client, Jamf Pro server, scope carried and not carried, components) and a
custom note such as a change number, with a live preview. Name, email and
department can be typed in; left empty, they come from the user Jamf Pro
inventory assigns to this Mac, found by serial number.

A short `[blueshift v1 …]` tag is always added at the end. It is how Blueshift
on any Mac recognises the blueprint as a migration.

## Keeping a profile as installed

**Convert is the default, and the right choice for a large fleet.** A kept profile
has the same identity as its original, so once the original is unscoped or deleted
Jamf Pro sends each device a Remove command it refuses, and can keep retrying, so
failed commands can pile up. Blueshift warns about this, and can't prevent it.

**Convert** builds a new blueprint beside the profile, so devices install
something new. **Keep as Installed** uses Apple's transition from MDM: when the
blueprint is deployed, each device that has the same profile takes it over where
it is, with no reinstall. Nothing becomes declarative, and every payload stays as
it is.

Blueshift sends the profile's own identifiers, as one configuration-profile
component, and reads the blueprint back to check they survived. A profile
qualifies only when they can: for example, a payload whose `PayloadUUID` differs
from its `PayloadIdentifier` can't, because the API forces them to be equal.
Blueshift names the reason. In a batch, a profile that doesn't qualify is skipped,
never quietly converted. Imported files don't offer this route, because no device
got them from Jamf Pro.

What we saw on a Mac and a supervised iPhone (iOS 26.6 and 27.2):

| Step | Result |
|---|---|
| Deploy | The blueprint reports success with no install command. On the Mac, the install date didn't change |
| Retire the original | Refused until the blueprint has succeeded with none failed or pending, then allowed |
| After retiring | Each device refuses Jamf Pro's Remove Configuration Profile, and the settings stay. A Mac says "not installed by the MDM server"; an iPhone says "managed by Declarative Device Management" |
| Rename the original after adoption (Mac) | Jamf Pro logs a failed Install Configuration Profile, and the device keeps its name |
| Edit a value and redeploy (Mac) | The same profile updates. Its identifier stays, its top-level UUID and install date change, and any setting the API dropped goes |
| Leave scope, original retired | The profile and its settings come off |
| Leave scope, original still scoped (Mac) | The profile comes off, then returns from Jamf Pro at the next inventory update. The settings are missing until then |

**The retire check has a limit.** It reads the blueprint's own rollout, which
says nothing about devices outside its groups. If a profile reaches 1,000 Macs and
the blueprint has succeeded on a test group of 5, retiring the original strips the
profile and its settings from the other 995. Blueshift warns about this rather
than blocking, because different groups can hold the same devices and comparing
group names would refuse valid setups. Make sure the blueprint's groups cover the
profile's scope first.

iPad and Apple TV haven't been tested, so try those on a test group first.

## Retiring the original

Until the configuration profile stops managing devices, two things manage the
same settings and which one wins is not predictable.

Blueshift's default is not to delete it — deleting destroys the only record of
what the old configuration was, at the point you are least sure the replacement
is right. The recommended action **renames and unscopes** it and files it in the
**Migrated by Blueshift** category: it reaches no device, still reads as
superseded in Jamf Pro, and can be put back by re-scoping. Deleting is offered
last, behind its own confirmation. An original that is already retired is
offered only **Delete** — retiring it again would change nothing.

On every route, unscoping or deleting an original that still reaches devices is
refused until the blueprint's rollout confirms it: deployed, at least one device
succeeded, none failed or pending. A rollout that can't be read counts as a no.

From a migration you can also **Show the profile** or **Open the profile in
Jamf Pro** while it still exists.

A profile [kept as installed](#keeping-a-profile-as-installed) follows stricter
rules, because retiring it on a device that hasn't adopted it removes its settings.

## Migration Evidence

A blueprint's report gives counts. Evidence answers the question that matters
before retiring a profile: *did the blueprint reach every device that had it?*

**Check Profile.** Just before a blueprint is created (one profile or a batch),
Blueshift reads Jamf Pro inventory and records which devices list the profile,
matched by its PayloadIdentifier or UUID (by name only when neither is known,
and flagged). It also resolves the profile's scope to devices: computers,
groups, buildings, departments and users, minus exclusions. Limitations, user
groups, network segments and iBeacons can't be resolved from inventory, so a
scope that uses them is marked approximate. **Check Profile** on a profile
does the same any time.

Each profile check is kept as a new file, never rewritten:
`~/Library/Containers/com.jamf.blueshift/Data/Library/Application Support/Blueshift/Evidence/<environment>/<type>-<id>/`.
Check Blueprint uses the latest profile check taken before the migration.

**Check Blueprint.** Every device that had the profile is matched by serial number to
the blueprint's per-device declaration status and to current inventory:

| Verdict | Meaning | Blocks retiring |
|---|---|---|
| Verified | Blueprint applied; after retiring, original gone (Convert) or still present (Keep as Installed) | No |
| Pending | Blueprint not reported yet | Yes |
| Failed | Blueprint failed on the device | Yes |
| Not covered | Device had the profile but isn't in the blueprint's scope | Yes |
| Original still installed | Converted and retired, fresh inventory still lists the original | No (already retired) |
| Inventory not updated | Inventory predates the retirement | No |
| New device | In the blueprint's scope, never had the profile | No |

The retire gate runs this check after the count check. Migrations whose
profile wasn't checked first (before 1.8) keep the count check alone.

**Reports.** Each Evidence or profile check exports as PDF, Excel (.xlsx) or CSV.
**File ▸ Export Pre-Migration Report** lists every profile with who has it and
who its scope reaches; **Export Migration Report** lists every migration with
its evidence, plus a sheet of every device line. Cells a spreadsheet would run
as a formula are neutralised.

## Dry Run

Dry Run is enforced where every request leaves the app, not in the views: the
two functions all Jamf traffic passes through answer any write locally and
record it. `Tools/verify.sh` fails the build if any request path skips them.

Creating a blueprint is the exception that makes Dry Run useful: the real
request is sent with an extra component the API always refuses (a blank
display name, the same marker the registry probe uses). The API validates the
whole request and rejects it, so nothing is created. If the marker is the only
error, the real request would be accepted; anything else is the API's own
rejection, shown as it would be live. A smart group invented earlier in the
same Dry Run is swapped for a real group while validating.

Simulated blueprints and migrations stay in memory. The ledger, the shared
folder and the launch snapshot are never written while Dry Run is on, and
turning it off discards everything it simulated.

## Client secret expiry

No Jamf Platform API returns an integration's expiry date, so it's entered
once in Connections. It's stored per environment and client ID, warns in the
main window inside the chosen reminder window, and turns a refused secret after
that date into an "expired" message.

## Working as a team

**Which profiles have been migrated is shared automatically.** Every blueprint
Blueshift creates carries a tag naming the profile it replaced and who migrated
it, so any admin who connects to the tenant sees the same picture.

**Cleanup history is shared by choice.** When each original was renamed,
unscoped or deleted is not in the tenant, so it travels as a `.blueshift`
snapshot (**File ▸ Save / Open**) or through a shared folder both Macs can reach
(**Settings ▸ Shared Ledger**). Each Mac writes one file named after itself and
reads the others:

```
<shared folder>/Blueshift/
  ledger-Deployment-Mac.blueshift
  ledger-Helpdesk-iMac.blueshift
```

No Mac writes another's file, so there is no lock and no lost write. Merging is a
union of facts — the earliest known timestamp wins for each field — so it is
order-independent and repeatable. The ledger is backed up automatically before
anything rewrites it (**File ▸ Show Backups in Finder**).

## Declarations for newer OS versions

Blueprints refuses some profile payloads outright, and Apple replaced others
with declarations in macOS and iOS 26.4 and 27. Blueshift sends these as
**Custom Declarations**, mapping keys from Apple's schemas
([apple/device-management](https://github.com/apple/device-management),
Release-v27.2; `Tools/schema/generate.py` regenerates the table).

| Profile payload | Becomes | Needs |
|---|---|---|
| CalDAV, CardDAV, subscribed calendar, LDAP, Mail, Exchange (EAS, EWS) | Account declarations, user channel | macOS 13, iOS 15 |
| Extensible SSO | Extensible SSO declaration | macOS and iOS 27 |
| DNS Settings | `network.dns-settings` | macOS and iOS 27 |
| VPN: IKEv2 with no machine authentication, VPN app with a password | `network.vpn.ikev2`, `network.vpn.vpn-plugin` | macOS and iOS 27 |
| Web content filter, filter app | `webcontent-filter.plugin` | macOS and iOS 27 |
| DNS Proxy, Relay, Content Caching | Their declarations on **Upgrade**; kept as profiles by default, since the profile still works everywhere | 27 |
| Apple Intelligence and keyboard restrictions | `intelligence.settings`, `keyboard.settings`, on opt-in; the rest of Restrictions stays | 26.4 |

On a Mac on macOS 27.3 (2026-11-12), these reported successful: the Mail,
CalDAV, CardDAV, LDAP, subscribed-calendar and Exchange (EWS) accounts, Extensible
SSO, DNS Settings, an IKEv2 VPN, Content Caching, and the Apple Intelligence and
keyboard declarations. Not yet tested on a device: DNS Proxy and Relay as
declarations, the VPN app and filter app (they need the vendor's app), and iOS.

**What never moves.** Passwords, shared secrets and certificates: a declaration
takes them only as assets, which Blueshift can't create. Accounts ask the user to
sign in instead. A VPN needing a secret or certificate, IPSec, weak IKE
algorithms, a VPN app whose settings don't fit the declaration's four
`VendorConfig` keys (Jamf Trust, for example), and Apple's built-in content
filter are blocked with the reason, because they would deploy and not work.

**Where it applies.** Each payload shows where Apple supports it. A payload in a
profile for the wrong kind of device is marked **Wrong platform**; one Apple has
deprecated or removed says so (macOS 27 deprecates DNS Settings, DNS Proxy,
Relay, Passcode and Content Caching profiles, and removes Software Update).
**Create Blueprint** counts the devices in the chosen groups that are below a
declaration's minimum OS and adds that floor to the suggested activation
condition.

**Jamf Pro variables.** Jamf Pro fills in `$EMAIL`, `$UDID` and the like for
profiles. Nothing confirms Blueprints does inside a custom declaration, so any
declaration carrying one is flagged for a check on a test device.

## Settings the API drops

Inside a configuration-profile component, the blueprints API keeps only the
keys Apple's schema defines for the payload and drops the rest without an error
(live, 2026-11-12: the Dock's `contents-immutable` stays, but
`show-process-indicators-immutable` and Finder's `InterfaceLevel` go). The same
keys inside a Managed Preferences wrapper all survive, so Blueshift sends
out-of-schema keys beside the payload as managed preferences for the same
domain. Jamf Pro's `-ProfileNumber` bookkeeping keys are left out.

Keys inside a group can't travel separately. A profile made in Jamf Pro's
Energy Saver UI writes `Sleep On Power Button` and `Disk Sleep Timer-boolean` in
each power group; the API drops both, and Blueshift names them.

## Where legacy and declarative genuinely disagree

- **Passcode policies that constrain without forcing.** The declarative
  component requires `RequirePasscode` to be present and true whenever any other
  passcode field is managed. A legacy policy carrying only `maxFailedAttempts`
  cannot be promoted without starting to require a passcode, so Blueshift keeps
  it legacy and says why.
- **Empty lists.** Jamf Pro stores `AllowList: []`; the API rejects it and fails
  the whole blueprint. They manage nothing either way, so they are dropped and
  reported.
- **Missing required keys.** A payload in the registry can still be refused for
  a key it lacks. `com.apple.screensaver` without `moduleName` is known, so it is
  sent as managed preferences from the start. Others name the payload when the
  API refuses them, and Blueshift offers to re-send it as managed preferences.
- **One payload type, several settings.** Jamf Pro writes Energy Saver, Date &
  Time, Accounts and Mobile Accounts all as `com.apple.MCX`. The API refuses that
  spelling and takes each group by name (`com.apple.MCX.EnergySaver` and so on),
  without checking which keys travel under which name. Blueshift sorts the keys
  into their groups and sends each under its own name. FileVault keys are refused
  by the API; Wi-Fi keys have no name it takes, so they stay managed preferences.
  Keep as Installed can't rename a type, so these profiles use Convert.
  Energy Saver values stay on a Mac after the payload is removed, whether the
  profile is retired or the blueprint undeployed (checked on macOS 26.6), so
  set the values you want back before removing it.
- **Extensible SSO.** The API refuses the `com.apple.extensiblesso` payload, but
  takes Apple's `com.apple.configuration.extensible-sso` declaration as a custom
  declaration. Blueshift maps the payload onto it, Platform SSO included (macOS
  only), and checks it first, because the API doesn't check a custom
  declaration's contents. It needs macOS, iOS, iPadOS or visionOS 27: an earlier
  device reports "Unknown Declaration Type" and counts as failed, which also keeps
  the retire check from removing the original profile it still relies on.

## The registry corrects itself

Jamf enables payload types between releases, so a built-in list goes stale in
the direction that hurts. **Payload Registry → Re-check against this tenant**
asks the tenant directly, in one request that creates nothing: validation
reports every bad component at once, and a refused type has its own signature
(`Payload disabled: <type>`), so one blueprint carrying one component per
candidate type classifies the whole registry. A deliberately invalid sentinel
component guarantees the request can never succeed — a probe never leaves a
blueprint behind — and its error must come back in the reply, or the result is
discarded rather than trusted.
