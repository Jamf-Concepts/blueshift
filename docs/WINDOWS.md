# Windows users

[← Back to the README](../README.md)

Blueshift is a Mac app; there is no Windows version. On Windows, use
[jamf-cli](https://github.com/Jamf-Concepts/jamf-cli), Jamf Concepts'
open-source command line for the Jamf Platform, which can do the same
migration:

```bash
# Migrate one configuration profile to a blueprint, by Classic API ID or name
jamf-cli pro blueprints import-profile 42
jamf-cli pro blueprints import-profile --name "Passcode Policy" --type computer
```

`import-profile` downloads the profile from Jamf Pro, promotes passcode, Safari,
software-update and Rapid Security Response settings to declarative components,
wraps the rest, skips payload types blueprints refuse, and carries target
computer and mobile device groups over as scope. Exclusions, limitations and
individual device assignments are not carried over.

jamf-cli also has a built-in **MCP server**, so an AI assistant can plan and run
the migration in conversation:

```bash
jamf-cli mcp serve        # speaks MCP over stdio; add it to your AI client's MCP servers
```

jamf-cli's releases publish macOS and Linux builds, so on Windows either build
it with Go (`go install github.com/Jamf-Concepts/jamf-cli/cmd/jamf-cli@latest`)
or run the Linux build under WSL. Set up credentials with `jamf-cli pro setup`;
see the [jamf-cli wiki](https://github.com/Jamf-Concepts/jamf-cli/wiki).

What jamf-cli does not have is Blueshift's interface: the readiness overview,
per-payload verdicts before sending, the scope comparison and smart group
builder, batch migration with retry, retiring originals, and the shared ledger.
