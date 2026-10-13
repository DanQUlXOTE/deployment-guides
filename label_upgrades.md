# Bindplane: Label Collectors by Attribute and Upgrade by Label

Oct 13, 2026 · @Dan

## Overview

This procedure labels every collector that shares an attribute, then upgrades only the collectors carrying that label to a pinned version. The running example labels all Ubuntu hosts with `env=test` and upgrades them.

The flow is the same whichever tool you use:

1. Find the collectors that match the attribute.
2. Apply a label to them.
3. Verify the label.
4. Upgrade the collectors that carry the label to a specific version.
5. Verify the upgrade.

Steps 1 to 5 use the `bindplane` CLI and the REST API from a shell. The Ansible section runs the same flow as one playbook. Commands were checked against Bindplane CLI v1.94.0 and Bindplane Cloud.

If the collectors that you want to upgrade already have a label you want to use, skip to step 4.

## Prerequisites

You need a working CLI profile, an API key exported in the shell, and `jq`.

- **CLI:** `bindplane` installed and pointed at your server. `bindplane get agents` should list your collectors.
- **API key:** an API key for the project. The REST calls send it in the `X-Bindplane-Api-Key` header.
- **Server URL:** `https://app.bindplane.com` for Bindplane Cloud, or your own server URL for self-hosted. The API base path is `/v1`.
- **Tools:** `curl` and `jq` for the shell steps; `ansible-core` for the playbook.

Set these once per shell session. The remaining steps assume them:

```bash
export BINDPLANE_URL=https://app.bindplane.com
export BINDPLANE_API_KEY=<your-api-key>
```

If the CLI profile already holds the key, you can read it from there:

```bash
export BINDPLANE_API_KEY=$(bindplane profile get "$(bindplane profile current)" | awk '/apiKey:/{print $2}')
```

The API reference for your server version is served at `$BINDPLANE_URL/swagger/doc.json`. The agent endpoints are under `/v1/collectors`.

## Step 1: Find the collectors that match the attribute

Filter the collector list on the attribute field with `jq` and keep the IDs. Attributes such as the operating system are fields on the collector, not labels, so a label selector cannot find them.

First look at the values the field takes across your fleet:

```bash
bindplane get agents --limit 1000 -o json | jq -r '.[].operatingSystem' | sort | uniq -c
```

Then build the list of matching IDs. This example matches every operating system string that starts with `Ubuntu`:

```bash
ids=$(bindplane get agents --limit 1000 -o json \
  | jq -c '[.[] | select((.operatingSystem // "") | test("^Ubuntu")) | .id]')

echo "$ids" | jq length
```

Check that the count is what you expect before going on. Other fields you can match the same way:

| Field | Example value |
| --- | --- |
| `operatingSystem` | `Ubuntu 24.04` |
| `platform` | `linux` |
| `arch` | `amd64` |
| `hostname` | `web-01` |
| `version` | `v1.94.2` |
| `type` | `com.observiq.collector` |

`get agents` returns 100 collectors by default. Raise `--limit`, or page with `--offset`, if your fleet is larger.

## Step 2: Apply the label

Send the IDs and the label to the bulk label endpoint, `PATCH /v1/collectors/labels`, in one call:

```bash
curl -sS -w '\nHTTP %{http_code}\n' -X PATCH "$BINDPLANE_URL/v1/collectors/labels" \
  -H "X-Bindplane-Api-Key: $BINDPLANE_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"ids\": $ids, \"labels\": {\"env\": \"test\"}, \"includeFailedIds\": true}"
```

A successful call returns `HTTP 200` and a body like `{"errors":[],"count":9}`, where `count` is the number of collectors updated. Keep the `-w` option: a missing or wrong API key returns `HTTP 401` with an empty body, which otherwise looks like nothing happened.

Request body fields:

| Field | Purpose |
| --- | --- |
| `ids` | Collector IDs to label. |
| `labels` | Label names and values. An empty value deletes that label. |
| `overwrite` | Set to `true` to change a label that already exists with a different value. Defaults to `false`. |
| `includeFailedIds` | Returns the IDs that could not be updated in `failedIds`. |
| `selector` | Alternative to `ids`: target collectors by an existing label, for example `configuration=my-config`. |
| `query` | Alternative to `ids`: target collectors by a search string. |

**CLI alternative.** The CLI labels one collector per call, so loop over the IDs:

```bash
echo "$ids" | jq -r '.[]' | xargs -I{} bindplane label agent {} env=test
```

Add `--overwrite` to replace an existing `env` value.

## Step 3: Verify the label

List the collectors by label selector and confirm the set matches Step 1:

```bash
bindplane get agents -l env=test --limit 1000
```

The `LABELS` column should show `env=test` on every row, and the row count should equal the count from Step 1. To compare the two sets exactly:

```bash
diff <(echo "$ids" | jq -r '.[]' | sort) \
     <(bindplane get agents -l env=test --limit 1000 -o json | jq -r '.[].id' | sort) \
  && echo "label set matches"
```

Extra IDs on the right side are collectors that already carried the label before this run. Decide whether they belong in the upgrade before continuing, because Step 4 upgrades everything with the label.

## Step 4: Upgrade the labeled collectors to a specific version

Select the collectors by label, then pass their IDs to the upgrade command with the version pinned. `bindplane update agent` has no label flag of its own, so the selection happens in `get agents`.

Pick the version. It must be one the server knows about, written with the leading `v`:

```bash
bindplane get agent-versions
export TARGET_VERSION=v1.94.2
```

**CLI.** Upgrade each labeled collector:

```bash
bindplane get agents -l env=test --limit 1000 -o json \
  | jq -r '.[].id' \
  | xargs -n1 bindplane update agent --version "$TARGET_VERSION"
```

**API.** The bulk endpoint `PATCH /v1/collectors/version` takes all the IDs in one request:

```bash
upgrade_ids=$(bindplane get agents -l env=test --limit 1000 -o json | jq -c '[.[].id]')

curl -sS -w '\nHTTP %{http_code}\n' -X PATCH "$BINDPLANE_URL/v1/collectors/version" \
  -H "X-Bindplane-Api-Key: $BINDPLANE_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"ids\": $upgrade_ids, \"version\": \"$TARGET_VERSION\"}"
```

The API answers `HTTP 204` with no body when the request is accepted. That means the upgrade was queued, not that it finished; Step 5 confirms the result.

Only connected collectors can act on an upgrade. To skip disconnected ones, add `select(.status == 1)` to the `jq` filter, for example `jq -c '[.[] | select(.status == 1) | .id]'`.

## Step 5: Verify the upgrade

The upgrade is done when every labeled collector reports the target version and a `Connected` status:

```bash
bindplane get agents -l env=test --limit 1000
```

Watch the `VERSION` and `STATUS` columns. Collectors show `Upgrading` while the new package installs, then return to `Connected` on the new version. To list only the collectors that are not there yet:

```bash
bindplane get agents -l env=test --limit 1000 -o json \
  | jq -r --arg v "$TARGET_VERSION" \
      '.[] | select(.version != $v) | [.id, .name, .version, .status] | @tsv'
```

An empty result means every labeled collector is on the target version. The numeric `status` values in the JSON output are:

| Value | Status |
| --- | --- |
| 0 | Disconnected |
| 1 | Connected |
| 2 | Error |
| 4 | Component failed |
| 6 | Configuring |
| 7 | Upgrading |
| 8 | Pending |
| 9 | Incompatible |

If an upgrade fails, the collector's `upgrade.error` field in the JSON output holds the reason.

## Ansible playbook

The playbook below runs the whole flow against the Bindplane REST API from the control node. It uses only `ansible.builtin` modules, so it needs `ansible-core` and nothing else. It talks to the Bindplane server, not to the collector hosts, so no inventory is required.

Save it as `bindplane_label_upgrade.yml`:

```yaml
---
# Label Bindplane collectors by attribute, then upgrade the labeled collectors.
# Usage:
#   export BINDPLANE_API_KEY=...
#   ansible-playbook bindplane_label_upgrade.yml -e target_version=v1.94.2
- name: Label Bindplane collectors by attribute and upgrade by label
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    bindplane_url: "https://app.bindplane.com"
    bindplane_api_key: "{{ lookup('ansible.builtin.env', 'BINDPLANE_API_KEY') }}"

    # Attribute to match. Any top-level string field on a collector works,
    # for example operatingSystem, platform, arch, hostname, version.
    match_field: operatingSystem
    match_regex: "^Ubuntu"

    # Label to apply to the matching collectors.
    label_key: env
    label_value: test
    label_overwrite: false

    # Version to upgrade the labeled collectors to.
    target_version: v1.94.2

    page_limit: 1000
    upgrade_retries: 30
    upgrade_delay: 20

    api_headers:
      X-Bindplane-Api-Key: "{{ bindplane_api_key }}"

  tasks:
    - name: Require an API key
      ansible.builtin.assert:
        that: bindplane_api_key | length > 0
        fail_msg: "Set BINDPLANE_API_KEY (an empty key returns a 401 with no body)."
      tags: [always]

    - name: List all collectors
      ansible.builtin.uri:
        url: "{{ bindplane_url }}/v1/collectors?limit={{ page_limit }}"
        headers: "{{ api_headers }}"
        return_content: true
      register: all_collectors
      tags: [label]

    - name: Select collectors whose attribute matches
      ansible.builtin.set_fact:
        match_ids: >-
          {{ all_collectors.json.agents
             | selectattr(match_field, 'string')
             | selectattr(match_field, 'match', match_regex)
             | map(attribute='id') | list }}
        unlabeled_ids: >-
          {{ all_collectors.json.agents
             | selectattr(match_field, 'string')
             | selectattr(match_field, 'match', match_regex)
             | rejectattr('labels.' ~ label_key, 'equalto', label_value)
             | map(attribute='id') | list }}
      tags: [label]

    - name: Stop if nothing matched
      ansible.builtin.assert:
        that: match_ids | length > 0
        fail_msg: "No collectors have {{ match_field }} matching {{ match_regex }}."
        success_msg: "{{ match_ids | length }} collectors matched, {{ unlabeled_ids | length }} still need the label."
      tags: [label]

    - name: Apply the label
      ansible.builtin.uri:
        url: "{{ bindplane_url }}/v1/collectors/labels"
        method: PATCH
        headers: "{{ api_headers }}"
        body_format: json
        body:
          ids: "{{ unlabeled_ids }}"
          labels: "{{ {label_key: label_value} }}"
          overwrite: "{{ label_overwrite | bool }}"
          includeFailedIds: true
        status_code: 200
      register: label_result
      when: unlabeled_ids | length > 0
      changed_when: label_result.json.count | default(0) > 0
      failed_when: >-
        label_result.status != 200
        or (label_result.json.errors | default([]) | length > 0)
        or (label_result.json.failedIds | default([]) | length > 0)
      tags: [label]

    - name: List collectors that carry the label
      ansible.builtin.uri:
        url: "{{ bindplane_url }}/v1/collectors?limit={{ page_limit }}&selector={{ (label_key ~ '=' ~ label_value) | urlencode }}"
        headers: "{{ api_headers }}"
        return_content: true
      register: labeled
      tags: [label, plan, upgrade]

    - name: Verify every matched collector has the label
      ansible.builtin.assert:
        that: match_ids | difference(labeled.json.agents | map(attribute='id') | list) | length == 0
        fail_msg: "Some matched collectors are missing {{ label_key }}={{ label_value }}."
      tags: [label]

    - name: Pick labeled collectors that are connected and not on the target version
      ansible.builtin.set_fact:
        upgrade_ids: >-
          {{ labeled.json.agents
             | selectattr('status', 'equalto', 1)
             | rejectattr('version', 'equalto', target_version)
             | map(attribute='id') | list }}
      tags: [plan, upgrade]

    - name: Show the upgrade plan
      ansible.builtin.debug:
        msg: >-
          {{ labeled.json.agents | length }} labeled,
          {{ upgrade_ids | length }} to upgrade to {{ target_version }}
          (disconnected collectors and collectors already on the target are skipped).
      tags: [plan, upgrade]

    - name: Start the upgrade
      ansible.builtin.uri:
        url: "{{ bindplane_url }}/v1/collectors/version"
        method: PATCH
        headers: "{{ api_headers }}"
        body_format: json
        body:
          ids: "{{ upgrade_ids }}"
          version: "{{ target_version }}"
        status_code: 204
      when: upgrade_ids | length > 0
      changed_when: true
      tags: [upgrade]

    - name: Wait until the upgraded collectors report the target version
      ansible.builtin.uri:
        url: "{{ bindplane_url }}/v1/collectors?limit={{ page_limit }}&selector={{ (label_key ~ '=' ~ label_value) | urlencode }}"
        headers: "{{ api_headers }}"
        return_content: true
      register: after
      when: upgrade_ids | length > 0
      until: >-
        after.json.agents
        | selectattr('id', 'in', upgrade_ids)
        | rejectattr('version', 'equalto', target_version)
        | list | length == 0
      retries: "{{ upgrade_retries }}"
      delay: "{{ upgrade_delay }}"
      tags: [upgrade]
```

### Running it

Export the API key, then run the stages with tags. Running `label` and `plan` first lets you review the target set before anything is upgraded.

| Command | What it does |
| --- | --- |
| `ansible-playbook bindplane_label_upgrade.yml --tags label` | Finds the matching collectors, labels them, verifies the label. |
| `ansible-playbook bindplane_label_upgrade.yml --tags plan` | Prints how many labeled collectors would be upgraded. Changes nothing. |
| `ansible-playbook bindplane_label_upgrade.yml --tags upgrade` | Upgrades the labeled collectors and waits for them to report the target version. |
| `ansible-playbook bindplane_label_upgrade.yml` | Runs everything end to end. |

Override any variable on the command line, for example:

```bash
ansible-playbook bindplane_label_upgrade.yml \
  -e match_field=platform -e match_regex='^windows$' \
  -e label_key=ring -e label_value=canary \
  -e target_version=v1.94.2
```

### Behavior to know

- **Idempotent labeling:** collectors that already carry the label are left alone, so a second run reports no change.
- **Existing values:** with `label_overwrite: false`, a collector that already has the label key with a different value is expected to make the label task fail (the API requires overwrite to change an existing value). Set it to `true` to replace the value.
- **Upgrade scope:** only collectors that are connected and not already on `target_version` are sent an upgrade.
- **Wait time:** the last task polls every 20 seconds for up to 30 tries (10 minutes). Raise `upgrade_retries` for large fleets.
- **Secrets:** the key is read from the `BINDPLANE_API_KEY` environment variable. In a pipeline, supply it from Ansible Vault or your secret store, and add `no_log: true` to the `uri` tasks if task output is archived.

### Test status

The `label` and `plan` stages were run against Bindplane Cloud with ansible-core 2.21 and passed, including the empty API key check. The two `upgrade` tasks (the `PATCH /v1/collectors/version` call and the wait loop) follow the server's API spec but have not been run, because no connected collector was available to upgrade in the test project. Run the `upgrade` stage against a small label group first.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `curl` prints nothing and no labels change | `BINDPLANE_API_KEY` is empty or wrong. The server returns `HTTP 401` with an empty body. | Export the key and keep `-w '\nHTTP %{http_code}\n'` on every `curl`. |
| Label call returns `count` lower than expected, or entries in `errors` or `failedIds` | Some collectors already have the label key with a different value. | Resend with `"overwrite": true`, or `--overwrite` on the CLI. |
| Fewer collectors found than you expect | `get agents` and `GET /v1/collectors` return 100 by default. | Raise `--limit` (or `limit=`), or page with `--offset`. |
| A label selector finds nothing for an attribute such as the OS | Attributes like `operatingSystem` are fields, not labels. | Filter the JSON output on the field, as in Step 1. |
| `-q ubuntu` or `"query": "ubuntu"` matches extra collectors | A free-text query matches any field, including hostnames. | Filter on the specific field and pass explicit `ids`. |
| Upgrade accepted (`HTTP 204`) but the version does not change | The collector is disconnected, or the version is not available for its type. | Check `STATUS`, check `bindplane get agent-versions`, and read `upgrade.error` on the collector. |
| Single-collector upgrade returns `HTTP 409` | That collector does not support remote upgrade. | Upgrade it with its installer or package manager. |
