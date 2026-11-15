# CLI workflows

Replace `my-agent` with the user's chosen profile. `npx @seshat.sh/cli@0.8.1` can replace `seshat`. Commands here need the CLI release that includes this skill.

## Connect and inspect

```sh
seshat agent setup --profile my-agent --name "My agent" --scopes artifacts:read,artifacts:write
seshat auth status --profile my-agent --json
seshat whoami --profile my-agent --json
```

Approve the browser request only after the user checks its code, space and scopes. Connection setup defaults to upload/edit access; use the explicit read/write scopes above for inspection and updates. On a headless POSIX host, `--no-browser --storage file` explicitly selects owner-only credential files. Credentials never belong in command arguments.

## New file and later iteration

```sh
seshat upload ./report.html --title "Quarterly report" --request-id quarterly-report-first --profile my-agent --json
seshat artifacts get ARTIFACT_ID --profile my-agent --json
seshat artifacts replace ARTIFACT_ID --file ./report.html --revision CURRENT_REVISION --request-id quarterly-report-update-2 --profile my-agent --json
seshat artifacts versions ARTIFACT_ID --profile my-agent --json
```

Read CURRENT_REVISION from `get`. Inherit sharing defaults on creation; replacement preserves current protection/expiry. Use a new request ID for each intentional change, unchanged for retries. Unchanged-update request receipts are kept for 30 days. Inspect an old ambiguous operation before retrying it after that window.

The response contains `id`, `revision`, `url`, `size`, `expiresAt`, and sometimes `password` or `notice`. Verify the returned metadata, then give the user the share URL and any password/expiry. With read access, inspect it again using `artifacts get`.

## Bundles and folders

```sh
seshat upload ./release --folder "Release report" --request-id release-report-first --profile my-agent --json
seshat upload ./release --into FOLDER_ID --request-id release-report-refresh --profile my-agent --json
seshat upload ./images --into FOLDER_ID --prefix images --request-id release-images-refresh --profile my-agent --json
```

`--folder` creates one folder; retain its returned ID. `--into` accepts that ID or its share link. Paths are preserved inside a directory; unchanged content keeps its stored copy. Remote paths absent locally are retained. Review `published` and `failed` even on a response containing an artifact URL. A failed batch leaves the previous folder unchanged; retry the full batch at the known folder ID. Successful batches are atomic revisions.

CLI folder uploads reuse matching small files without transfer, including with upload-only access. Large files stream once from the checked descriptor; the returned digest is verified against that same stream. Run `seshat artifacts files FOLDER_ID --profile my-agent --json` or MCP `seshat_list_folder_files` to verify the intended paths, sizes and SHA-256 digests. Compare digests with local files (for example, `shasum -a 256 ./report.md`).

## Sharing and destructive operations

Only use `--no-password`, `--protect`, `--no-expiry` or `--expires-at` when the user requests that choice. Existing folder upload flags do not change its sharing settings.

The CLI provides `artifacts delete --dry-run` to inspect a deletion and `--yes` to execute an authorized deletion. Unpublishing a retained file version blocks its public pinned link; deleting a version removes its bytes. Both require an explicit user purpose. Files and folders both support retained revisions. A folder revision includes its complete manifest; unchanged assets share stored content. Free retains three revisions including current; Personal is bounded by storage. Restore with `seshat artifacts restore ARTIFACT_ID --target-revision OLD --revision CURRENT --request-id STABLE_ID --profile my-agent --json`. Restore preserves current protection/expiry. A folder restore that removes current paths also requires delete permission.
