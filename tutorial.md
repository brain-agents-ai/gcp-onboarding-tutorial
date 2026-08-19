# Grant Brain Agents AI read access to your GCP billing

This guided setup takes about **3 minutes**. You will grant a read-only service
account access to your billing export dataset plus a handful of read-only IAM
roles on your organization (or billing account plus projects, in fallback mode).

Everything runs in **your** Cloud Shell with **your** credentials. Brain Agents
AI never sees your grant commands; we only see the resulting IAM binding
after you click Verify Connection in the wizard.

## Prerequisites

Before starting, make sure you have:

- A GCP billing account with **billing export to BigQuery** enabled. If not
  set up, follow
  [Google's billing export guide](https://cloud.google.com/billing/docs/how-to/export-data-bigquery)
  first, then come back.
- Either **Organization Admin** on your GCP org (recommended, simpler setup)
  OR **Billing Account Admin** plus **Project IAM Admin** on the projects you
  want covered (fallback for organizations with stricter security policies).

The URL that opened this tutorial pre-filled three values for you as
environment variables. Confirm they look right:

```bash
echo "Billing account: ${brain_agents_ba:-not set}"
echo "Billing project: ${brain_agents_project:-not set}"
echo "Billing dataset: ${brain_agents_dataset:-not set}"
```

If any show `not set`, close this tutorial and reopen it from the Brain
Agents AI wizard so the values pre-fill correctly.

## Confirm the service account you are granting to

Brain Agents AI reads your data through a single dedicated service account.
The same account is used across every read path (BigQuery, Cloud Billing,
Recommender, Compute Engine commitments). You grant read access to it once;
we impersonate it internally when calling GCP APIs on your behalf.

```bash
export READER_SA="brainagents-ai-reader@ba-foundation-814201dc.iam.gserviceaccount.com"
echo "Reader SA: $READER_SA"
```

## Choose your grant mode

Pick **one** path below. The two are mutually exclusive.

### Path A: Organization-level grants (recommended)

Simplest setup: five grants at organization scope, works regardless of how
many projects or billing accounts you have. Requires Organization Admin.

Discover your organization ID:

```bash
gcloud organizations list --format="table(displayName, name)"
export ORGANIZATION_ID="$(gcloud organizations list --format='value(name)' | head -n1 | sed 's|organizations/||')"
echo "Detected organization ID: $ORGANIZATION_ID"
```

If the output above shows the wrong org (e.g., you belong to multiple GCP
organizations), override manually:

```bash
export ORGANIZATION_ID="123456789012"  # replace with the correct ID
```

Grant BigQuery Data Viewer on the billing export dataset:

```bash
bq add-iam-policy-binding \
  --member="serviceAccount:${READER_SA}" \
  --role="roles/bigquery.dataViewer" \
  "${brain_agents_project}:${brain_agents_dataset}"
```

Grant the four read-only roles at organization scope:

```bash
for role in \
  roles/billing.viewer \
  roles/billing.budgets.viewer \
  roles/recommender.viewer \
  roles/compute.viewer; do
  echo "Granting $role..."
  gcloud organizations add-iam-policy-binding "$ORGANIZATION_ID" \
    --member="serviceAccount:${READER_SA}" \
    --role="$role" \
    --condition=None
done
```

Skip to the "Verify and return" step at the bottom.

### Path B: Fallback mode (billing account plus per-project)

Use this if your security policy forbids organization-level grants, or if
you do not have Organization Admin. Grants live at billing-account scope for
what supports it (billing and budgets and billing-scoped recommender) and at
per-project scope for what does not (compute and project-scoped recommender).

Grant BigQuery Data Viewer on the dataset (same as Path A):

```bash
bq add-iam-policy-binding \
  --member="serviceAccount:${READER_SA}" \
  --role="roles/bigquery.dataViewer" \
  "${brain_agents_project}:${brain_agents_dataset}"
```

Grant three read-only roles at billing-account scope:

```bash
for role in \
  roles/billing.viewer \
  roles/billing.budgets.viewer \
  roles/recommender.viewer; do
  echo "Granting $role on billing account ${brain_agents_ba}..."
  gcloud billing accounts add-iam-policy-binding "${brain_agents_ba}" \
    --member="serviceAccount:${READER_SA}" \
    --role="$role"
done
```

Auto-discover every project associated with your billing account and grant
compute.viewer plus recommender.viewer on each:

```bash
mapfile -t PROJECTS < <(gcloud beta billing projects list \
  --billing-account="${brain_agents_ba}" \
  --format="value(projectId)")

echo "Detected ${#PROJECTS[@]} project(s) on billing account ${brain_agents_ba}:"
printf "  - %s\n" "${PROJECTS[@]}"
echo ""
echo "Press Enter to grant compute.viewer and recommender.viewer on each,"
echo "or Ctrl-C to abort and edit the PROJECTS list first."
read -r

for PROJECT in "${PROJECTS[@]}"; do
  for role in roles/compute.viewer roles/recommender.viewer; do
    echo "Granting $role on $PROJECT..."
    gcloud projects add-iam-policy-binding "$PROJECT" \
      --member="serviceAccount:${READER_SA}" \
      --role="$role" \
      --condition=None
  done
done
```

If any project fails ("permission denied"), that project stays uncovered but
the rest succeed. You can rerun the loop later against a curated list, or
run the equivalent `gcloud projects add-iam-policy-binding` command manually
per project with the correct IAM permissions.

## Verify and return to the wizard

Confirm the reader SA now shows up in the dataset IAM policy:

```bash
bq show --format=prettyjson "${brain_agents_project}:${brain_agents_dataset}" \
  | grep -A1 brainagents-ai-reader
```

You should see the reader SA listed with `roles/bigquery.dataViewer`.

Now return to the Brain Agents AI wizard tab in your browser and click
**Verify Connection** in Step 4. The wizard will probe all five grants and
show green checks for each.

If a grant fails to propagate immediately (IAM changes can take up to a
minute to become effective), the wizard retries automatically for a few
seconds. If any check stays red after 90 seconds, come back here to re-run
that specific grant, then click Verify again.

## Done

You can close this Cloud Shell tab. All commands ran with your credentials
only; nothing on our side needs to know they finished.
