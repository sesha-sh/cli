# Sesha CLI

[Documentation](https://sesha.sh/docs) · [CLI reference](https://sesha.sh/docs/cli) · [MCP setup](https://sesha.sh/docs/mcp)

Publish HTML, Markdown, images, PDF, JSON, and other artifacts to your space on [Sesha](https://sesha.sh). Node.js 22 or newer is required. Sesha is a hosted service with free public signup; no server clone or Cloudflare account is needed.

```sh
npx @sesha.sh/cli@0.9.0 login
npx @sesha.sh/cli@0.9.0 upload report.html
```

With pnpm:

```sh
pnpx @sesha.sh/cli@0.9.0 login
pnpx @sesha.sh/cli@0.9.0 upload report.html --title "Weekly report"
```

Or install once with `npm install --global @sesha.sh/cli@0.9.0`, then run `sesha`.

## Connect a terminal

`login` opens a browser approval page. Check its code, target space, permissions, and credential expiry before approving. The credential is generated locally; the approval API only sees its hash. Login output never contains the token.

Credentials use your OS credential vault. On Linux, a persistent Secret Service store is required. A headless POSIX machine can explicitly use `sesha login --no-browser --storage file`; this keeps its credential in a directory with 0700 permissions and file with 0600 permissions. File mode is not available on Windows; use Credential Manager there. `SESHA_CONFIG_DIR` overrides the configuration directory.

```sh
sesha auth status
sesha whoami
sesha logout
```

`sesha whoami` shows your space name and readable permissions:

```text
✓ Connected to Sesha

  Space     Djamil
  Access    Read artifacts · Upload and edit
```

Use `sesha whoami --json` for the space ID, credential ID, and scope values. Use `--json` in scripts and pipelines too.

Logout revokes the connection on Sesha before removing the local credential. Connections are listed under **Agent access → Connected devices**, with their name, permissions, last activity, and expiry. Remove a device there to revoke access immediately; its artifacts remain available. Credentials expire after 90 days by default; `login --expires-at ISO_DATE` chooses a date within the next year.

## Connect an agent

Use a separate profile for each agent or harness. Setup defaults to upload-only access and returns a profile-based upload command, without a token to paste into configuration.

```sh
sesha agent setup --profile t3 --name "T3 agent" --json
sesha upload visualizer.html --profile t3 --json
```

Set `SESHA_PROFILE=t3` in the harness environment to use that profile automatically. Use `--scopes artifacts:read,artifacts:write` during setup if the agent needs to list or inspect uploads. Existing resource commands remain available; run `sesha --help` and `sesha upload --help` for examples.

For CI without an explicitly selected profile, use `SESHA_TOKEN` and optional `SESHA_ENDPOINT`. Explicit `--profile` or `SESHA_PROFILE` selects its saved credential even when a token is inherited in the environment. Tokens are not accepted as command-line flags. `--endpoint` supports another Sesha instance; stored credentials are never sent to a different endpoint.

## Upload and share

```sh
sesha upload report.html --title "Weekly report" --request-id weekly-2026-11-09
cat notes.md | sesha upload - --filename notes.md --content-type text/markdown
sesha list --limit 20
```

Uploads show connection, reading and transfer progress on stderr, then keep a spinner running while Sesha prepares the share link. Percentages count multipart request bytes; completion is confirmed only after the service returns the artifact. `--json` disables progress.

All commands show readable results and errors by default. Pass `--json` for structured results and errors, including in scripts and pipelines. Login instructions and errors use stderr. Return the upload’s `url` to the user. Reuse a request ID only for retries of the exact same file and metadata. Files may be up to 25 MiB on Free or 100 MiB on Personal and inherit your saved expiration defaults. HTML should be self-contained. Keep secrets out of shared artifacts.

## Upload several files, or a folder

Pass more than one file to publish each as its own artifact:

```sh
sesha upload report.html notes.md chart.png
```

`--title`, `--description`, `--content-type` and stdin apply to a single upload. With `--request-id`, each file gets that ID plus its position, so repeating the command adds nothing twice. One bad path does not stop the others; the command reports it and exits non-zero.

To keep files together under one link, gather them into a folder:

```sh
sesha upload ./dist --folder "Launch site"
sesha upload ~/Desktop/logo.png ./exports/deck.pdf --folder "Launch kit"
sesha upload ./photos --into FOLDER_ID_OR_LINK --prefix images
```

- `--folder NAME` creates a folder. `--into FOLDER` adds to one that exists; it takes a folder ID or its share link.
- A file is stored under its own name. A directory contributes its contents and keeps their layout, so `./dist/index.html` becomes `index.html`.
- `--prefix PATH` places everything from that command in a subfolder. Two sources that would land on the same path are refused before anything is sent.
- Hidden files and `node_modules` are skipped unless you pass `--include-hidden`. Symbolic links are never followed.
- A folder holds up to 100 files and 100 MiB total, with 25 MiB Free and 100 MiB Personal per-file limits.
- Unchanged content keeps its stored copy. Small files can be skipped before transfer; large files stream once and verify their digest. If an upload is interrupted, repeat the same command with the same `--request-id`, or use `--into` with the folder ID from the output.

## Use with an AI assistant (MCP)

Assistants such as Claude Code, Claude Desktop, Cursor and Codex can publish to Sesha through MCP. You do not start anything yourself: the assistant runs `sesha mcp` as a local server when it needs it and signs in with a profile you approved.

```sh
sesha agent setup --profile my-agent
claude mcp add --env SESHA_PROFILE=my-agent --transport stdio sesha -- npx -y @sesha.sh/cli@0.9.0 mcp
```

For Claude Desktop or Cursor, add the same command to the assistant's MCP settings file:

```json
{
  "mcpServers": {
    "sesha": {
      "command": "npx",
      "args": ["-y", "@sesha.sh/cli@0.9.0", "mcp"],
      "env": { "SESHA_PROFILE": "my-agent" }
    }
  }
}
```

Without further setup the assistant can publish what it writes. To let it upload existing files, also set `SESHA_UPLOAD_ROOTS` to a JSON array of allowed directories; files outside them are refused. A hosted MCP endpoint is not provided yet.

This package contains only the service client. Server implementation and deployment sources are not distributed with it.

New terminal and agent connections report their device category, OS, and model when available. Sesha shows these details with the last service-observed IP in Agent access. MAC addresses and hardware serial numbers are not sent. Older connections retain a generic terminal icon until reconnected with this client.


## Keep iterating at the same link

When revising an artifact you already uploaded, keep its ID and replace its content rather than creating a new artifact:

```sh
npx @sesha.sh/cli@0.9.0 artifacts get ARTIFACT_ID --json
npx @sesha.sh/cli@0.9.0 artifacts replace ARTIFACT_ID --file report.html --revision 1 --request-id revision-2-attempt --json
npx @sesha.sh/cli@0.9.0 artifacts versions ARTIFACT_ID --json
```

Use the current `revision` returned by `get`; keep the request ID and content unchanged when retrying. A revision conflict means someone updated the artifact: read it again before deciding to replace it. Agents need both `artifacts:read` and `artifacts:write` for this workflow. The latest content stays at `/a/ID`; a retained version is linked as `/a/ID?version=N`.

Free retains three versions including the current file. Personal has no version-count cap within its storage allowance. Older versions can be unpublished and republished; their bytes still count toward storage. Deleting an older version is permanent and requires delete permission. The current version cannot be unpublished or deleted on its own. The viewer's version picker is on by default; hiding it removes browsing, while existing pinned links still work unless that version is unpublished or deleted. Password and expiration rules apply to every version. CLI 0.8.0 publishes folder batches as atomic revisions. Unchanged assets share stored content; a failed batch keeps the previous folder unchanged. Restore a complete revision with `sesha artifacts restore ID --target-revision OLD --revision CURRENT --request-id STABLE_ID`. Current sharing settings are preserved; restores that remove paths also require delete permission.

MCP exposes `sesha_list_versions`, `sesha_replace_file`, `sesha_set_version_visibility`, `sesha_restore_version` and `sesha_delete_version`. File paths must remain within the configured upload roots.


## Artifact activity

The private activity feed records uploads, replacements, settings changes, version publication/deletion, folder edits and requested AI metadata generation. View it in artifact settings or run:

```sh
npx @sesha.sh/cli@0.9.0 artifacts activity ARTIFACT_ID
npx @sesha.sh/cli@0.9.0 artifacts activity ARTIFACT_ID --json
```

The API is `GET /v1/artifacts/:id/activity?limit=25&cursor=...`; MCP exposes `sesha_list_activity`. Both require `artifacts:read` in the same space, including when reading a deleted artifact's retained activity. Public share links never include activity or actor identities.

Browser actions identify the signed-in person. Named agent and terminal connections identify the credential used and the person who authorized it; generic API tokens remain labeled as tokens. Give each agent a separate connection, for example `npx @sesha.sh/cli@0.9.0 agent setup --profile codex --name "Codex" --scopes artifacts:read,artifacts:write`. Shared credentials cannot distinguish programs or prove a particular model executed an action.

Activity keeps names as they were at the time, even after a connection is renamed or revoked. It stores changed field names, not password values or file contents. Tracking starts when this feature is deployed; earlier operations are not reconstructed. Activity remains with the artifact, including its deletion record, and is removed when its space/account is deleted. Visitor analytics are separate from this change history.


## Agent skill

Included in every Sesha CLI release.

The CLI bundles the Sesha publishing skill. It teaches profile/space selection, stable artifact updates, inherited sharing settings, partial-failure handling and verification. Install it for your assistant, then reload the assistant:

```sh
sesha skill install --agent codex --global
sesha skill install --agent claude-code --global
sesha skill install --agent cursor --global
```

Omit `--global` for the current project, or use `--path /absolute/path/to/skills/sesha`. Preview with `--dry-run`; local edits require review before `--force`. Installation does not connect an account, grant publishing permission, or modify MCP settings. `sesha skill show` prints the bundled entrypoint. The npm package also contains `skills/sesha/`.

Authorize a dedicated profile separately with `sesha agent setup --profile my-agent --scopes artifacts:read,artifacts:write`. Connected MCP tools use their configured profile. See [skill installation](https://sesha.sh/docs/skill) and [MCP setup](https://sesha.sh/docs/mcp).

Use `sesha artifacts files FOLDER_ID --profile my-agent --json` to inspect folder paths, sizes and digests. Folder batches publish atomic, recoverable revisions. Pinned links keep HTML and supporting assets together. Legacy folders acquire a baseline at their first atomic publication; earlier overwritten content cannot be recovered.

## Upgrading from Seshat

Sesha is the new name for Seshat. Install `@sesha.sh/cli@0.9.0` and use `sesha`; the `seshat` executable remains an alias. Existing profiles and OS credentials are read automatically, and Sesha saves future changes in its own configuration directory. Existing tokens and links remain valid. `SESHA_*` environment variables take precedence; their `SESHAT_*` predecessors remain supported. Stored credentials stay bound to the endpoint you approved.

## Agent skills and plugins

Plugin 0.9.0 bundles CLI 0.9.0.

This public distribution contains the MIT CLI release, the canonical Sesha skill and plugin manifests. The Sesha application remains private. Bundles are built in the application release pipeline; this repository distributes those reviewed outputs.

Install the skill for your selected agent:

```sh
npx skills@1.8.1 add sesha-sh/cli --skill sesha --agent codex
# Or choose --agent claude-code
```

Claude Code marketplace:

```text
/plugin marketplace add sesha-sh/cli
/plugin install sesha@sesha
```

Codex marketplace:

```sh
codex plugin marketplace add sesha-sh/cli
codex plugin add sesha@sesha
```

Start a fresh agent session after installation. This is a skill-only plugin requiring local execution. It does not bundle an MCP connection or run install hooks. Install Node.js 22+ and authorize a dedicated Sesha profile separately using the CLI setup above; never embed tokens in agent prompts or plugin files. With a connected Sesha MCP server, the same skill can use its available tools.

The repository marketplace is immediately installable. Listing in the Claude or OpenAI public directories requires separate publisher submission and review; a valid package does not establish directory acceptance. skills.sh discovery follows skill installs.

CLI, skills and packaging are MIT. The Sesha name and brand artwork are reserved; see assets/NOTICE. Support: https://sesha.sh/contact.
