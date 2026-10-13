# pnpm Update Management

A reminder on how to update dependencies **per project folder** (e.g. `template/`, `official/`), not from the repo root. Each folder has its own `package.json` and `pnpm-workspace.yaml`, so updates must run inside it.

`<dir>` = the project folder you want to update.

## Steps

### 1. Enter the project folder

```bash
cd "<dir>"
```

Quote the path — this repo lives under a directory containing a space (`Coding Project`).

### 2. Set `minimumReleaseAge: 0` in `pnpm-workspace.yaml`

This setting blocks packages published more recently than the given number of minutes. Set it to `0` so the newest releases are eligible:

```yaml
allowBuilds:
  esbuild: true
minimumReleaseAge: 0
```

Add the key if missing; change it to `0` if set to a non-zero value.

## Notes

- Repeat for each project folder — updating `template/` does not touch `official/`.
- `pnpm outdated` (inside `<dir>`) previews what would change.
