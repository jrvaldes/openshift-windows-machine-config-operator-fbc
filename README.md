# Windows Machine Config Operator FBC

FBC for the [Windows Machine Config Operator](https://github.com/openshift/windows-machine-config-operator)

The [olm-sample](https://github.com/konflux-ci/olm-operator-konflux-sample/blob/main/docs/konflux-onboarding.md) should be used as a source of truth, over the information here.

## Adding a release stream

### Adding a new release stream

```
# Copy an existing directory
cp -r windows-machine-config-operator/release-4.15/ windows-machine-config-operator/new-stream

# Replace references to old OCP version with new
...

# Generate catalog using a recent bundle build
opm render quay.io/redhat-user-workloads/windows-machine-conf-tenant/windows-machine-config-operator/windows-machine-config-operator-bundle-<$release>@<$DIGEST> --migrate-level=bundle-object-to-csv-metadata >catalog.json
# Convert the catalog to a basic template
opm alpha convert-template basic catalog.json >catalog-template.json

# Ensure all references in catalog.json point to the same registry.
# This should be registry.redhat.io for most streams and registry.stage.redhat.io for the stream tracked by the WMCO master branch
# Both the catalog and the template should be committed to git
```

### Adding a release stream from an older catalog source (migrating to konflux)

```
# Adding 4.17 as an example
# Copy an existing directory
cp -r windows-machine-config-operator/release-4.15/ windows-machine-config-operator/release-4.17

# Replace references to old OCP version with new
...

# Pull Red Hat operators catalog for the given OCP version
opm migrate registry.redhat.io/redhat/redhat-operator-index:v4.17 ./catalog-migrate

# Copy the WMCO catalog
cp catalog-migrate/windows-machine-config-operator/catalog.json windows-machine-config-operator/release-4.17/catalog/catalog.json
# Convert the catalog to a basic template
opm alpha convert-template basic catalog.json >catalog-template.json

# Both the catalog and the template should be committed to source
```

## Updating a patch version before release

```
# Update the bundle digest in the catalog-template for the stream with your editor of choice.
nvim v10.19/catalog-template.json

# Generate a new catalog
opm alpha render-template basic v10.19/catalog-template.json --migrate-level=bundle-object-to-csv-metadata  > v10.19/catalog/windows-machine-config-operator/catalog.json; 

# Ensure all references in catalog.json point to the same registry.
# This should be registry.redhat.io for most streams and registry.stage.redhat.io for the stream tracked by the WMCO master branch
```

## Promoting a patch release

> **Prerequisite:** [`hack/promote-release.sh` at commit `916953f`](https://github.com/jrvaldes/openshift-windows-machine-config-operator-fbc/blob/916953f00c43d5fdd5ad7ddc7f2849affc2e3f72/hack/promote-release.sh) must be merged into `main` before these instructions can be used. The script is not yet present on `main`.

Use this script to cut the next patch version in the catalog's `stable` channel from the staged candidate. To update a candidate's bundle digest without adding a stable version, use the preceding procedure or `hack/release-snapshot.sh <TARGET_BRANCH> [COMMIT_SHA]` instead.

Before promoting:

- Have Bash 4 or later (the map uses associative arrays), `git`, `jq`, `sed`, and `mv` available, with Git configured to create commits.
- Work on your intended change branch with a clean working tree and index. The script checks the selected template for pending changes, but its automatic commit can include other already-staged files.
- Ensure `TARGET_BRANCH` has an entry in `hack/release_catalog_map.sh` and the mapped directory contains `catalog-template.json`. The template needs an `olm.package`, a nonempty `stable` channel with a last version and a `skipRange` in the form `>=X.Y.Z <A.B.C`, and a final `olm.bundle` reference on `registry.stage.redhat.io`.

Run from the repository root, since the script uses relative paths. `TARGET_BRANCH` selects a catalog; it does not check out that Git branch. For example, `release-4.19` maps to `v10.19`:

```bash
TARGET_BRANCH=release-4.19
./hack/promote-release.sh "$TARGET_BRANCH"
```

The script increments the patch number of the last `stable` entry, preserving its major/minor version and checking that it matches the catalog directory. It appends a stable entry whose `replaces` names the previous entry and whose `skipRange` retains the previous lower bound and excludes the new version at the upper bound (`>=LOWER_BOUND <NEW_VERSION`).

It rewrites **all existing staged `olm.bundle` image references** in the template from `registry.stage.redhat.io` to `registry.redhat.io`, then appends a copy of the original staged candidate as the last bundle slot for future snapshot updates. These are catalog reference substitutions only: the script does **not** copy or publish registry images, check their production availability, or render `catalog.json`.

The template is automatically staged and committed on the current branch as `<TARGET_BRANCH>: add v<NEW_VERSION>`. The script then asks whether to run `hack/release-snapshot.sh` for the same target; the default is no. Answering yes uses the latest matching snapshot, updates the retained candidate slot, renders the catalog, and creates a separate snapshot branch and commit. This requires `oc` access to the `windows-machine-conf-tenant` project in the rhtap-releng tenant, `jq`, `opm`, and local configuration for `registry.stage.redhat.io`, as described in the snapshot script.

If you decline the snapshot step, render the updated template before submitting the change, using the preceding procedure for the selected stream. For the example above:

```bash
opm alpha render-template basic v10.19/catalog-template.json --migrate-level=bundle-object-to-csv-metadata > v10.19/catalog/windows-machine-config-operator/catalog.json
```

For either path, review the template and rendered catalog. Run `bash lint.sh` to check that each rendered catalog's bundle related-image references use a single registry; keep the staged candidate slot in the template. Commit any rendered catalog changes, then push the resulting change branch and open a pull request against `main`. Neither script pushes automatically; the snapshot script only prints a suggested push command.
