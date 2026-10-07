# CloudWatch

- CloudWatch collects and stores operational data. It has three parts: **Metrics**, **Logs** and **Events**.
- **Alarms** belong to Metrics. They watch a metric and act when it crosses a threshold. Events have their own mechanism, **rules**.
- CloudWatch is a **public service**: it has public endpoints, so anything with internet access and AWS credentials can send data to it, including on-premises servers and other clouds.
- Most AWS services send metrics on their own. For data from inside an OS (memory, disk usage, app log files), or from outside AWS, install the **CloudWatch agent**.
- CloudWatch is **regional**. Metrics and logs live in the region you send them to.

```
                    ┌───────────────────── CloudWatch ────────────────────┐
AWS services ──────►│ Metrics ──► Alarms ──► SNS / Auto Scaling / EC2     │
                    │                                                     │
CloudWatch agent ──►│ Logs ──► metric filter ──► Metrics                  │
(EC2, on-prem)      │                                                     │
AWS events ────────►│ Events ──► rules ──► targets (Lambda, SNS, SQS...)  │
                    └─────────────────────────────────────────────────────┘
```

---

## Metrics

A **metric** is a time-ordered set of **datapoints**: one measured value over time, e.g. CPU usage of an EC2 instance.

A metric is identified by:

- **Namespace**: a container that groups metrics. `AWS/EC2`, `AWS/S3` and every other `AWS/` name is reserved for AWS services. Custom metrics use any other name, e.g. `MyApp`.
- **Metric name**: what's measured, e.g. `CPUUtilization`.
- **Dimensions**: name/value pairs that split one metric by source, e.g. `InstanceId=i-0abc...` or `AutoScalingGroupName=web`. Up to 30 per metric.

A **datapoint** is one measurement: a timestamp, a value and an optional unit.

```
Namespace: AWS/EC2
Metric:    CPUUtilization
Dimension: InstanceId=i-0abc123

  timestamp              value
  2026-10-06T10:00:00Z   12.5   ← datapoint
  2026-10-06T10:01:00Z   18.0   ← datapoint
  2026-10-06T10:02:00Z   71.3   ← datapoint
```

Without a dimension you see the metric for all instances combined. With `InstanceId` you see one instance.

### What EC2 sends by default

EC2 metrics come from the hypervisor, so AWS sees only what's visible from outside the VM: CPU, network in/out, disk I/O, status checks. **Memory and disk space usage aren't included.** For those, run the CloudWatch agent inside the instance. It needs an IAM role with permission to write to CloudWatch.

### Resolution and retention

- **Standard resolution**: one datapoint per minute. EC2 basic monitoring sends every 5 minutes. Detailed monitoring (paid) sends every minute.
- **High resolution**: down to one second, for custom metrics only.
- CloudWatch keeps datapoints for 15 months and rolls them up as they age:

| Granularity | Kept for |
|---|---|
| < 60 s (high resolution) | 3 hours |
| 60 s | 15 days |
| 5 min | 63 days |
| 1 hour | 455 days (15 months) |

- A **statistic** (Average, Sum, Minimum, Maximum, SampleCount, percentiles like p99) aggregates datapoints over a **period**, e.g. "average CPU per 5 minutes".

### Alarms

An **alarm** watches one metric (or a math expression over metrics) and compares a statistic to a threshold over a number of periods.

An alarm is always in one of three states:

- `OK`: the metric is within the threshold.
- `ALARM`: the metric crossed the threshold.
- `INSUFFICIENT_DATA`: not enough datapoints yet, e.g. right after creating the alarm.

On a state change the alarm can:

- notify an **SNS topic** (email, SMS, Lambda...);
- trigger an **Auto Scaling** policy;
- stop, terminate, reboot or recover an **EC2 instance**.

Example: `CPUUtilization` average > 80% for 3 periods of 5 minutes → `ALARM` → SNS email.

---

## Logs

CloudWatch Logs stores and searches log data. AWS services (Lambda, API Gateway, VPC Flow Logs, CloudTrail, Route 53) write to it directly. EC2 and on-premises servers need the agent.

```
Log group  /aws/lambda/resize-image          ← settings live here
├── Log stream  2026/10/06/[$LATEST]a1b2...   ← one source (instance, container, Lambda run)
│   ├── log event  10:00:01  START RequestId ...
│   ├── log event  10:00:02  Resized 3 images
│   └── log event  10:00:02  END RequestId ...
└── Log stream  2026/10/06/[$LATEST]c3d4...
```

- **Log event**: one entry, a timestamp and a message.
- **Log stream**: the ordered events from one source.
- **Log group**: a set of streams for the same kind of thing, e.g. all instances of one app. Retention, permissions, encryption and metric filters are set on the group.
- **Retention** defaults to "never expire". Set it per group, from 1 day to 10 years, or storage cost grows forever.
- A **metric filter** matches a pattern in a group's events and turns matches into a metric, e.g. count `ERROR` lines or failed SSH logins. You can then put an alarm on that metric.
- **Logs Insights** runs queries over log groups.
- **Subscription filters** stream events in near real time to Lambda, Kinesis or Firehose, e.g. to ship logs to S3 or OpenSearch.

---

## Events (now EventBridge)

CloudWatch Events delivers events about changes in AWS and runs actions in response. It is now **Amazon EventBridge**. EventBridge uses the same API and adds more features, so new exam questions usually say EventBridge.

- An **event** is a JSON document describing something that happened, e.g. "EC2 instance `i-0abc` changed state to `stopped`".
- A **rule** matches events and sends them to **targets** (Lambda, SNS, SQS, Step Functions, another event bus...). A rule is one of two kinds:
  - **Event pattern**: matches events, e.g. any EC2 instance entering `stopped`.
  - **Schedule**: fires on a cron or rate expression, e.g. every day at 02:00. This is the AWS-native cron.
- An **event bus** receives events. Every account has a **default bus** that AWS services publish to. EventBridge adds custom buses and events from SaaS partners.

```
EC2 instance stopped ──► default event bus ──► rule (pattern match) ──► Lambda
cron(0 2 * * ? *)    ─────────────────────────► rule (schedule)     ──► Lambda
```

### Alarms or events?

| | Alarm | Event rule |
|---|---|---|
| Reacts to | a metric crossing a threshold | something that happened, or a schedule |
| Example | CPU > 80% for 15 min | instance stopped, object uploaded, every night |
| Lives in | CloudWatch Metrics | CloudWatch Events / EventBridge |
