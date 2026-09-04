# Runner label catalog

CI runner labels live in one place: [`.github/runners.yaml`](./runners.yaml).
This lets a runner-image migration (OS bump, hosted-image retirement) update a
single file instead of grepping every workflow.

## How it works

`runners.yaml` is the source of truth. A generated reusable workflow,
[`.github/workflows/reusable-runner-names.yml`](./workflows/reusable-runner-names.yml),
exports each catalog entry as a job output. Callers pull the label from that
output instead of hardcoding it, because `runs-on:` is evaluated before any step
runs, so a composite action cannot supply the label — a reusable workflow's
outputs can.

A catalog value is one of:

- a **scalar** label (`windows-2019`, `ubuntu-20.04`) — consumed as
  `runs-on: ${{ needs.runner_names.outputs.<key> }}`
- a **composite label set** (`[self-hosted, Linux, X64]`) — exported as a JSON
  array string and consumed as
  `runs-on: ${{ fromJSON(needs.runner_names.outputs.<key>) }}`

Rolling `-latest` aliases are intentionally left hardcoded — they are GitHub
aliases, not fleet labels.

## `matrix.os` stays a logical identity

`matrix.os` values (and `matrix.os == '...'` conditionals) are **not** managed by
the catalog — they are frozen logical identities used in names, cache keys and
`if:` guards. Where a matrix job's `runs-on` depends on the lane, drive it from
the outputs via a conditional keyed on `matrix.os`:

```yaml
jobs:
  runner_names:
    permissions:
      contents: read
    uses: ./.github/workflows/reusable-runner-names.yml

  ubuntu:
    needs: runner_names
    strategy:
      matrix:
        include:
          - os: ubuntu-22.04
          - os: ubuntu-20.04
    runs-on: ${{ matrix.os == 'ubuntu-20.04' && needs.runner_names.outputs.ubuntu_2004 || needs.runner_names.outputs.ubuntu_2204 }}
    steps: ...
```

## Changing a label

1. Edit `.github/runners.yaml`.
2. Regenerate: `node .github/scripts/sync-runner-names.mjs`
3. Test: `node --test .github/scripts/test/runner-names.test.mjs`

CI enforces both invariants via
[`runner-names-validate.yml`](./workflows/runner-names-validate.yml).
