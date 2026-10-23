# Blueshift FAQ

[← Back to the README](../README.md)

**Does Blueshift deploy blueprints?**
No. Migrating creates the blueprint and stops there; deploying is done in Jamf
Pro, where it is recorded against your change process. The Deploy and Undeploy
buttons on a blueprint are optional and need their own permission.

**Will it delete my configuration profiles?**
Not unless you ask. The recommended cleanup renames the original, unscopes it
and files it in the **Migrated by Blueshift** category, so it reaches no device
but can be restored by re-scoping. Deleting is offered last, behind its own
confirmation, and needs Configuration Profiles — Delete.

**I migrated the wrong thing. Can I undo it?**
Yes, straight away: the result sheet has **Undo this migration**, which removes
the blueprint just created. The profile is untouched either way.

**Does it need a Jamf Pro API client or password?**
No. One Jamf Account integration reaches both the Jamf Platform API and your Jamf
Pro tenant through the Jamf API gateway. See
[How Blueshift works](HOW-IT-WORKS.md#one-integration-through-the-jamf-api-gateway).

**What does Blueshift send, and where?**
Every request goes to the Jamf API gateway for your region
(`{us|eu|apac}.api.jamfcloud.com`). When an editor needs the schema for a
declarative type, Blueshift also reads Apple's public `device-management`
repository on GitHub; that request carries nothing about your tenant.
Credentials stay in your login keychain and logs stay on the Mac.

**Why does the window open with data before it has connected?**
Blueshift keeps a copy of the profiles and blueprints it last read and opens on
it while it reconnects. Nothing can be changed until the connection is live. The
copy is readable only by your account; because profile payloads can contain
secrets such as Wi-Fi passwords, you can turn it off under **Settings ▸ Jamf ▸
Open on the last saved data**, which also deletes it.

**What happens to a payload type Blueshift doesn't recognise?**
It is wrapped and delivered as managed preferences, so its settings are kept.
Only types the API explicitly refuses are left out, and each is named first.

**Why must every blueprint have a device group?**
The Platform API refuses to create or save a blueprint without one. To keep one
unscoped, remove its groups in Jamf Pro afterwards.

**How is scope carried over?**
Create Blueprint compares every entry of the profile's scope with what a
blueprint can express, and recommends either the matching device groups or one
smart group. See
[Scope](HOW-IT-WORKS.md#scope-comparison-smart-groups-and-all-scopes).

**Is building a smart group safe?**
It is created only when you click **Create Smart Group**, and nothing is
deployed. Before that, Blueshift shows every criterion, what is approximate or
impossible, and how deeply the group would nest. Jamf recommends nesting smart
groups only when necessary.

**Can Blueshift edit any blueprint?**
It edits only the values Jamf Pro's editor cannot open, on any blueprint;
everything else has **Edit in Jamf Pro**. It also edits scope and condition on
blueprints it migrated.

**Can I still edit a migrated blueprint in Jamf Pro?**
Usually. Profiles marked **Blueshift-editable** are the exception: they deploy
normally, but Jamf Pro's blueprint editor cannot open their values. Use **Edit
Values** in Blueshift, or the API.

**Can several admins use it on the same tenant?**
Yes. Which profiles have been migrated — and by whom — is read from the
blueprints themselves. Cleanup history can be shared through a `.blueshift` file
or a shared ledger folder.

**I use more than one Jamf tenant. Does history get mixed up?**
No. Migration history is kept separately for each Jamf environment.

**Does it handle iPhone and iPad profiles?**
Yes, through the same registry and converters as macOS profiles.

**Can I migrate a `.mobileconfig` from another MDM?**
Yes. Drag it in. A signed profile cannot be read, so export it unsigned first.

**Can I try it without a tenant?**
Yes: `open -n /Applications/Blueshift.app --args -BlueshiftDemo YES` opens a
fictional tenant and never contacts Jamf.

**What does it run on?**
macOS 14 or newer.

**Is there a Windows version?**
No. See [Windows users](WINDOWS.md) for jamf-cli.

**Is it a supported Jamf product?**
No. Blueshift is a Jamf Concepts app, offered under the [Jamf Concepts Use Agreement](../LICENCE.md). Report problems in
[GitHub Issues](https://github.com/Jamf-Concepts/blueshift/issues).
