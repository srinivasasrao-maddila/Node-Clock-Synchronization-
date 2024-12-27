# Node Clock Synchronization Monitoring in EKS
This guide covers monitoring and troubleshooting clock synchronization issues in Amazon EKS nodes using New Relic monitoring.

## Overview
Clock synchronization is critical for distributed systems. Unsynchronized clocks can cause various issues:
- Certificate validation failures
- Authentication problems
- Inconsistent logs and metrics
- Database transaction anomalies
- Service mesh timing issues

## Alert Conditions

### Critical Alert Criteria
- **Synchronization Status**: Remains at 0 (unsynchronized) for > 2 minutes
- **Maximum Clock Drift**: Exceeds 16 seconds
- **Impact**: High - affects system-wide operations

### Warning Alert Criteria
- **Synchronization Status**: Fluctuating between synchronized/unsynchronized
- **Clock Drift**: Between 1-16 seconds
- **Impact**: Medium - potential for timing-related issues

## Monitoring Setup

### 1. Prerequisites
- New Relic Infrastructure Agent installed on EKS nodes
- Access to node system configuration
- Appropriate IAM permissions

### 2. Implementation Steps

#### 2.1 Create Monitoring Configuration
Create a new file: `/etc/newrelic-infra/integrations.d/clock-sync.yml`

```yaml
integrations:
  - name: nri-flex
    config:
      name: clockDriftMonitor
      apis:
        - name: clockDrift
          commands:
            - run: |
                # Get NTP server info
                ntp_server=$(chronyc tracking | grep "Reference ID" | 
                  awk 'match($0,/\((.*)\)/) {print substr($0,RSTART+1,RLENGTH-2)}')
                
                # Calculate drift in milliseconds
                drift_ms=$(printf '%.0f\n' $(echo "$(chronyc tracking | 
                  grep "Last offset" | 
                  awk 'match($0,/(\+|\-).*(\s)/) {print substr($0,RSTART+1,RLENGTH-2)}')*1000" | bc -l))
                
                # Get sync status
                sync_status=$(chronyc tracking | 
                  grep "Leap status" | 
                  awk '{print $4}')
                
                echo "{
                  \"ntp.server\": \"$ntp_server\",
                  \"clock.drift_ms\": $drift_ms,
                  \"sync.status\": \"$sync_status\"
                }"
          event_type: NodeClockSync
```

#### 2.2 Configure Alert Policies

Create the following NRQL alert conditions in New Relic:

```sql
-- Critical Alert: High Clock Drift
SELECT max(clock.drift_ms) 
FROM NodeClockSync 
WHERE sync.status != 'Normal' 
FACET hostname 
EVALUATE EVERY 5 minutes 
ALERT WHEN max > 16000 FOR 2 minutes

-- Warning Alert: Moderate Clock Drift
SELECT max(clock.drift_ms) 
FROM NodeClockSync 
FACET hostname 
EVALUATE EVERY 5 minutes 
ALERT WHEN max > 1000 FOR 5 minutes
```

### 3. Monitoring Dashboard

Create a dashboard with these key metrics:

```sql
-- Clock Drift Over Time
FROM NodeClockSync SELECT 
  average(clock.drift_ms), 
  max(clock.drift_ms) 
FACET hostname 
TIMESERIES

-- Sync Status Distribution
FROM NodeClockSync SELECT 
  count(*) 
FACET sync.status, hostname 
TIMESERIES 1 minute
```

## Troubleshooting Guide

### Common Issues and Solutions

1. **High Clock Drift**
   - Check NTP server accessibility
   - Verify network latency to NTP servers
   - Review chrony configuration

   ```bash
   # Check chrony status
   chronyc tracking
   chronyc sources -v
   ```

2. **Persistent Unsynchronized Status**
   - Verify chrony service status
   - Check firewall rules for NTP traffic
   - Review system logs

   ```bash
   systemctl status chronyd
   journalctl -u chronyd
   ```

3. **Network-Related Issues**
   - Test NTP server connectivity
   - Verify UDP port 123 accessibility
   - Check network policies

   ```bash
   # Test NTP server connectivity
   nc -uvz pool.ntp.org 123
   ```

## Best Practices

1. **NTP Configuration**
   - Use multiple NTP servers for redundancy
   - Prefer geographically closer NTP servers
   - Implement proper stratum hierarchy

2. **Monitoring**
   - Set appropriate alert thresholds
   - Monitor all nodes consistently
   - Track long-term drift patterns

3. **Security**
   - Use authenticated NTP where possible
   - Restrict NTP traffic to known servers
   - Regular security audits

## Reference Architecture

```plaintext
┌─────────────────┐    ┌──────────────┐    ┌────────────────┐
│   EKS Nodes     │    │  NTP Servers │    │   New Relic    │
│  ┌───────────┐  │    │              │    │                │
│  │ chronyd   │◄─┼────┤ pool.ntp.org │    │  ┌──────────┐ │
│  └───────────┘  │    │              │    │  │ Alerts   │ │
│  ┌───────────┐  │    └──────────────┘    │  └──────────┘ │
│  │ NR Agent  │◄─┼───────────────────────►│  ┌──────────┐ │
│  └───────────┘  │                        │  │Dashboard │ │
└─────────────────┘                        │  └──────────┘ │
                                          └────────────────┘
```

## Additional Resources

- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [Chrony Documentation](https://chrony.tuxfamily.org/documentation.html)
- [New Relic Flex Integration Guide](https://docs.newrelic.com/docs/infrastructure/host-integrations/host-integrations-list/flex-integration-tool-build-your-own-integration)
