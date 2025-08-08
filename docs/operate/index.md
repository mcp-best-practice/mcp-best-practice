# Operations Guide

## Operating MCP Servers in Production

Effective operations ensure MCP servers maintain high availability, performance, and reliability in production environments.

## Operational Excellence

### Key Principles
1. **Observability** - Monitor, log, trace everything
2. **Automation** - Automate repetitive tasks
3. **Reliability** - Design for failure
4. **Performance** - Optimize continuously
5. **Security** - Defense in depth

## Operational Readiness

### Pre-Production Checklist
- [ ] Monitoring dashboards configured
- [ ] Alerts and notifications set up
- [ ] Logging pipeline established
- [ ] Backup and recovery tested
- [ ] Runbooks documented
- [ ] On-call rotation scheduled
- [ ] Incident response plan ready
- [ ] Capacity planning completed

## Monitoring Strategy

### Four Golden Signals
1. **Latency** - Response time
2. **Traffic** - Request rate
3. **Errors** - Failure rate
4. **Saturation** - Resource utilization

### Key Metrics
```python
# Application metrics
mcp_requests_total
mcp_request_duration_seconds
mcp_errors_total
mcp_active_connections

# System metrics
cpu_utilization_percent
memory_usage_bytes
disk_io_operations
network_throughput_bytes

# Business metrics
tools_executed_total
resources_accessed_total
user_sessions_active
api_quota_remaining
```

## Health Checks

### Liveness Probe
```python
@app.route('/health/live')
def liveness():
    """Basic liveness check - is the process running?"""
    return {'status': 'alive'}, 200
```

### Readiness Probe
```python
@app.route('/health/ready')
def readiness():
    """Readiness check - can we serve traffic?"""
    checks = {
        'database': check_database_connection(),
        'cache': check_cache_connection(),
        'dependencies': check_external_services()
    }
    
    if all(checks.values()):
        return {'status': 'ready', 'checks': checks}, 200
    else:
        return {'status': 'not_ready', 'checks': checks}, 503
```

## Logging Best Practices

### Structured Logging
```python
import structlog

logger = structlog.get_logger()

def process_request(request_id, tool_name, params):
    logger.info(
        "request_started",
        request_id=request_id,
        tool=tool_name,
        params_size=len(str(params))
    )
    
    try:
        result = execute_tool(tool_name, params)
        logger.info(
            "request_completed",
            request_id=request_id,
            tool=tool_name,
            duration_ms=elapsed_time,
            result_size=len(str(result))
        )
        return result
        
    except Exception as e:
        logger.error(
            "request_failed",
            request_id=request_id,
            tool=tool_name,
            error=str(e),
            error_type=type(e).__name__
        )
        raise
```

### Log Aggregation
```yaml
# Fluentd configuration
<source>
  @type tail
  path /var/log/mcp-server/*.log
  pos_file /var/log/td-agent/mcp-server.pos
  tag mcp.server
  <parse>
    @type json
  </parse>
</source>

<match mcp.**>
  @type elasticsearch
  host elasticsearch.example.com
  port 9200
  logstash_format true
  logstash_prefix mcp
  <buffer>
    @type file
    path /var/log/td-agent/buffer/mcp
    flush_interval 10s
  </buffer>
</match>
```

## Performance Management

### Performance Baseline
```python
# Establish baseline metrics
PERFORMANCE_BASELINE = {
    'p50_latency_ms': 100,
    'p95_latency_ms': 500,
    'p99_latency_ms': 1000,
    'error_rate_percent': 0.1,
    'requests_per_second': 1000
}

def check_performance():
    current_metrics = get_current_metrics()
    
    for metric, baseline in PERFORMANCE_BASELINE.items():
        current = current_metrics[metric]
        if current > baseline * 1.5:  # 50% degradation
            alert(f"Performance degradation: {metric} = {current}")
```

### Performance Tuning
```python
# Connection pooling
from sqlalchemy import create_engine

engine = create_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
    pool_recycle=3600
)

# Caching strategy
from functools import lru_cache
import redis

redis_client = redis.Redis(decode_responses=True)

@lru_cache(maxsize=1000)
def get_cached_result(key):
    # Memory cache first
    result = redis_client.get(key)
    if result:
        return json.loads(result)
    
    # Compute if not cached
    result = compute_expensive_operation(key)
    redis_client.setex(key, 3600, json.dumps(result))
    return result
```

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