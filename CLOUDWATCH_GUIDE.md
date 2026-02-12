# CloudWatch Monitoring and Alarms Guide

This project includes comprehensive CloudWatch integration for monitoring your Nginx server.

## Overview

After deployment, your infrastructure automatically sends logs and metrics to AWS CloudWatch, with alarms configured to notify you of issues.

## CloudWatch Components

### 1. Log Group: `/aws/ec2/nginx`

The CloudWatch Log Group stores logs from your Nginx server:

**Log Streams:**
- `nginx-access-log` - All HTTP requests (GET, POST, etc.)
- `nginx-error-log` - Nginx errors and warnings

**Log Retention:** 7 days (configurable in code)

**Accessing Logs:**
```bash
# View recent logs from command line
aws logs tail /aws/ec2/nginx --follow

# View error logs only
aws logs tail /aws/ec2/nginx:nginx-error-log --follow

# View access logs only
aws logs tail /aws/ec2/nginx:nginx-access-log --follow
```

### 2. CloudWatch Metrics

The CloudWatch Agent collects and sends these metrics every 60 seconds:

**EC2 Instance Metrics:**
- CPU Utilization (%)
- CPU Idle Time (%)
- CPU I/O Wait (%)
- Network In (bytes)
- Network Out (bytes)
- Disk Read (bytes)
- Disk Write (bytes)

**Custom Metrics (from CloudWatch Agent):**
- `CPU_IDLE` - CPU idle percentage
- `CPU_IOWAIT` - I/O wait percentage
- `MEM_USED` - Memory usage percentage
- `DISK_USED` - Disk usage percentage

### 3. CloudWatch Alarms

The stack creates 4 alarms that send email notifications via SNS:

#### Alarm 1: CPU Utilization
- **Metric:** Instance CPU Utilization
- **Threshold:** > 80%
- **Evaluation:** 2 consecutive periods (2 minutes)
- **Action:** Send email notification

#### Alarm 2: Instance Status Check
- **Metric:** Instance Status Check Failed
- **Threshold:** 1 or more failures
- **Evaluation:** 2 consecutive periods
- **Action:** Send email notification

#### Alarm 3: Network Traffic
- **Metric:** Network In
- **Threshold:** > 1 GB per minute
- **Evaluation:** 1 period
- **Action:** Send email notification

#### Alarm 4: Error Log Activity
- **Metric:** CloudWatch Logs Incoming Events
- **Threshold:** > 100 events in 5 minutes
- **Evaluation:** 1 period
- **Action:** Send email notification

### 4. CloudWatch Dashboard

A pre-configured dashboard displays:
- Real-time CPU utilization graph
- Network traffic (in/out) graph
- Disk read/write operations
- Instance status check status

**Access Dashboard:**
Check the deployment output for `DashboardUrl` or navigate to:
```
https://console.aws.amazon.com/cloudwatch/home#dashboards:name=nginx-monitoring
```

## Configuration

### Email Notifications

By default, alarms notify `your-email@example.com`. To change this:

```bash
# Set your email before deployment
$env:ALARM_EMAIL = "your-real-email@example.com"  # PowerShell
npm run cdk:deploy
```

After deployment, check your email for SNS subscription confirmation.

### Modifying Alarm Thresholds

Edit `lib/aws-nginx-stack.ts` to adjust thresholds:

```typescript
// CPU Alarm - Change threshold from 80 to 70
const cpuAlarm = new cloudwatch.Alarm(this, 'NginxCpuAlarm', {
  metric: instance.metricCPUUtilization(),
  threshold: 70,  // Change this value
  // ...
});
```

After editing, redeploy:
```bash
npm run build
npm run cdk:deploy
```

### Custom Metrics

Add custom metrics by editing the CloudWatch Agent config in the user data script:

```bash
# In aws-nginx-stack.ts, modify the JSON config section
'      \"disk\": {',
'        \"measurement\": [',
'          {\"name\": \"used_percent\", \"rename\": \"DISK_USED\", \"unit\": \"Percent\"}',
'        ],',
```

## Accessing CloudWatch Data

### AWS Console
1. Go to CloudWatch Dashboard: Check deployment output for `DashboardUrl`
2. View Logs: CloudWatch Logs → Log Groups → `/aws/ec2/nginx`
3. View Alarms: CloudWatch → Alarms → View all alarms

### AWS CLI

**List all alarms:**
```bash
aws cloudwatch describe-alarms
```

**Get specific alarm state:**
```bash
aws cloudwatch describe-alarms --alarm-names AwsNginxStack-NginxCpuAlarm
```

**Get recent log events:**
```bash
aws logs get-log-events \
  --log-group-name /aws/ec2/nginx \
  --log-stream-name nginx-access-log \
  --limit 10
```

**Filter logs for errors:**
```bash
aws logs filter-log-events \
  --log-group-name /aws/ec2/nginx \
  --log-stream-name nginx-error-log \
  --filter-pattern "error"
```

### CloudWatch Insights

Run complex queries on your logs using CloudWatch Insights:

**Example: Count requests by status code**
```sql
fields @timestamp, status
| stats count() as requests by status
```

**Example: Find slow requests**
```sql
fields @duration
| filter @duration > 1000
| stats avg(@duration), max(@duration), pct(@duration, 95)
```

**Example: Error rate**
```sql
fields @timestamp, status
| stats count(status = 400 or status = 500) as error_count, count() as total_count
| fields error_count / total_count * 100 as error_rate_percent
```

## Troubleshooting

### Alarms Not Triggering

1. **Check CloudWatch Agent Status:**
   ```bash
   # SSH into instance and check
   sudo systemctl status amazon-cloudwatch-agent
   ```

2. **Verify Log Group Exists:**
   ```bash
   aws logs describe-log-groups --log-group-name-prefix /aws/ec2/nginx
   ```

3. **Check SNS Subscription:**
   - Go to SNS → Topics → nginx-alarms
   - Verify subscription status is "Confirmed"

### No Logs Appearing

1. **Check CloudWatch Agent Config:**
   ```bash
   sudo cat /opt/aws/amazon-cloudwatch-agent/etc/config.json
   ```

2. **Restart CloudWatch Agent:**
   ```bash
   sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
     -a fetch-config -m ec2 -s \
     -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
   ```

3. **Check IAM Permissions:**
   Ensure the instance IAM role has `CloudWatchAgentServerPolicy`

### High Costs

To reduce CloudWatch costs:

1. **Increase Log Retention:**
   In `aws-nginx-stack.ts`, change:
   ```typescript
   retention: logs.RetentionDays.THREE_DAYS,  // Shorter retention
   ```

2. **Reduce Metric Collection Interval:**
   In user data, change:
   ```json
   "metrics_collection_interval": 300,  // 5 minutes instead of 60 seconds
   ```

3. **Remove Unused Metrics:**
   Comment out metrics in the CloudWatch Agent config

## Cost Breakdown

- **CloudWatch Logs Ingestion:** $0.50 per GB (5 GB free/month)
- **Log Storage:** Minimal with 7-day retention
- **Metrics:** $0.30 per custom metric per month
- **Alarms:** $0.10 per alarm per month
- **SNS Emails:** Free (1,000/month included)

**Typical monthly cost:** $0-2 with normal usage

## Next Steps

1. **Set up SNS Subscriptions:** Add additional recipients for alarms
2. **Create Custom Metrics:** Monitor application-specific metrics
3. **Set up Log Filters:** Create metric alarms based on log patterns
4. **Enable CloudWatch Synthetics:** Monitor endpoint availability
5. **Integrate with other AWS Services:** Send logs to S3, Elasticsearch, etc.

## Resources

- [CloudWatch Documentation](https://docs.aws.amazon.com/cloudwatch/)
- [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)
- [CloudWatch Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [CloudWatch Agent Configuration](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
