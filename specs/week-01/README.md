# Week 01 Workflow Specs

> Week of 2026-06-21

## Workflows

| Workflow | Trigger | Status |
|----------|---------|--------|
| [Gmail Manager](gmail-manager.md) | Polling (1 min) | Draft (hierarchical multi-label) |
| [Label Resolver](label-resolver.md) | Sub-workflow | Draft (parent-first creation) |
| [Backfill Email Classifier](backfill-email-classifier.md) | Schedule (30 min) + Manual | Draft (batch classification) |
| [Record Active Labels](record-active-labels.md) | Manual | Production |
| [Delete Orphan Labels](delete-orphan-labels.md) | Manual | Production |
| [Daily Email Digest](daily-email-digest.md) | Schedule (8 AM) | Draft (Discord digest) |

See [runbooks](../../runbooks/week-01/) for operational details.
