# Helm Chart Vendor + Patch Workflow

## How It Works

The setup ensures reproducible, auditable chart modifications:

1. **`.charts.yml`** — Source of truth for upstream chart versions. Lists each chart's source repo and version.
2. **`charts/patches/`** — All customizations stored as discrete patch files. Each patch represents one logical change to an upstream chart.
3. **`charts/`** — The final vendored result: upstream chart + all patches applied in order.

## Why This Matters

- When upstream releases a new version, update `.charts.yml` and re-run `chart-vendor`. It fetches the new upstream and re-applies patches.
- If patches don't apply cleanly to the new upstream, you know immediately what broke.
- All customizations are visible as patches rather than hidden in a large vendored directory.
- CI verifies that `charts/` = upstream + patches (no drift).

## Modifying a Chart

1. Make your changes directly in the `charts/<chart-name>/` directory.
2. Create a corresponding patch file in `charts/patches/<chart-name>/`.
3. The patch should capture only your logical change against the upstream base.

## Upgrading an Upstream Chart

1. Update the version in `.charts.yml`.
2. Run `chart-vendor` to fetch the new upstream.
3. If patches fail to apply, resolve conflicts in the patch files.
4. Verify the final `charts/` directory matches upstream + patches.
