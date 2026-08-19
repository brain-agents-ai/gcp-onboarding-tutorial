# Brain Agents AI GCP Onboarding Tutorial

This repository hosts the interactive Cloud Shell tutorial for granting
Brain Agents AI read access to your Google Cloud billing data during
onboarding.

## What this is

When you sign up at [brainagents.ai](https://brainagents.ai) and connect
your GCP billing, the wizard's Step 3 offers three ways to grant the read
access we need. One of them is "Open in Cloud Shell", which opens a
guided walkthrough that runs entirely in your own Cloud Shell with your
own credentials.

That walkthrough lives in [`tutorial.md`](tutorial.md).

## Why a public repo

Google Cloud Shell's tutorial mode loads walkthrough content from a
public git repository. This repo exists solely to host that walkthrough.

## What the tutorial does

Grants five read-only IAM roles to our reader service account
(`brainagents-ai-reader@ba-foundation-814201dc.iam.gserviceaccount.com`):

- `roles/bigquery.dataViewer` on your billing export dataset (read
  billing rows).
- `roles/billing.viewer` on your organization or billing account (read
  billing account metadata).
- `roles/billing.budgets.viewer` on your organization or billing account
  (read budgets for alerting).
- `roles/recommender.viewer` on your organization or on the billing
  account plus each project (read Google Cloud Recommender API output).
- `roles/compute.viewer` on your organization or on each project (list
  Compute Engine commitments).

All roles are read-only. No write access is requested at any point.

## Feedback and support

For product feedback, questions about your onboarding, or bug reports,
please contact us through [brainagents.ai/contact](https://brainagents.ai/contact).

External pull requests are not accepted at this time.

## License

MIT. See [`LICENSE`](LICENSE).
