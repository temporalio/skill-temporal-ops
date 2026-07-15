# Cloud Migration

Three migration paths exist for Temporal workflows: <!-- docs/cloud/migrate/index.mdx:9 -->

| Path | From | To | Downtime |
|---|---|---|---|
| Automated | Self-hosted | Temporal Cloud | Zero <!-- docs/cloud/migrate/automated.mdx:30 --> |
| Manual | Self-hosted | Temporal Cloud | Varies <!-- docs/cloud/migrate/manual.mdx:24 --> |
| Within Cloud | Cloud region A | Cloud region B | Zero <!-- docs/cloud/migrate/migrate-within-cloud.mdx:27 --> |

---

## Automated Migration (Self-Hosted to Cloud)

Pre-release feature. Contact your Temporal account executive before planning. <!-- docs/cloud/migrate/automated.mdx:23 -->

Migrate in order of least-critical to most-critical namespace. Start with testing namespaces where downtime is acceptable. <!-- docs/cloud/migrate/automated.mdx:37-38 -->

### Limitations

- Self-hosted server version 1.22+ required <!-- docs/cloud/migrate/automated.mdx:414 -->
- History shard counts must be a power of two (e.g. 512, 1024) <!-- docs/cloud/migrate/automated.mdx:415 -->
- Multiple self-hosted servers with the same cluster name (default `active`) cannot connect to one migration server simultaneously; either migrate one at a time or use separate migration servers <!-- docs/cloud/migrate/automated.mdx:416-422 -->
- Cross-namespace commands (`system.enableCrossNamespaceCommands`) must be disabled and related code removed before migration <!-- docs/cloud/migrate/automated.mdx:429-431 -->
- Target cloud namespace must be empty (no workflows); cannot combine manual and auto migration <!-- docs/cloud/migrate/automated.mdx:513 -->
- If Global Namespace was previously enabled: Initial Failover Version must be <= 1,000,000 and Failover Version Increment must be a divisor of 1,000,000 <!-- docs/cloud/migrate/automated.mdx:425-427 -->

### Phase 1: Prepare

Collect data and submit to Temporal via support ticket: <!-- docs/cloud/migrate/automated.mdx:62-63 -->

**Cluster configuration** (run per cluster): <!-- docs/cloud/migrate/automated.mdx:68-69 -->
```
temporal operator cluster describe --address <frontend:7233> --output json   # server > 1.28.1
tctl --address <frontend:7233> admin cluster describe                        # server <= 1.28.1
```

**Custom search attributes** (must be Cloud-compatible): <!-- docs/cloud/migrate/automated.mdx:89-91 -->
```
temporal operator search-attribute list                          # Elasticsearch/OpenSearch
temporal operator search-attribute list --namespace="your_ns"    # SQL
```

**Namespace metrics**: total open/closed workflows, total storage, current retention policy, peak APS <!-- docs/cloud/migrate/automated.mdx:105-106 -->

**mTLS certificates** for S2S Proxy: `openssl verify -CAfile ca.pem client-cert.pem` <!-- docs/cloud/migrate/automated.mdx:117 -->

**Cloud namespaces**: create empty target namespaces, apply custom search attributes, adjust rate limits <!-- docs/cloud/migrate/automated.mdx:130-134 -->

**Submit CSV mapping** to Temporal: <!-- docs/cloud/migrate/automated.mdx:150-157 -->
```
cluster_name, cloud_region, source_namespace, cloud_namespace
cluster1,     us-east-1,    default,          use1.nnnnn
```

### Phase 2: Setup

Proceed only after Temporal approves the migration request. <!-- docs/cloud/migrate/automated.mdx:163-164 -->

**S2S Proxy deployment**: <!-- docs/cloud/migrate/automated.mdx:173 -->
1. Pull latest image from `temporalio/s2s-proxy` Docker Hub <!-- docs/cloud/migrate/automated.mdx:184-185 -->
2. Deploy 3 replicas (min 4 CPU, 512 MB memory per replica). Replica count must match cloud side. <!-- docs/cloud/migrate/automated.mdx:188-191 -->
3. Proxy initiates outbound TCP 8233 to cloud-side proxy. Ensure firewalls permit this. <!-- docs/cloud/migrate/automated.mdx:177-178 -->
4. Verify connectivity: <!-- docs/cloud/migrate/automated.mdx:194-196 -->
```
temporal operator cluster describe --address {proxy-external-address}
```

Monitor proxy health via Prometheus endpoint (`proxy-pod-ip:9090/metrics`), in particular `temporal_s2s_proxy_mux_connection_active`. <!-- docs/cloud/migrate/automated.mdx:200-205 -->

**Dynamic configuration changes**: <!-- docs/cloud/migrate/automated.mdx:211-212 -->

```yaml
frontend.keepAliveMaxConnectionAge:
  - value: '2h'
```

If Global Namespace was never enabled, enable it: set `clusterMetadata.enableGlobalNamespace: true`, `failoverVersionIncrement: 1000000` (coordinate with Temporal), `initialFailoverVersion: <2-99>` (unique per cluster), and `dcRedirectionPolicy.policy: 'all-apis-forwarding'`. <!-- docs/cloud/migrate/automated.mdx:229-248 -->

Restart all services (frontend first, then history, matching, worker) and verify with `temporal operator cluster describe`. <!-- docs/cloud/migrate/automated.mdx:231,254 -->

For server versions 1.22.x-1.23.x, also enable stream-based replication: <!-- docs/cloud/migrate/automated.mdx:436-439 -->
```yaml
history.enableReplicationStream:
  - value: true
```

Verify your persistence/database layer has sufficient CPU and I/O capacity. <!-- docs/cloud/migrate/automated.mdx:265-266 -->

### Phase 3: Test

Use a non-production namespace that can tolerate data loss. Run a mix of completed, active, and new workflows. Perform a full end-to-end migration. Testing succeeds if all data migrates to Cloud. <!-- docs/cloud/migrate/automated.mdx:270-283 -->

### Phase 4: Initiate

**Start migration** (Temporal generates the endpoint-id): <!-- docs/cloud/migrate/automated.mdx:289-290 -->
```
tcld migration start --endpoint-id <endpoint-id> --source-namespace <source-namespace> --target-namespace <target-namespace>
```

Self-hosted namespace is active, cloud namespace is passive. Workflows replicate self-hosted to cloud. <!-- docs/cloud/migrate/automated.mdx:290-292 -->

Billing for the cloud namespace does not begin until migration is confirmed. <!-- docs/cloud/migrate/automated.mdx:296-298 -->

**Monitor progress**: <!-- docs/cloud/migrate/automated.mdx:310-311 -->
```
tcld migration get --id <migration-id>
```
Also monitor `replication_stream_stuck` metric from self-hosted side. <!-- docs/cloud/migrate/automated.mdx:311 -->

**Handover to Cloud**: <!-- docs/cloud/migrate/automated.mdx:323-324 -->
```
tcld migration handover --id <migration-id> --to-replica-id cloud
```
Cloud becomes active, self-hosted becomes passive. To hand back to self-hosted: <!-- docs/cloud/migrate/automated.mdx:334-335 -->
```
tcld migration handover --id <migration-id> --to-replica-id on-prem
```

### Phase 5: Finalize

Complete client transfer, then validate: confirm worker access to cloud namespaces, verify metrics access, monitor schedule-to-start latency / start vs. completion rate / sync match rate, and plan a worker tuning session (performance may differ). <!-- docs/cloud/migrate/automated.mdx:339-351 -->

**Confirm migration** (final, cannot be undone; halts replication): <!-- docs/cloud/migrate/automated.mdx:355-357 -->
```
tcld migration confirm --id <migration-id>
```

**Abort migration** (rolls back without impacting workflows): <!-- docs/cloud/migrate/automated.mdx:366-367 -->
```
tcld migration abort --id <migration-id>
```

### Transfer Clients to Cloud

**Option 1 (recommended)**: Deploy two sets of clients, one pointing to self-hosted and one to Cloud. <!-- docs/cloud/migrate/automated.mdx:378-379 -->
1. Cloud clients connect and poll but receive no tasks initially
2. Start migration: self-hosted active, cloud passive. Cloud client requests forward to self-hosted automatically
3. Handover: cloud active, self-hosted passive. Self-hosted client requests forward to cloud automatically
4. Confirm migration: self-hosted clients stop receiving tasks; shut them down

**Option 2**: Single set of clients, switch endpoint during migration. Risk: if workers are misconfigured during switch, workflows stop making progress. <!-- docs/cloud/migrate/automated.mdx:395-396 -->

### Key Facts

All workflows migrate by default; for closed workflows you may specify a date range (top speed optimization). <!-- docs/cloud/migrate/automated.mdx:521-527 --> Schedules are supported. <!-- docs/cloud/migrate/automated.mdx:529 --> Cannot split one source namespace into multiple cloud namespaces. <!-- docs/cloud/migrate/automated.mdx:509 --> Encrypted payloads remain encrypted through migration. <!-- docs/cloud/migrate/automated.mdx:538-539 -->

---

## Manual Migration (Self-Hosted to Cloud)

Use when automated migration requirements are not met or when migration scope is smaller. <!-- docs/cloud/migrate/manual.mdx:19-20 -->

### Client Code Changes

Update Worker and Starter connection code: <!-- docs/cloud/migrate/manual.mdx:37-38 -->
- Add SSL certificate and private key associated with the namespace <!-- docs/cloud/migrate/manual.mdx:42 -->
- Set gRPC endpoint to `<namespace_id>.<account_id>.tmprl.cloud:port` <!-- docs/cloud/migrate/manual.mdx:43-44 -->
- Configure `tcld` with the same address, namespace, and certificate <!-- docs/cloud/migrate/manual.mdx:46-47 -->

### Workflow Execution Strategies

**New workflows**: Once updated client code is deployed, new executions automatically go to Cloud. Maintain the self-hosted client as long as you need to send Signals or Queries to old executions. <!-- docs/cloud/migrate/manual.mdx:30-31 -->

**Running workflows**: <!-- docs/cloud/migrate/manual.mdx:31-32 -->
- Short-running: drain (let them complete), then restart on Cloud
- Long-running / continuous: cancel and pass current state to a new workflow on Cloud. Example implementation: `github.com/temporalio/temporal-migration` (Java) <!-- docs/cloud/migrate/manual.mdx:65 -->

During live migration, a Signal and Query execute per workflow. The Query API loads the full history into Workers. Ensure self-hosted Worker capacity supports this memory load. <!-- docs/cloud/migrate/manual.mdx:68-71 -->

**Completed workflows**: Execution history cannot be automatically migrated to Cloud via manual migration. Maintain self-hosted access or export JSON for analytics. <!-- docs/cloud/migrate/manual.mdx:32-33 -->

### Considerations When Resuming Workflows

- **Idempotency**: Determine whether to skip non-idempotent steps when resuming <!-- docs/cloud/migrate/manual.mdx:75 -->
- **Elapsed time**: Calculate sleep deltas for resumed executions <!-- docs/cloud/migrate/manual.mdx:76 -->
- **Child workflows**: Pass child state into parent to resume children correctly; parent/child relationship does not carry over <!-- docs/cloud/migrate/manual.mdx:77 -->
- **Heartbeat state**: Long-running activities relying on heartbeat details will not receive latest details in target namespace <!-- docs/cloud/migrate/manual.mdx:78-80 -->
- **Signal handling**: Handle `NotFound` when signaling between workflows; they may resume out of order <!-- docs/cloud/migrate/manual.mdx:82 -->
- **Duration between awaitables**: Factor elapsed time accuracy for sleeps between awaitables <!-- docs/cloud/migrate/manual.mdx:81 -->

### Other Considerations

- Add mTLS certificate to Cloud namespace <!-- docs/cloud/migrate/manual.mdx:86 -->
- Metrics differ between self-hosted and Cloud; review Cloud metrics documentation <!-- docs/cloud/migrate/manual.mdx:87 -->
- Review security and access implications <!-- docs/cloud/migrate/manual.mdx:88 -->
- Review current APS load with your AE/SA to set appropriate namespace limits <!-- docs/cloud/migrate/manual.mdx:89 -->

---

## Migrate Within Cloud (Region to Region)

Uses Temporal Cloud High Availability features. Zero downtime. <!-- docs/cloud/migrate/migrate-within-cloud.mdx:27 -->

HA features affect pricing. <!-- docs/cloud/migrate/migrate-within-cloud.mdx:35 -->

### Prerequisites

- Namespaces using Export must stop Export and reconfigure for the new region before migration <!-- docs/cloud/migrate/migrate-within-cloud.mdx:31-32 -->
- If workers use API key authentication, update all client code to use the regional endpoint of the new replica <!-- docs/cloud/migrate/migrate-within-cloud.mdx:53 -->

### Migration Steps

1. **Add replica** in target region (see available regions and supported multi-region/multi-cloud configurations) <!-- docs/cloud/migrate/migrate-within-cloud.mdx:38-39 -->
2. **Wait** for the replica to become active. Cloud UI shows a time estimate; namespace admins receive an email on completion. <!-- docs/cloud/migrate/migrate-within-cloud.mdx:52 -->
3. **Update workers** (API key auth only) to use the new region's regional endpoint <!-- docs/cloud/migrate/migrate-within-cloud.mdx:53 -->
4. **Failover** to the new region via Cloud UI <!-- docs/cloud/migrate/migrate-within-cloud.mdx:54 -->
5. **Remove** the original region's replica <!-- docs/cloud/migrate/migrate-within-cloud.mdx:65 -->

If using API keys for worker authentication, removing the replica requires a support ticket. <!-- docs/cloud/migrate/migrate-within-cloud.mdx:66-67 -->

All replica changes are subject to a cooldown period before further changes can be made. <!-- docs/cloud/migrate/migrate-within-cloud.mdx:75-77 -->

---

## tcld Migration Command Reference

| Command | Purpose |
|---|---|
| `tcld migration start --endpoint-id <id> --source-namespace <ns> --target-namespace <ns>` | Begin migration <!-- docs/cloud/migrate/automated.mdx:305 --> |
| `tcld migration get --id <migration-id>` | Check migration status <!-- docs/cloud/migrate/automated.mdx:317 --> |
| `tcld migration list` (alias `l`) | List all migrations (no flags) <!-- docs/cloud/tcld/migration.mdx:41 --> |
| `tcld migration handover --id <migration-id> --to-replica-id cloud` | Hand over to Cloud <!-- docs/cloud/migrate/automated.mdx:332 --> |
| `tcld migration handover --id <migration-id> --to-replica-id on-prem` | Hand back to self-hosted <!-- docs/cloud/migrate/automated.mdx:335 --> |
| `tcld migration confirm --id <migration-id>` | Finalize (irreversible) <!-- docs/cloud/migrate/automated.mdx:362 --> |
| `tcld migration abort --id <migration-id>` | Abort and roll back <!-- docs/cloud/migrate/automated.mdx:369 --> |

`start`, `handover`, `confirm`, and `abort` accept an optional `--request-id`/`-r`; the server assigns one if unset. <!-- docs/cloud/tcld/migration.mdx:53 -->

---

## Troubleshooting

**S2S Proxy not connecting**
- Verify outbound TCP 8233 is open through firewalls <!-- docs/cloud/migrate/automated.mdx:177-178 -->
- Check `temporal_s2s_proxy_mux_connection_active` metric on `proxy-pod-ip:9090/metrics` <!-- docs/cloud/migrate/automated.mdx:203-205 -->
- Confirm replica count matches between self-hosted and cloud-side proxy <!-- docs/cloud/migrate/automated.mdx:190-191 -->

**Replication appears stuck**
- Monitor `replication_stream_stuck` metric from self-hosted side <!-- docs/cloud/migrate/automated.mdx:311 -->
- For server 1.22.x-1.23.x, ensure `history.enableReplicationStream` is set to `true` and history pods are restarted <!-- docs/cloud/migrate/automated.mdx:443-445 -->

**Cluster name collision**
- Multiple self-hosted servers with the same cluster name (default `active`) cannot use the same migration server. Migrate one at a time or use separate migration servers. <!-- docs/cloud/migrate/automated.mdx:416-422 -->

**Workflows not progressing after handover**
- Option 2 (single client set) risk: if workers are misconfigured during endpoint switch, workflows stop. Verify all workers connect to Cloud before handover. <!-- docs/cloud/migrate/automated.mdx:395-397 -->
- Option 1 (dual client set) is recommended to avoid this scenario <!-- docs/cloud/migrate/automated.mdx:378 -->

**Manual migration: Query overloading Workers**
- Query API loads full workflow history into Worker memory. Ensure capacity before migrating large numbers of workflows via `ListFilter`. <!-- docs/cloud/migrate/manual.mdx:69-71 -->

**Within-Cloud: Cannot remove replica**
- If using API keys for worker auth, a support ticket is required to remove the replica <!-- docs/cloud/migrate/migrate-within-cloud.mdx:66-67 -->
- Replica changes are subject to a cooldown period <!-- docs/cloud/migrate/migrate-within-cloud.mdx:75-77 -->
