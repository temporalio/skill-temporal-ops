# Command Index

Alphabetical index of every `temporal` and `tcld` leaf subcommand. Use this
file as a "find-by-name" escape hatch when the main ops references do not
match your phrasing.

**How this file is organized.** Every row's command is transcribed from a
heading in the corresponding file under `docs/cli/` (for `temporal`) or
`docs/cloud/tcld/` (for `tcld`). The *Docs source* column gives the exact
file. The *Reference file* column points to the ops reference file that
explains the command.

Leaf-only: command groups (`temporal workflow`, `tcld namespace`) are not
listed -- only their terminal subcommands are.

## Table of contents

- [temporal](#temporal)
- [tcld](#tcld)
- [Notes on scope](#notes-on-scope)

## temporal

| Command | Binary | Docs source | Reference file |
|---------|--------|-------------|----------------|
| `temporal activity cancel` | `temporal` | `docs/cli/activity.mdx#cancel` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity complete` | `temporal` | `docs/cli/activity.mdx#complete` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity count` | `temporal` | `docs/cli/activity.mdx#count` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity describe` | `temporal` | `docs/cli/activity.mdx#describe` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity execute` | `temporal` | `docs/cli/activity.mdx#execute` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity fail` | `temporal` | `docs/cli/activity.mdx#fail` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity list` | `temporal` | `docs/cli/activity.mdx#list` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity pause` | `temporal` | `docs/cli/activity.mdx#pause` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity reset` | `temporal` | `docs/cli/activity.mdx#reset` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity result` | `temporal` | `docs/cli/activity.mdx#result` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity start` | `temporal` | `docs/cli/activity.mdx#start` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity terminate` | `temporal` | `docs/cli/activity.mdx#terminate` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity unpause` | `temporal` | `docs/cli/activity.mdx#unpause` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal activity update-options` | `temporal` | `docs/cli/activity.mdx#update-options` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal batch describe` | `temporal` | `docs/cli/batch.mdx#describe` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal batch list` | `temporal` | `docs/cli/batch.mdx#list` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal batch terminate` | `temporal` | `docs/cli/batch.mdx#terminate` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal env delete` | `temporal` | `docs/cli/env.mdx#delete` | [cli-scripting.md](cli-scripting.md) |
| `temporal env get` | `temporal` | `docs/cli/env.mdx#get` | [cli-scripting.md](cli-scripting.md) |
| `temporal env list` | `temporal` | `docs/cli/env.mdx#list` | [cli-scripting.md](cli-scripting.md) |
| `temporal env set` | `temporal` | `docs/cli/env.mdx#set` | [cli-scripting.md](cli-scripting.md) |
| `temporal config delete` | `temporal` | `docs/cli/config.mdx#delete` | [cli-scripting.md](cli-scripting.md) |
| `temporal config delete-profile` | `temporal` | `docs/cli/config.mdx#delete-profile` | [cli-scripting.md](cli-scripting.md) |
| `temporal config get` | `temporal` | `docs/cli/config.mdx#get` | [cli-scripting.md](cli-scripting.md) |
| `temporal config list` | `temporal` | `docs/cli/config.mdx#list` | [cli-scripting.md](cli-scripting.md) |
| `temporal config set` | `temporal` | `docs/cli/config.mdx#set` | [cli-scripting.md](cli-scripting.md) |
| `temporal operator cluster describe` | `temporal` | `docs/cli/operator.mdx#describe` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator cluster health` | `temporal` | `docs/cli/operator.mdx#health` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator cluster list` | `temporal` | `docs/cli/operator.mdx#list` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator cluster remove` | `temporal` | `docs/cli/operator.mdx#remove` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator cluster system` | `temporal` | `docs/cli/operator.mdx#system` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator cluster upsert` | `temporal` | `docs/cli/operator.mdx#upsert` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator namespace create` | `temporal` | `docs/cli/operator.mdx#create` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator namespace delete` | `temporal` | `docs/cli/operator.mdx#delete` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator namespace describe` | `temporal` | `docs/cli/operator.mdx#describe` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator namespace list` | `temporal` | `docs/cli/operator.mdx#list` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator namespace update` | `temporal` | `docs/cli/operator.mdx#update` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator nexus endpoint create` | `temporal` | `docs/cli/operator.mdx#create` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator nexus endpoint delete` | `temporal` | `docs/cli/operator.mdx#delete` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator nexus endpoint get` | `temporal` | `docs/cli/operator.mdx#get` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator nexus endpoint list` | `temporal` | `docs/cli/operator.mdx#list` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator nexus endpoint update` | `temporal` | `docs/cli/operator.mdx#update` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator search-attribute create` | `temporal` | `docs/cli/operator.mdx#create` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator search-attribute list` | `temporal` | `docs/cli/operator.mdx#list` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal operator search-attribute remove` | `temporal` | `docs/cli/operator.mdx#remove` | [self-hosted-admin.md](self-hosted-admin.md) |
| `temporal schedule backfill` | `temporal` | `docs/cli/schedule.mdx#backfill` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal schedule create` | `temporal` | `docs/cli/schedule.mdx#create` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal schedule delete` | `temporal` | `docs/cli/schedule.mdx#delete` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal schedule describe` | `temporal` | `docs/cli/schedule.mdx#describe` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal schedule list` | `temporal` | `docs/cli/schedule.mdx#list` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal schedule toggle` | `temporal` | `docs/cli/schedule.mdx#toggle` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal schedule trigger` | `temporal` | `docs/cli/schedule.mdx#trigger` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal schedule update` | `temporal` | `docs/cli/schedule.mdx#update` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal server start-dev` | `temporal` | `docs/cli/server.mdx#start-dev` | [cli-scripting.md](cli-scripting.md) |
| `temporal task-queue config get` | `temporal` | `docs/cli/task-queue.mdx#get` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue config set` | `temporal` | `docs/cli/task-queue.mdx#set` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue describe` | `temporal` | `docs/cli/task-queue.mdx#describe` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue get-build-id-reachability` | `temporal` | `docs/cli/task-queue.mdx#get-build-id-reachability` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue get-build-ids` | `temporal` | `docs/cli/task-queue.mdx#get-build-ids` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue list-partition` | `temporal` | `docs/cli/task-queue.mdx#list-partition` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue update-build-ids add-new-compatible` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#add-new-compatible` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue update-build-ids add-new-default` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#add-new-default` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue update-build-ids promote-id-in-set` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#promote-id-in-set` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue update-build-ids promote-set` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#promote-set` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue versioning add-redirect-rule` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#add-redirect-rule` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue versioning commit-build-id` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#commit-build-id` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue versioning delete-assignment-rule` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#delete-assignment-rule` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue versioning delete-redirect-rule` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#delete-redirect-rule` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue versioning get-rules` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#get-rules` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue versioning insert-assignment-rule` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#insert-assignment-rule` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue versioning replace-assignment-rule` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#replace-assignment-rule` | [workflow-health.md](workflow-health.md) |
| `temporal task-queue versioning replace-redirect-rule` *(deprecated)* | `temporal` | `docs/cli/task-queue.mdx#replace-redirect-rule` | [workflow-health.md](workflow-health.md) |
| `temporal workflow cancel` | `temporal` | `docs/cli/workflow.mdx#cancel` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal workflow count` | `temporal` | `docs/cli/workflow.mdx#count` | [workflow-health.md](workflow-health.md) |
| `temporal workflow delete` | `temporal` | `docs/cli/workflow.mdx#delete` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal workflow describe` | `temporal` | `docs/cli/workflow.mdx#describe` | [workflow-health.md](workflow-health.md) |
| `temporal workflow execute` | `temporal` | `docs/cli/workflow.mdx#execute` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow execute-update-with-start` *(experimental)* | `temporal` | `docs/cli/workflow.mdx#execute-update-with-start` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow fix-history-json` | `temporal` | `docs/cli/workflow.mdx#fix-history-json` | [cli-scripting.md](cli-scripting.md) |
| `temporal workflow list` | `temporal` | `docs/cli/workflow.mdx#list` | [workflow-health.md](workflow-health.md) |
| `temporal workflow metadata` | `temporal` | `docs/cli/workflow.mdx#metadata` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow pause` *(experimental)* | `temporal` | `docs/cli/workflow.mdx#pause` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal workflow query` | `temporal` | `docs/cli/workflow.mdx#query` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow reset` | `temporal` | `docs/cli/workflow.mdx#reset` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal workflow result` | `temporal` | `docs/cli/workflow.mdx#result` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow show` | `temporal` | `docs/cli/workflow.mdx#show` | [workflow-health.md](workflow-health.md) |
| `temporal workflow signal` | `temporal` | `docs/cli/workflow.mdx#signal` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow signal-with-start` | `temporal` | `docs/cli/workflow.mdx#signal-with-start` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow stack` | `temporal` | `docs/cli/workflow.mdx#stack` | [workflow-health.md](workflow-health.md) |
| `temporal workflow start` | `temporal` | `docs/cli/workflow.mdx#start` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow start-update-with-start` *(experimental)* | `temporal` | `docs/cli/workflow.mdx#start-update-with-start` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow terminate` | `temporal` | `docs/cli/workflow.mdx#terminate` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal workflow trace` | `temporal` | `docs/cli/workflow.mdx#trace` | [workflow-health.md](workflow-health.md) |
| `temporal workflow unpause` *(experimental)* | `temporal` | `docs/cli/workflow.mdx#unpause` | [batch-and-lifecycle.md](batch-and-lifecycle.md) |
| `temporal workflow update describe` | `temporal` | `docs/cli/workflow.mdx#describe` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow update execute` | `temporal` | `docs/cli/workflow.mdx#execute` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow update result` | `temporal` | `docs/cli/workflow.mdx#result` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow update start` | `temporal` | `docs/cli/workflow.mdx#start` | See skill-temporal-developer `cli-workflow-commands.md` |
| `temporal workflow update-options` | `temporal` | `docs/cli/workflow.mdx#update-options` | See skill-temporal-developer `cli-workflow-commands.md` |

## tcld

| Command | Binary | Docs source | Reference file |
|---------|--------|-------------|----------------|
| `tcld account audit-log kinesis create` | `tcld` | `docs/cloud/tcld/account.mdx#create` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log kinesis delete` | `tcld` | `docs/cloud/tcld/account.mdx#delete` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log kinesis get` | `tcld` | `docs/cloud/tcld/account.mdx#account-audit-log-kinesis-get` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log kinesis list` | `tcld` | `docs/cloud/tcld/account.mdx#list` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log kinesis update` | `tcld` | `docs/cloud/tcld/account.mdx#update` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log kinesis validate` | `tcld` | `docs/cloud/tcld/account.mdx#validate` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log pubsub create` | `tcld` | `docs/cloud/tcld/account.mdx#create` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log pubsub delete` | `tcld` | `docs/cloud/tcld/account.mdx#delete` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log pubsub get` | `tcld` | `docs/cloud/tcld/account.mdx#account-audit-log-pubsub-get` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log pubsub list` | `tcld` | `docs/cloud/tcld/account.mdx#list` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log pubsub update` | `tcld` | `docs/cloud/tcld/account.mdx#update` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account audit-log pubsub validate` | `tcld` | `docs/cloud/tcld/account.mdx#validate` | [cloud-audit-logs.md](cloud-audit-logs.md) |
| `tcld account get` | `tcld` | `docs/cloud/tcld/account.mdx#get` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld account list-regions` | `tcld` | `docs/cloud/tcld/account.mdx#list-regions` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld account metrics accepted-client-ca add` | `tcld` | `docs/cloud/tcld/account.mdx#add` | [cloud-certs.md](cloud-certs.md) |
| `tcld account metrics accepted-client-ca list` | `tcld` | `docs/cloud/tcld/account.mdx#list` | [cloud-certs.md](cloud-certs.md) |
| `tcld account metrics accepted-client-ca remove` | `tcld` | `docs/cloud/tcld/account.mdx#remove` | [cloud-certs.md](cloud-certs.md) |
| `tcld account metrics accepted-client-ca set` | `tcld` | `docs/cloud/tcld/account.mdx#set` | [cloud-certs.md](cloud-certs.md) |
| `tcld account metrics disable` | `tcld` | `docs/cloud/tcld/account.mdx#disable` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld account metrics enable` | `tcld` | `docs/cloud/tcld/account.mdx#enable` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld apikey create` | `tcld` | `docs/cloud/tcld/apikey.mdx#create` | [cloud-iam.md](cloud-iam.md) |
| `tcld apikey delete` | `tcld` | `docs/cloud/tcld/apikey.mdx#delete` | [cloud-iam.md](cloud-iam.md) |
| `tcld apikey disable` | `tcld` | `docs/cloud/tcld/apikey.mdx#disable` | [cloud-iam.md](cloud-iam.md) |
| `tcld apikey enable` | `tcld` | `docs/cloud/tcld/apikey.mdx#enable` | [cloud-iam.md](cloud-iam.md) |
| `tcld apikey get` | `tcld` | `docs/cloud/tcld/apikey.mdx#get` | [cloud-iam.md](cloud-iam.md) |
| `tcld apikey list` | `tcld` | `docs/cloud/tcld/apikey.mdx#list` | [cloud-iam.md](cloud-iam.md) |
| `tcld connectivity-rule create` | `tcld` | `docs/cloud/tcld/connectivity-rule.mdx#create` | [cloud-connectivity.md](cloud-connectivity.md) |
| `tcld connectivity-rule delete` | `tcld` | `docs/cloud/tcld/connectivity-rule.mdx#delete` | [cloud-connectivity.md](cloud-connectivity.md) |
| `tcld connectivity-rule get` | `tcld` | `docs/cloud/tcld/connectivity-rule.mdx#get` | [cloud-connectivity.md](cloud-connectivity.md) |
| `tcld connectivity-rule list` | `tcld` | `docs/cloud/tcld/connectivity-rule.mdx#list` | [cloud-connectivity.md](cloud-connectivity.md) |
| `tcld feature get` | `tcld` | `docs/cloud/tcld/feature.mdx#get` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld feature toggle` | `tcld` | `docs/cloud/tcld/feature.mdx#toggle` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld generate-certificates certificate-authority-certificate` | `tcld` | `docs/cloud/tcld/generate-certificates.mdx#certificate-authority-certificate` | [cloud-certs.md](cloud-certs.md) |
| `tcld generate-certificates end-entity-certificate` | `tcld` | `docs/cloud/tcld/generate-certificates.mdx#end-entity-certificate` | [cloud-certs.md](cloud-certs.md) |
| `tcld login` | `tcld` | `docs/cloud/tcld/login.mdx` (no subcommand; top-level) | [cloud-iam.md](cloud-iam.md) |
| `tcld logout` | `tcld` | `docs/cloud/tcld/logout.mdx` (no subcommand; top-level) | [cloud-iam.md](cloud-iam.md) |
| `tcld migration start` | `tcld` | `docs/cloud/migrate/automated.mdx` | [cloud-migration.md](cloud-migration.md) |
| `tcld migration get` | `tcld` | `docs/cloud/migrate/automated.mdx` | [cloud-migration.md](cloud-migration.md) |
| `tcld migration handover` | `tcld` | `docs/cloud/migrate/automated.mdx` | [cloud-migration.md](cloud-migration.md) |
| `tcld migration confirm` | `tcld` | `docs/cloud/migrate/automated.mdx` | [cloud-migration.md](cloud-migration.md) |
| `tcld migration abort` | `tcld` | `docs/cloud/migrate/automated.mdx` | [cloud-migration.md](cloud-migration.md) |
| `tcld namespace accepted-client-ca add` | `tcld` | `docs/cloud/tcld/namespace.mdx#add` | [cloud-certs.md](cloud-certs.md) |
| `tcld namespace accepted-client-ca list` | `tcld` | `docs/cloud/tcld/namespace.mdx#list` | [cloud-certs.md](cloud-certs.md) |
| `tcld namespace accepted-client-ca remove` | `tcld` | `docs/cloud/tcld/namespace.mdx#remove` | [cloud-certs.md](cloud-certs.md) |
| `tcld namespace accepted-client-ca set` | `tcld` | `docs/cloud/tcld/namespace.mdx#set` | [cloud-certs.md](cloud-certs.md) |
| `tcld namespace add-region` | `tcld` | `docs/cloud/tcld/namespace.mdx#add-region` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace certificate-filters add` | `tcld` | `docs/cloud/tcld/namespace.mdx#add` | [cloud-certs.md](cloud-certs.md) |
| `tcld namespace certificate-filters clear` | `tcld` | `docs/cloud/tcld/namespace.mdx#clear` | [cloud-certs.md](cloud-certs.md) |
| `tcld namespace certificate-filters export` | `tcld` | `docs/cloud/tcld/namespace.mdx#export` | [cloud-certs.md](cloud-certs.md) |
| `tcld namespace certificate-filters import` | `tcld` | `docs/cloud/tcld/namespace.mdx#import` | [cloud-certs.md](cloud-certs.md) |
| `tcld namespace create` | `tcld` | `docs/cloud/tcld/namespace.mdx#create` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace delete` | `tcld` | `docs/cloud/tcld/namespace.mdx#delete` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace delete-region` | `tcld` | `docs/cloud/tcld/namespace.mdx#delete-region` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace export s3 create` | `tcld` | `docs/cloud/tcld/namespace.mdx#create` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export s3 delete` | `tcld` | `docs/cloud/tcld/namespace.mdx#delete` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export s3 get` | `tcld` | `docs/cloud/tcld/namespace.mdx#get` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export s3 list` | `tcld` | `docs/cloud/tcld/namespace.mdx#list` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export s3 update` | `tcld` | `docs/cloud/tcld/namespace.mdx#update` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export s3 validate` | `tcld` | `docs/cloud/tcld/namespace.mdx#validate` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export gcs create` | `tcld` | `docs/cloud/gcp-export-gcs.mdx#using-tcld` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export gcs delete` | `tcld` | `docs/cloud/gcp-export-gcs.mdx#using-tcld` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export gcs get` | `tcld` | `docs/cloud/gcp-export-gcs.mdx#using-tcld` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export gcs list` | `tcld` | `docs/cloud/gcp-export-gcs.mdx#using-tcld` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export gcs update` | `tcld` | `docs/cloud/gcp-export-gcs.mdx#using-tcld` | [cloud-export.md](cloud-export.md) |
| `tcld namespace export gcs validate` | `tcld` | `docs/cloud/gcp-export-gcs.mdx#using-tcld` | [cloud-export.md](cloud-export.md) |
| `tcld namespace failover` | `tcld` | `docs/cloud/tcld/namespace.mdx#failover` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace get` | `tcld` | `docs/cloud/tcld/namespace.mdx#get` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace list` | `tcld` | `docs/cloud/tcld/namespace.mdx#list` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace retention get` | `tcld` | `docs/cloud/tcld/namespace.mdx#get` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace retention set` | `tcld` | `docs/cloud/tcld/namespace.mdx#set` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace search-attributes add` | `tcld` | `docs/cloud/tcld/namespace.mdx#add` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace search-attributes rename` | `tcld` | `docs/cloud/tcld/namespace.mdx#rename` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace set-connectivity-rules` | `tcld` | `docs/cloud/tcld/namespace.mdx#set-connectivity-rules` | [cloud-connectivity.md](cloud-connectivity.md) |
| `tcld namespace tags remove` | `tcld` | `docs/cloud/tcld/namespace.mdx#remove` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace tags upsert` | `tcld` | `docs/cloud/tcld/namespace.mdx#upsert` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld namespace update-codec-server` | `tcld` | `docs/cloud/tcld/namespace.mdx#update-codec-server` | [codec-server.md](codec-server.md) |
| `tcld namespace update-high-availability` | `tcld` | `docs/cloud/tcld/namespace.mdx#update-high-availability` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld nexus endpoint allowed-namespace add` | `tcld` | `docs/cloud/tcld/nexus.mdx#add` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld nexus endpoint allowed-namespace list` | `tcld` | `docs/cloud/tcld/nexus.mdx#list` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld nexus endpoint allowed-namespace remove` | `tcld` | `docs/cloud/tcld/nexus.mdx#remove` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld nexus endpoint allowed-namespace set` | `tcld` | `docs/cloud/tcld/nexus.mdx#set` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld nexus endpoint create` | `tcld` | `docs/cloud/tcld/nexus.mdx#create` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld nexus endpoint delete` | `tcld` | `docs/cloud/tcld/nexus.mdx#delete` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld nexus endpoint get` | `tcld` | `docs/cloud/tcld/nexus.mdx#get` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld nexus endpoint list` | `tcld` | `docs/cloud/tcld/nexus.mdx#list` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld nexus endpoint update` | `tcld` | `docs/cloud/tcld/nexus.mdx#update` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld request get` | `tcld` | `docs/cloud/tcld/request.mdx#get` | [cloud-namespace-admin.md](cloud-namespace-admin.md) |
| `tcld user delete` | `tcld` | `docs/cloud/tcld/user.mdx#delete` | [cloud-iam.md](cloud-iam.md) |
| `tcld user get` | `tcld` | `docs/cloud/tcld/user.mdx#get` | [cloud-iam.md](cloud-iam.md) |
| `tcld user invite` | `tcld` | `docs/cloud/tcld/user.mdx#invite` | [cloud-iam.md](cloud-iam.md) |
| `tcld user list` | `tcld` | `docs/cloud/tcld/user.mdx#list` | [cloud-iam.md](cloud-iam.md) |
| `tcld user resend-invite` | `tcld` | `docs/cloud/tcld/user.mdx#resend-invite` | [cloud-iam.md](cloud-iam.md) |
| `tcld user set-account-role` | `tcld` | `docs/cloud/tcld/user.mdx#set-account-role` | [cloud-iam.md](cloud-iam.md) |
| `tcld user set-namespace-permissions` | `tcld` | `docs/cloud/tcld/user.mdx#set-namespace-permissions` | [cloud-iam.md](cloud-iam.md) |
| `tcld user-group add-users` | `tcld` | `docs/cloud/tcld/user-group.mdx#add-users` | [cloud-iam.md](cloud-iam.md) |
| `tcld user-group create` | `tcld` | `docs/cloud/tcld/user-group.mdx#create` | [cloud-iam.md](cloud-iam.md) |
| `tcld user-group delete` | `tcld` | `docs/cloud/tcld/user-group.mdx#delete` | [cloud-iam.md](cloud-iam.md) |
| `tcld user-group get` | `tcld` | `docs/cloud/tcld/user-group.mdx#get` | [cloud-iam.md](cloud-iam.md) |
| `tcld user-group list` | `tcld` | `docs/cloud/tcld/user-group.mdx#list` | [cloud-iam.md](cloud-iam.md) |
| `tcld user-group list-members` | `tcld` | `docs/cloud/tcld/user-group.mdx#list-members` | [cloud-iam.md](cloud-iam.md) |
| `tcld user-group remove-users` | `tcld` | `docs/cloud/tcld/user-group.mdx#remove-users` | [cloud-iam.md](cloud-iam.md) |
| `tcld user-group set-access` | `tcld` | `docs/cloud/tcld/user-group.mdx#set-access` | [cloud-iam.md](cloud-iam.md) |
| `tcld version` | `tcld` | `docs/cloud/tcld/version.mdx` (no subcommand; top-level) | [cloud-namespace-admin.md](cloud-namespace-admin.md) |

## Notes on scope

- **`temporal` heading-tree caveat.** In `docs/cli/*.mdx`, each subcommand group is a top-level `## ` heading and the group's leaf subcommands are `### ` under it. The anchor slug is the leaf heading alone. In some groups (`workflow update`, `task-queue config`, `task-queue update-build-ids`, `task-queue versioning`), the leaf is another level deep; those rows still cite the leaf's own heading anchor.
- **Duplicate heading slugs inside one docs file** (e.g. `### describe` under `## cluster` and `### describe` under `## namespace` in `docs/cli/operator.mdx`). Read the `## <group>` section first, then scroll to the leaf.
- **`temporal operator namespace` vs `tcld namespace` are different commands.** The former registers a namespace inside a self-hosted cluster; the latter provisions one in Temporal Cloud. They are listed as separate rows pointing to different reference files.
- **Deprecated `temporal` commands** are tagged *(deprecated)*. `temporal task-queue update-build-ids *` and `temporal task-queue versioning *` carry deprecation banners in `docs/cli/task-queue.mdx`; they are replaced by the Worker Deployments surface (`temporal worker deployment *`), which is out of scope for ops -- see `skill-temporal-developer`.
- **Experimental `temporal` commands** are tagged *(experimental)*. `workflow pause`, `workflow unpause`, `workflow execute-update-with-start`, `workflow start-update-with-start` are marked experimental in `docs/cli/workflow.mdx`.
- **Developer-facing commands** (`temporal workflow start`, `temporal workflow execute`, `temporal workflow query`, `temporal workflow update *`, `temporal workflow signal`, `temporal workflow result`, `temporal workflow metadata`) are pointed to skill-temporal-developer `cli-workflow-commands.md` because they concern application-level workflow interaction rather than operational administration.
- **Top-level `tcld` commands without subcommands.** `tcld login`, `tcld logout`, and `tcld version` are leaves at the binary level.
- The six `tcld namespace export gcs *` rows cite `docs/cloud/gcp-export-gcs.mdx#using-tcld` because that section documents all six subcommands collectively.
- **Not listed here** (scoped out of ops): `temporal worker *` (worker and worker-deployment management -- belongs to `skill-temporal-developer`), `temporal server` subcommands other than `start-dev`, and the `temporal` top-level help/completion scaffolding.
