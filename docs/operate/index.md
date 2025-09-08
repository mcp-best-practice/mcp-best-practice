# Operate (Lifecycle & Management)

Operate MCP servers with consistent lifecycle management, observability, and governance. This page combines operations and management into one high‑level guide.

## Operational Excellence

### Key Principles
- Observability: standardized logs, metrics, and tracing; correlate via request IDs and gateway.
- Automation: automate rollouts, scaling, and remediation; codify runbooks.
- Reliability: design for failure; use circuit breakers and graceful degradation.
- Performance: set SLOs; monitor and optimize continuously.
- Security: defense in depth at runtime (policies, approvals, least privilege).

## Operational Readiness

### Pre-Production Checklist
- Monitoring dashboards and alerts configured
- Logging, tracing, and metrics standardized (OpenTelemetry)
- Backup and recovery tested for any externalized state
- Runbooks documented; on‑call rotation and incident response in place
- Capacity planning completed (load/burst and dependency failure modes tested)

## Monitoring Strategy

### Signals and Metrics
- Four signals: latency, traffic, errors, saturation
- Key metrics: tool success rate, p95 latency, error classes, approvals/policy denials, cost/quotas, dependency health

Health/readiness endpoints should include dependency checks with timeouts and fallbacks.

## Health Checks

## Logging and Aggregation
- Prefer structured logs; ensure correlation IDs are present and consistent.
- Centralize log aggregation; redact sensitive fields; retain searchable audits (who/what/when/why).

## Logging Best Practices

## Lifecycle Management
- Curated catalog: only deploy approved servers; require SBOMs, provenance, signatures, and scans.
- Approval workflow: registration, promotion, ownership, SLOs; immutable audit trails.
- Multi‑tenancy: per‑tenant isolation for configs, logs, metrics, and limits; tenant‑aware authZ.
- External/remote servers: govern through a gateway/proxy; document constraints and SLAs.

## Performance Management

## SLOs and Incident Response
- Baselines (tune to context): tool success rate ≥ 99.0% monthly; p95 ≤ 400 ms.
- Incident taxonomy: availability, latency, correctness, security, quota/budget.
- Flow: detect → triage (blast radius) → mitigate (degrade/circuit break) → RCA → fix/rollback → catalog learnings.

## Tooling and Automation
- Prefer a single operator/controller framework across teams; standardize Helm/operator usage.
- Version operations configs; keep runbooks up to date; exercise disaster recovery basics.

## Capacity Planning

### Resource Calculation
```python
def calculate_capacity():
    # Current metrics
    avg_request_time = 100  # ms
    peak_rps = 1000
    cpu_per_request = 0.01  # cores
    memory_per_connection = 10  # MB
    
    # Required resources
    required_cores = peak_rps * cpu_per_request * 1.5  # 50% buffer
    required_memory = peak_rps * memory_per_connection * 1.5
    
    # Instance sizing
    instances_needed = math.ceil(required_cores / 4)  # 4 cores per instance
    
    return {
        'cores_needed': required_cores,
        'memory_needed_mb': required_memory,
        'instances': instances_needed
    }
```

## Backup and Recovery

### Backup Strategy
```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/backups/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# Database backup
pg_dump $DATABASE_URL > "$BACKUP_DIR/database.sql"

# Configuration backup
tar -czf "$BACKUP_DIR/config.tar.gz" /etc/mcp-server/

# Application state backup
redis-cli --rdb "$BACKUP_DIR/redis.rdb"

# Upload to S3
aws s3 sync "$BACKUP_DIR" "s3://backups/mcp-server/$(date +%Y%m%d)/"

# Cleanup old backups (keep 30 days)
find /backups -type d -mtime +30 -exec rm -rf {} +
```

### Disaster Recovery
```python
class DisasterRecovery:
    def __init__(self):
        self.backup_location = "s3://backups/mcp-server/"
        self.recovery_point_objective = timedelta(hours=1)
        self.recovery_time_objective = timedelta(minutes=30)
    
    def initiate_recovery(self, target_time):
        # Find nearest backup
        backup = self.find_backup(target_time)
        
        # Restore database
        self.restore_database(backup['database'])
        
        # Restore configuration
        self.restore_config(backup['config'])
        
        # Restore cache if needed
        if self.should_restore_cache():
            self.restore_cache(backup['cache'])
        
        # Verify recovery
        if self.verify_recovery():
            logger.info("Recovery successful")
            return True
        else:
            logger.error("Recovery failed")
            return False
```

## Maintenance Windows

### Zero-Downtime Maintenance
```python
def rolling_update(new_version):
    instances = get_all_instances()
    batch_size = max(1, len(instances) // 4)  # Update 25% at a time
    
    for batch in chunks(instances, batch_size):
        # Remove from load balancer
        for instance in batch:
            remove_from_lb(instance)
        
        # Wait for connections to drain
        time.sleep(30)
        
        # Update instances
        for instance in batch:
            update_instance(instance, new_version)
        
        # Health check
        for instance in batch:
            wait_for_healthy(instance)
        
        # Add back to load balancer
        for instance in batch:
            add_to_lb(instance)
        
        # Monitor for issues
        time.sleep(60)
        if detect_issues():
            rollback()
            return False
    
    return True
```

## Troubleshooting

### Common Issues and Solutions

#### High Latency
```bash
# Check slow queries
SELECT query, mean_time, calls
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

# Check connection pool
netstat -an | grep :8000 | wc -l

# Check CPU throttling
top -H -p $(pgrep mcp-server)
```

#### Memory Leaks
```python
import tracemalloc
import gc

# Start tracing
tracemalloc.start()

# Take snapshot
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

for stat in top_stats[:10]:
    print(stat)

# Force garbage collection
gc.collect()
```

#### Connection Errors
```bash
# Check network connectivity
ping -c 4 database.example.com
traceroute database.example.com

# Check DNS resolution
nslookup database.example.com

# Check firewall rules
iptables -L -n -v
```

## Runbooks

### Sample Runbook: High Error Rate
```markdown
## Alert: High Error Rate

### Symptoms
- Error rate > 1%
- HTTP 5xx responses increasing

### Initial Response (5 min)
1. Check dashboards for error patterns
2. Review recent deployments
3. Check system resources

### Investigation (15 min)
1. Analyze error logs
2. Check database connectivity
3. Verify external dependencies
4. Review recent changes

### Mitigation
1. If deployment issue: Rollback
2. If resource issue: Scale up
3. If dependency issue: Enable circuit breaker
4. If database issue: Failover to replica

### Resolution
1. Fix root cause
2. Deploy fix
3. Monitor for recurrence
4. Update documentation

### Post-Incident
1. Write incident report
2. Update runbook if needed
3. Schedule post-mortem
```

## Compliance and Auditing

### Audit Logging
```python
def audit_log(event_type, user, action, resource, result):
    audit_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'event_type': event_type,
        'user': user,
        'action': action,
        'resource': resource,
        'result': result,
        'ip_address': request.remote_addr,
        'user_agent': request.user_agent.string
    }
    
    # Write to audit log
    audit_logger.info(json.dumps(audit_entry))
    
    # Store in audit database
    audit_db.insert(audit_entry)
```

## Next Steps

- 📊 [Monitoring Deep Dive](monitoring.md)
- 📜 [Logging Strategy](logging.md)
- 🎯 [Performance Optimization](performance.md)
- 🌡️ [Scaling Operations](scaling.md)
- 🎆 [Incident Response](incidents.md)
