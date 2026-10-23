# How Blueshift works

[← Back to the README](../README.md)

- [Verdicts and readiness](#verdicts-and-readiness)
- [What it handles](#what-it-handles)
- [One integration, through the Jamf API gateway](#one-integration-through-the-jamf-api-gateway)
- [The screens](#the-screens)
- [Scope: comparison, smart groups and "All" scopes](#scope-comparison-smart-groups-and-all-scopes)
- [The blueprint's description](#the-blueprints-description)
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
| **Wrapped** | Not in the registry; delivered as managed preferences instead, settings preserved |
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

From a migration you can also **Show the profile** or **Open the profile in
Jamf Pro** while it still exists.

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
  a key it lacks (`com.apple.screensaver` without `moduleName`). These name the
  payload, so Blueshift offers to re-send it as managed preferences.

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
