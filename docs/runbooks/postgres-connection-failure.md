# Runbook: PostgreSQL Connection Failure Diagnosis

**Service:** Zava API (AKS) → PostgreSQL Flexible Server  
**Last updated:** 2026-05-29  
**Related incidents:** [#9](https://github.com/sorgina13/grubify/issues/9), [#10](https://github.com/sorgina13/grubify/issues/10)

---

## Symptoms

- `zava-api` logs: `connect ECONNREFUSED <ip>:5432`
- `zava-api` logs: `timeout exceeded when trying to connect`
- `zava-api` logs: `PostgreSQL error of type 'error' occurred (code: 28P01)`
- HTTP 5xx on `/api/health`, `/api/products`, `/api/products/category/*`
- Alert: `postgres-server-down` (Sev1) or `Zava-http-5xx-errors` (Sev2)

---

## Decision Tree

```
PostgreSQL connection failures?
│
├─ Error: ECONNREFUSED
│   ├─ 1. Check server state
│   │     az postgres flexible-server show -n zava-pg-sqzlc6 -g rg-zava-aks-postgres --query state
│   │
│   ├─ If state = Stopped
│   │     → Server was stopped (manually or scheduled)
│   │     → Check activity logs for stop action (see Step A below)
│   │     → Remediate: az postgres flexible-server start -n zava-pg-sqzlc6 -g rg-zava-aks-postgres
│   │
│   ├─ If state = Ready but ECONNREFUSED persists
│   │     → Check NSG rules on db-subnet for port 5432 blocks
│   │     → Check private DNS zone resolution (sqzlc6.private.postgres.database.azure.com)
│   │     → Check VNet peering / delegated subnet connectivity
│   │
│   └─ If state = Starting
│         → Server is coming back up; wait 2-3 minutes and re-check
│
├─ Error: timeout exceeded when trying to connect
│   │  (Same as ECONNREFUSED but TCP SYN gets no RST — server unreachable)
│   └─ Follow ECONNREFUSED path above
│
├─ Error: code 28P01 (password authentication failed)
│   ├─ 2. Check auth configuration
│   │     az postgres flexible-server show -n zava-pg-sqzlc6 -g rg-zava-aks-postgres --query authConfig
│   │
│   ├─ If passwordAuth = Disabled and app uses password auth
│   │     → Re-enable password auth OR migrate app to Entra ID auth
│   │     → Check activity logs for auth config changes (see Step B below)
│   │
│   └─ If passwordAuth = Enabled
│         → Verify connection string credentials in app config / K8s secrets
│         → Check if password was recently rotated
│
└─ Error: CredentialUnavailableError (EnvironmentCredential)
      → App is configured for AAD/Entra auth but credentials are not available
      → Check workload identity configuration on AKS pods
      → Verify managed identity is assigned and has DB access
```

---

## Step A: Check Activity Logs for Server Stop/Start

```bash
az monitor activity-log list \
  --resource-group rg-zava-aks-postgres \
  --subscription 325177b6-ea0e-4cae-9862-db57a44eee60 \
  --offset 2h \
  --query "[?contains(operationName.value, 'flexibleServers/stop') || contains(operationName.value, 'flexibleServers/start')].{time:eventTimestamp, op:operationName.value, status:status.value, caller:caller}" \
  -o table
```

## Step B: Check Activity Logs for Auth Config Changes

```bash
az monitor activity-log list \
  --resource-group rg-zava-aks-postgres \
  --subscription 325177b6-ea0e-4cae-9862-db57a44eee60 \
  --offset 2h \
  --query "[?contains(operationName.value, 'flexibleServers/administrators') || contains(operationName.value, 'flexibleServers/write')].{time:eventTimestamp, op:operationName.value, status:status.value, caller:caller}" \
  -o table
```

---

## Log Analytics Queries

### ECONNREFUSED errors (AKS ContainerLog table)

```kql
ContainerLog
| where LogEntry has "ECONNREFUSED"
| summarize ErrorCount=count() by bin(TimeGenerated, 1m)
| order by TimeGenerated asc
```

### All PostgreSQL connection errors by type

```kql
ContainerLog
| where LogEntry has_any ("ECONNREFUSED", "timeout exceeded", "28P01", "password authentication")
| summarize ErrorCount=count() by bin(TimeGenerated, 1m), ErrorType=case(
    LogEntry has "ECONNREFUSED", "ECONNREFUSED",
    LogEntry has "timeout exceeded", "ConnectionTimeout",
    LogEntry has "28P01", "AuthError_28P01",
    "OtherError")
| order by TimeGenerated asc
```

### App Insights request success rate

```kql
requests
| summarize TotalRequests=count(), FailedRequests=countif(success == false),
    SuccessRate=round(100.0 * countif(success == true) / count(), 2)
  by bin(timestamp, 1m)
| order by timestamp asc
```

---

## Error Transition Pattern (Server Lifecycle)

Understanding the error type helps pinpoint where the server is in its lifecycle:

| Server State | Error Seen | Explanation |
|---|---|---|
| Stopping | `timeout exceeded` | Existing connections hang; TCP SYN gets no response |
| Stopped | `timeout exceeded` | Same — no process listening, no RST sent |
| Starting | `ECONNREFUSED` | TCP port not yet bound; kernel sends RST |
| Ready | None (or `28P01`) | TCP connects; auth layer may still fail |

---

## Key Resources

| Resource | Value |
|---|---|
| PostgreSQL Server | `zava-pg-sqzlc6` (private IP: `10.1.0.4`) |
| AKS Cluster | `aks-Zava-sqzlc6` |
| App Insights | `ai-Zava-sqzlc6` |
| Log Analytics (Zava) | Workspace ID: `81fbaac6-0e2d-4175-b3bb-990c2ae3e9ba` |
| Resource Group | `rg-zava-aks-postgres` |
| Subscription | `325177b6-ea0e-4cae-9862-db57a44eee60` |
| Private DNS Zone | `sqzlc6.private.postgres.database.azure.com` |
| DB Subnet | `vnet-Zava-sqzlc6/db-subnet` |

---

## Incident History

| Date | Issue | Root Cause | Caused By |
|---|---|---|---|
| 2026-05-28 | [#9](https://github.com/sorgina13/grubify/issues/9) | `passwordAuth` disabled on PostgreSQL server | `loreaa` |
| 2026-05-29 | [#10](https://github.com/sorgina13/grubify/issues/10) | PostgreSQL server manually stopped | `loreaa` |

---

## Prevention Recommendations

1. **RBAC:** Restrict `Microsoft.DBforPostgreSQL/flexibleServers/stop/action` to break-glass accounts only
2. **Azure Policy:** Deny `flexibleServers/stop/action` on production resource groups
3. **Resource Lock:** Add `CanNotDelete` lock on `zava-pg-sqzlc6`
4. **Activity Log Alert:** Trigger immediately on `flexibleServers/stop/action` (faster than log-based alerts)
5. **App Resilience:** Implement connection retry with exponential backoff in `zava-api`
