# Monitoring & Observability Guide - 10DLC SMS System

## Table of Contents
1. [Overview](#overview)
2. [Monitoring Architecture](#monitoring-architecture)
3. [Metrics Collection](#metrics-collection)
4. [Logging Strategy](#logging-strategy)
5. [Alerting Framework](#alerting-framework)
6. [Performance Monitoring](#performance-monitoring)
7. [Business Intelligence](#business-intelligence)
8. [Incident Management](#incident-management)
9. [Dashboards & Visualization](#dashboards--visualization)
10. [Future Monitoring Enhancements](#future-monitoring-enhancements)

## Overview

This document provides comprehensive guidance for implementing monitoring and observability for the 10DLC SMS system, enabling proactive issue detection, performance optimization, and business intelligence.

### Monitoring Objectives
- **Proactive Issue Detection**: Identify problems before they impact users
- **Performance Optimization**: Continuous performance improvement insights
- **Business Intelligence**: Data-driven decision making capabilities
- **Compliance Monitoring**: Regulatory adherence tracking
- **Cost Optimization**: Resource usage and cost tracking
- **Security Monitoring**: Threat detection and security posture

### Observability Principles
- **Three Pillars**: Metrics, Logs, and Traces
- **High Cardinality**: Detailed dimensional data
- **Real-time Processing**: Immediate insights and alerting
- **Historical Analysis**: Trend analysis and capacity planning
- **Actionable Insights**: Monitoring that drives action
- **Self-Service**: Empower teams with monitoring tools

## Monitoring Architecture

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                    Data Sources                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │Application  │  │Infrastructure│  │  Business   │        │
│  │   Metrics   │  │   Metrics    │  │   Metrics   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                 Collection Layer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Prometheus  │  │    Fluentd   │  │    Jaeger   │        │
│  │  (Metrics)  │  │    (Logs)    │  │   (Traces)  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                 Processing Layer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Alertmanager│  │ Elasticsearch│  │   Grafana   │        │
│  │ (Alerting)  │  │ (Log Search) │  │(Visualization)│      │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│               Presentation Layer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Dashboards │  │    Alerts   │  │   Reports   │        │
│  │   (Grafana) │  │(Slack/Email)│  │(Scheduled)  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack
- **Metrics**: Prometheus + Grafana
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Tracing**: Jaeger for distributed tracing
- **Alerting**: Alertmanager + PagerDuty
- **Business Intelligence**: Custom dashboards + DataDog
- **APM**: New Relic or Datadog APM

## Metrics Collection

### Application Metrics Implementation
```typescript
// src/monitoring/metrics.service.ts
import { register, Counter, Histogram, Gauge, Summary } from 'prom-client';
import { Request, Response, NextFunction } from 'express';

export class MetricsService {
  // HTTP Metrics
  private httpRequestsTotal = new Counter({
    name: 'http_requests_total',
    help: 'Total number of HTTP requests',
    labelNames: ['method', 'route', 'status_code', 'user_id']
  });

  private httpRequestDuration = new Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests in seconds',
    labelNames: ['method', 'route'],
    buckets: [0.1, 0.5, 1, 2, 5, 10, 30]
  });

  // Business Metrics
  private messagesProcessed = new Counter({
    name: 'messages_processed_total',
    help: 'Total number of messages processed',
    labelNames: ['direction', 'status', 'campaign_id', 'use_case']
  });

  private messageProcessingDuration = new Histogram({
    name: 'message_processing_duration_seconds',
    help: 'Time taken to process messages',
    labelNames: ['direction', 'status'],
    buckets: [0.1, 0.5, 1, 2, 5]
  });

  private activeCampaigns = new Gauge({
    name: 'active_campaigns_total',
    help: 'Number of active campaigns',
    labelNames: ['status', 'use_case']
  });

  private messageQueueLength = new Gauge({
    name: 'message_queue_length',
    help: 'Current length of message processing queue',
    labelNames: ['queue_name', 'priority']
  });

  // Database Metrics
  private databaseConnections = new Gauge({
    name: 'database_connections_active',
    help: 'Number of active database connections',
    collect() {
      // This will be called every time the metric is scraped
      this.set(getDatabaseConnectionCount());
    }
  });

  private databaseQueryDuration = new Histogram({
    name: 'database_query_duration_seconds',
    help: 'Duration of database queries',
    labelNames: ['operation', 'table'],
    buckets: [0.001, 0.01, 0.1, 0.5, 1, 2]
  });

  // External API Metrics
  private externalApiCalls = new Counter({
    name: 'external_api_calls_total',
    help: 'Total external API calls',
    labelNames: ['provider', 'endpoint', 'status_code']
  });

  private externalApiDuration = new Histogram({
    name: 'external_api_duration_seconds',
    help: 'Duration of external API calls',
    labelNames: ['provider', 'endpoint'],
    buckets: [0.1, 0.5, 1, 2, 5, 10, 30]
  });

  // Cost Metrics
  private messageCost = new Counter({
    name: 'message_cost_total',
    help: 'Total cost of messages sent',
    labelNames: ['provider', 'message_type', 'campaign_id']
  });

  private monthlySpend = new Gauge({
    name: 'monthly_spend_dollars',
    help: 'Current monthly spend in dollars',
    labelNames: ['category', 'provider']
  });

  constructor() {
    this.registerMetrics();
  }

  private registerMetrics(): void {
    register.registerMetric(this.httpRequestsTotal);
    register.registerMetric(this.httpRequestDuration);
    register.registerMetric(this.messagesProcessed);
    register.registerMetric(this.messageProcessingDuration);
    register.registerMetric(this.activeCampaigns);
    register.registerMetric(this.messageQueueLength);
    register.registerMetric(this.databaseConnections);
    register.registerMetric(this.databaseQueryDuration);
    register.registerMetric(this.externalApiCalls);
    register.registerMetric(this.externalApiDuration);
    register.registerMetric(this.messageCost);
    register.registerMetric(this.monthlySpend);
  }

  // Middleware for HTTP metrics
  httpMetricsMiddleware() {
    return (req: Request, res: Response, next: NextFunction) => {
      const startTime = Date.now();

      res.on('finish', () => {
        const duration = (Date.now() - startTime) / 1000;
        const route = req.route?.path || req.path;
        const userId = req.user?.id || 'anonymous';

        this.httpRequestsTotal.inc({
          method: req.method,
          route,
          status_code: res.statusCode,
          user_id: userId
        });

        this.httpRequestDuration.observe(
          { method: req.method, route },
          duration
        );
      });

      next();
    };
  }

  // Business metrics methods
  recordMessageProcessed(
    direction: string,
    status: string,
    campaignId: string,
    useCase: string,
    duration?: number
  ): void {
    this.messagesProcessed.inc({
      direction,
      status,
      campaign_id: campaignId,
      use_case: useCase
    });

    if (duration) {
      this.messageProcessingDuration.observe(
        { direction, status },
        duration
      );
    }
  }

  recordExternalApiCall(
    provider: string,
    endpoint: string,
    statusCode: number,
    duration: number
  ): void {
    this.externalApiCalls.inc({
      provider,
      endpoint,
      status_code: statusCode
    });

    this.externalApiDuration.observe(
      { provider, endpoint },
      duration
    );
  }

  recordMessageCost(
    provider: string,
    messageType: string,
    campaignId: string,
    cost: number
  ): void {
    this.messageCost.inc(
      { provider, message_type: messageType, campaign_id: campaignId },
      cost
    );
  }

  updateQueueLength(queueName: string, priority: string, length: number): void {
    this.messageQueueLength.set({ queue_name: queueName, priority }, length);
  }

  updateActiveCampaigns(status: string, useCase: string, count: number): void {
    this.activeCampaigns.set({ status, use_case: useCase }, count);
  }

  getMetrics(): string {
    return register.metrics();
  }
}

// Database query instrumentation
export function instrumentDatabaseQueries() {
  const originalQuery = Connection.prototype.query;
  
  Connection.prototype.query = function(sql: string, params?: any[]) {
    const startTime = Date.now();
    const operation = sql.split(' ')[0].toUpperCase();
    const table = extractTableName(sql);
    
    const result = originalQuery.call(this, sql, params);
    
    if (result instanceof Promise) {
      return result.finally(() => {
        const duration = (Date.now() - startTime) / 1000;
        metricsService.databaseQueryDuration.observe(
          { operation, table },
          duration
        );
      });
    }
    
    return result;
  };
}
```

### Infrastructure Metrics Configuration
```yaml
# k8s/monitoring/prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
      - "/etc/prometheus/rules/*.yml"
    
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              - alertmanager:9093
    
    scrape_configs:
      # Application metrics
      - job_name: 'sms-api'
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - production
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: __tmp_app
          - source_labels: [__tmp_app]
            regex: sms-api
            action: keep
          - source_labels: [__meta_kubernetes_pod_container_port_name]
            regex: metrics
            action: keep
    
      # Node metrics
      - job_name: 'node-exporter'
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - source_labels: [__address__]
            regex: '(.*):10250'
            target_label: __address__
            replacement: '${1}:9100'
    
      # Kubernetes metrics
      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics:8080']
    
      # Database metrics
      - job_name: 'mysql-exporter'
        static_configs:
          - targets: ['mysql-exporter:9104']
    
      # Redis metrics
      - job_name: 'redis-exporter'
        static_configs:
          - targets: ['redis-exporter:9121']
```

### Custom Business Metrics
```typescript
// src/monitoring/business-metrics.service.ts
export class BusinessMetricsService {
  private campaignPerformanceGauge = new Gauge({
    name: 'campaign_performance_score',
    help: 'Campaign performance score (0-100)',
    labelNames: ['campaign_id', 'brand_id', 'use_case']
  });

  private complianceScore = new Gauge({
    name: 'compliance_score',
    help: 'Compliance score for campaigns (0-100)',
    labelNames: ['campaign_id', 'regulation_type']
  });

  private revenueMetrics = new Counter({
    name: 'revenue_generated_total',
    help: 'Total revenue generated',
    labelNames: ['customer_id', 'plan_type', 'region']
  });

  async calculateCampaignMetrics(): Promise<void> {
    const campaigns = await this.campaignService.getActiveCampaigns();
    
    for (const campaign of campaigns) {
      const performance = await this.calculatePerformanceScore(campaign);
      const compliance = await this.calculateComplianceScore(campaign);
      
      this.campaignPerformanceGauge.set(
        {
          campaign_id: campaign.id,
          brand_id: campaign.brandId,
          use_case: campaign.useCase
        },
        performance
      );
      
      this.complianceScore.set(
        {
          campaign_id: campaign.id,
          regulation_type: 'tcpa'
        },
        compliance.tcpa
      );
      
      this.complianceScore.set(
        {
          campaign_id: campaign.id,
          regulation_type: '10dlc'
        },
        compliance.tenDlc
      );
    }
  }

  private async calculatePerformanceScore(campaign: Campaign): Promise<number> {
    const stats = await this.messageService.getCampaignStats(campaign.id);
    
    const deliveryRate = stats.delivered / stats.sent;
    const optOutRate = stats.optOuts / stats.sent;
    const errorRate = stats.failed / stats.sent;
    
    // Weighted performance score
    const score = (
      (deliveryRate * 0.5) +
      ((1 - optOutRate) * 0.3) +
      ((1 - errorRate) * 0.2)
    ) * 100;
    
    return Math.max(0, Math.min(100, score));
  }

  private async calculateComplianceScore(campaign: Campaign): Promise<{tcpa: number, tenDlc: number}> {
    const violations = await this.complianceService.getViolations(campaign.id);
    
    const tcpaScore = Math.max(0, 100 - (violations.tcpa.length * 10));
    const tenDlcScore = Math.max(0, 100 - (violations.tenDlc.length * 15));
    
    return { tcpa: tcpaScore, tenDlc: tenDlcScore };
  }
}
```

## Logging Strategy

### Structured Logging Implementation
```typescript
// src/utils/logger.util.ts
import winston from 'winston';
import { ElasticsearchTransport } from 'winston-elasticsearch';

interface LogContext {
  userId?: string;
  requestId?: string;
  campaignId?: string;
  messageId?: string;
  [key: string]: any;
}

class StructuredLogger {
  private logger: winston.Logger;

  constructor() {
    const logFormat = winston.format.combine(
      winston.format.timestamp(),
      winston.format.errors({ stack: true }),
      winston.format.json(),
      winston.format.printf(({ timestamp, level, message, ...meta }) => {
        return JSON.stringify({
          timestamp,
          level,
          message,
          ...meta,
          service: 'sms-api',
          version: process.env.APP_VERSION || '1.0.0',
          environment: process.env.NODE_ENV || 'development'
        });
      })
    );

    this.logger = winston.createLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: logFormat,
      defaultMeta: {
        service: 'sms-api'
      },
      transports: [
        new winston.transports.Console(),
        new winston.transports.File({
          filename: 'logs/error.log',
          level: 'error'
        }),
        new winston.transports.File({
          filename: 'logs/combined.log'
        })
      ]
    });

    // Add Elasticsearch transport for production
    if (process.env.NODE_ENV === 'production') {
      this.logger.add(new ElasticsearchTransport({
        level: 'info',
        clientOpts: {
          node: process.env.ELASTICSEARCH_URL,
          auth: {
            username: process.env.ELASTICSEARCH_USERNAME,
            password: process.env.ELASTICSEARCH_PASSWORD
          }
        },
        index: 'sms-api-logs'
      }));
    }
  }

  info(message: string, context?: LogContext): void {
    this.logger.info(message, context);
  }

  warn(message: string, context?: LogContext): void {
    this.logger.warn(message, context);
  }

  error(message: string, error?: Error, context?: LogContext): void {
    this.logger.error(message, {
      error: error ? {
        name: error.name,
        message: error.message,
        stack: error.stack
      } : undefined,
      ...context
    });
  }

  debug(message: string, context?: LogContext): void {
    this.logger.debug(message, context);
  }

  // Business event logging
  logMessageEvent(event: string, messageId: string, context?: LogContext): void {
    this.info(`Message ${event}`, {
      event_type: 'message_lifecycle',
      message_id: messageId,
      ...context
    });
  }

  logComplianceEvent(event: string, campaignId: string, context?: LogContext): void {
    this.info(`Compliance ${event}`, {
      event_type: 'compliance',
      campaign_id: campaignId,
      ...context
    });
  }

  logSecurityEvent(event: string, userId?: string, context?: LogContext): void {
    this.warn(`Security ${event}`, {
      event_type: 'security',
      user_id: userId,
      ...context
    });
  }

  logPerformanceEvent(operation: string, duration: number, context?: LogContext): void {
    this.info(`Performance ${operation}`, {
      event_type: 'performance',
      duration_ms: duration,
      ...context
    });
  }
}

export const logger = new StructuredLogger();

// Request logging middleware
export function requestLoggingMiddleware() {
  return (req: Request, res: Response, next: NextFunction) => {
    const requestId = uuidv4();
    req.requestId = requestId;
    
    const startTime = Date.now();
    
    logger.info('Request started', {
      request_id: requestId,
      method: req.method,
      url: req.url,
      user_agent: req.get('user-agent'),
      ip: req.ip,
      user_id: req.user?.id
    });

    res.on('finish', () => {
      const duration = Date.now() - startTime;
      
      logger.info('Request completed', {
        request_id: requestId,
        method: req.method,
        url: req.url,
        status_code: res.statusCode,
        duration_ms: duration,
        user_id: req.user?.id
      });
    });

    next();
  };
}
```

### Log Aggregation Configuration
```yaml
# k8s/logging/fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      @id input_tail
      @label @mainstream
      path /var/log/containers/*sms-api*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag raw.kubernetes.*
      read_from_head true
      <parse>
        @type multi_format
        <pattern>
          format json
          time_key timestamp
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
        <pattern>
          format /^(?<timestamp>[^ ]*) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>

    <label @mainstream>
      <filter **>
        @type kubernetes_metadata
        @id filter_kube_metadata
        kubernetes_url "#{ENV['FLUENT_FILTER_KUBERNETES_URL'] || 'https://' + ENV['KUBERNETES_SERVICE_HOST'] + ':' + ENV['KUBERNETES_SERVICE_PORT'] + '/api'}"
        verify_ssl "#{ENV['KUBERNETES_VERIFY_SSL'] || true}"
        ca_file "#{ENV['KUBERNETES_CA_FILE']}"
        skip_labels false
        skip_container_metadata false
        skip_master_url false
        skip_namespace_metadata false
      </filter>

      <filter **>
        @type parser
        @id filter_parser
        key_name log
        reserve_data true
        remove_key_name_field true
        <parse>
          @type multi_format
          <pattern>
            format json
          </pattern>
          <pattern>
            format none
          </pattern>
        </parse>
      </filter>

      <match **>
        @type elasticsearch
        @id output_elasticsearch
        host "#{ENV['FLUENT_ELASTICSEARCH_HOST'] || 'elasticsearch'}"
        port "#{ENV['FLUENT_ELASTICSEARCH_PORT'] || '9200'}"
        scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME'] || 'http'}"
        user "#{ENV['FLUENT_ELASTICSEARCH_USER'] || use_default}"
        password "#{ENV['FLUENT_ELASTICSEARCH_PASSWORD'] || use_default}"
        index_name "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_INDEX_NAME'] || 'sms-api'}"
        type_name "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_TYPE_NAME'] || 'fluentd'}"
        include_timestamp true
        reload_connections false
        reconnect_on_error true
        reload_on_failure true
        log_es_400_reason false
        logstash_prefix "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_PREFIX'] || 'sms-api'}"
        logstash_format true
        buffer_chunk_limit 2M
        buffer_queue_limit 8
        flush_interval 5s
        max_retry_wait 30
        disable_retry_limit
        num_threads 2
      </match>
    </label>
```

## Alerting Framework

### Alert Rules Configuration
```yaml
# k8s/monitoring/alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: sms-api-alerts
spec:
  groups:
  - name: sms-api.rules
    rules:
    # High Error Rate
    - alert: HighErrorRate
      expr: rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
        service: sms-api
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"

    # High Response Time
    - alert: HighResponseTime
      expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
      for: 10m
      labels:
        severity: warning
        service: sms-api
      annotations:
        summary: "High response time detected"
        description: "95th percentile response time is {{ $value }}s"

    # Message Processing Backlog
    - alert: MessageProcessingBacklog
      expr: message_queue_length > 1000
      for: 5m
      labels:
        severity: critical
        service: sms-api
      annotations:
        summary: "Message processing backlog detected"
        description: "Message queue length is {{ $value }}"

    # Database Connection Issues
    - alert: DatabaseConnectionHigh
      expr: database_connections_active > 18
      for: 5m
      labels:
        severity: warning
        service: sms-api
      annotations:
        summary: "High database connection usage"
        description: "Active database connections: {{ $value }}"

    # External API Failures
    - alert: TelnyxAPIFailures
      expr: rate(external_api_calls_total{provider="telnyx",status_code=~"5.."}[5m]) > 0.1
      for: 3m
      labels:
        severity: critical
        service: sms-api
      annotations:
        summary: "Telnyx API failures detected"
        description: "Telnyx API failure rate is {{ $value | humanizePercentage }}"

    # Compliance Violations
    - alert: ComplianceViolations
      expr: compliance_score < 80
      for: 1m
      labels:
        severity: critical
        service: sms-api
      annotations:
        summary: "Compliance score below threshold"
        description: "Compliance score for campaign {{ $labels.campaign_id }} is {{ $value }}"

    # Cost Anomalies
    - alert: UnexpectedCostIncrease
      expr: increase(message_cost_total[1h]) > 100
      for: 15m
      labels:
        severity: warning
        service: sms-api
      annotations:
        summary: "Unexpected cost increase detected"
        description: "Message costs increased by ${{ $value }} in the last hour"

    # Infrastructure Alerts
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod is crash looping"
        description: "Pod {{ $labels.pod }} is crash looping"

    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="false"} == 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod not ready"
        description: "Pod {{ $labels.pod }} is not ready"

    # Business Metrics
    - alert: LowCampaignPerformance
      expr: campaign_performance_score < 70
      for: 30m
      labels:
        severity: warning
        service: sms-api
      annotations:
        summary: "Low campaign performance detected"
        description: "Campaign {{ $labels.campaign_id }} performance score is {{ $value }}"

    - alert: HighOptOutRate
      expr: rate(messages_processed_total{status="opted_out"}[1h]) / rate(messages_processed_total{direction="outbound"}[1h]) > 0.02
      for: 15m
      labels:
        severity: warning
        service: sms-api
      annotations:
        summary: "High opt-out rate detected"
        description: "Opt-out rate is {{ $value | humanizePercentage }} in the last hour"
```

### Alertmanager Configuration
```yaml
# k8s/monitoring/alertmanager-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
data:
  alertmanager.yml: |
    global:
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
      pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

    route:
      group_by: ['alertname', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'web.hook'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty-critical'
        continue: true
      - match:
          severity: warning
        receiver: 'slack-warnings'

    receivers:
    - name: 'web.hook'
      webhook_configs:
      - url: 'http://alertmanager-webhook:8080/webhook'

    - name: 'slack-warnings'
      slack_configs:
      - channel: '#alerts-warnings'
        title: 'SMS API Warning Alert'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}{{ end }}'
        send_resolved: true

    - name: 'pagerduty-critical'
      pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_ROUTING_KEY'
        description: '{{ .GroupLabels.alertname }}: {{ .GroupLabels.service }}'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'

    - name: 'email-notifications'
      email_configs:
      - to: 'oncall@yourcompany.com'
        from: 'alerts@yourcompany.com'
        subject: 'SMS API Alert: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Labels: {{ range .Labels.SortedPairs }}{{ .Name }}={{ .Value }} {{ end }}
          {{ end }}

    inhibit_rules:
    - source_match:
        severity: 'critical'
      target_match:
        severity: 'warning'
      equal: ['alertname', 'service']
```

## Performance Monitoring

### Application Performance Monitoring
```typescript
// src/monitoring/apm.service.ts
import * as newrelic from 'newrelic';
import { performance } from 'perf_hooks';

export class APMService {
  
  // Custom transaction monitoring
  async monitorOperation<T>(
    name: string,
    operation: () => Promise<T>,
    attributes?: Record<string, any>
  ): Promise<T> {
    const startTime = performance.now();
    
    try {
      const result = await newrelic.startWebTransaction(name, async () => {
        if (attributes) {
          newrelic.addCustomAttributes(attributes);
        }
        return await operation();
      });
      
      const duration = performance.now() - startTime;
      
      // Record custom metrics
      newrelic.recordMetric('Custom/Operation/Duration', duration);
      newrelic.recordMetric(`Custom/Operation/${name}/Duration`, duration);
      
      return result;
    } catch (error) {
      newrelic.noticeError(error);
      const duration = performance.now() - startTime;
      newrelic.recordMetric(`Custom/Operation/${name}/Error`, duration);
      throw error;
    }
  }

  // Database operation monitoring
  monitorDatabaseOperation<T>(
    operation: string,
    table: string,
    query: () => Promise<T>
  ): Promise<T> {
    return this.monitorOperation(
      `Database/${operation}/${table}`,
      query,
      { operation, table }
    );
  }

  // External API monitoring
  monitorExternalAPI<T>(
    provider: string,
    endpoint: string,
    apiCall: () => Promise<T>
  ): Promise<T> {
    return this.monitorOperation(
      `External/${provider}/${endpoint}`,
      apiCall,
      { provider, endpoint }
    );
  }

  // Message processing monitoring
  async monitorMessageProcessing<T>(
    messageId: string,
    campaignId: string,
    operation: () => Promise<T>
  ): Promise<T> {
    return this.monitorOperation(
      'MessageProcessing',
      operation,
      { 
        messageId, 
        campaignId,
        operation_type: 'message_processing'
      }
    );
  }

  // Custom business metrics
  recordBusinessMetric(name: string, value: number, attributes?: Record<string, any>): void {
    newrelic.recordMetric(`Custom/Business/${name}`, value);
    if (attributes) {
      newrelic.addCustomAttributes(attributes);
    }
  }

  // Error tracking with context
  recordError(error: Error, context?: Record<string, any>): void {
    if (context) {
      newrelic.addCustomAttributes(context);
    }
    newrelic.noticeError(error);
  }

  // Performance profiling
  startProfile(name: string): string {
    const profileId = `${name}_${Date.now()}`;
    newrelic.startSegment(profileId, false, () => {
      // Segment will be ended when promise resolves
    });
    return profileId;
  }

  endProfile(profileId: string): void {
    // Profile automatically ends when segment completes
    newrelic.recordMetric(`Custom/Profile/${profileId}/Completed`, 1);
  }
}

export const apmService = new APMService();

// Decorator for automatic monitoring
export function Monitor(name?: string) {
  return function(target: any, propertyName: string, descriptor: PropertyDescriptor) {
    const method = descriptor.value;
    const operationName = name || `${target.constructor.name}.${propertyName}`;

    descriptor.value = async function(...args: any[]) {
      return apmService.monitorOperation(
        operationName,
        () => method.apply(this, args),
        { class: target.constructor.name, method: propertyName }
      );
    };
  };
}
```

### Performance Benchmarking
```typescript
// src/monitoring/performance-benchmark.service.ts
export class PerformanceBenchmarkService {
  private benchmarks: Map<string, BenchmarkResult[]> = new Map();

  async runBenchmark(
    name: string,
    operation: () => Promise<void>,
    iterations: number = 100
  ): Promise<BenchmarkSummary> {
    const results: BenchmarkResult[] = [];

    for (let i = 0; i < iterations; i++) {
      const startTime = performance.now();
      const startMemory = process.memoryUsage();

      try {
        await operation();
        
        const endTime = performance.now();
        const endMemory = process.memoryUsage();
        
        results.push({
          iteration: i,
          duration: endTime - startTime,
          memoryDelta: endMemory.heapUsed - startMemory.heapUsed,
          success: true
        });
      } catch (error) {
        results.push({
          iteration: i,
          duration: performance.now() - startTime,
          memoryDelta: 0,
          success: false,
          error: error.message
        });
      }
    }

    this.benchmarks.set(name, results);
    return this.calculateSummary(results);
  }

  private calculateSummary(results: BenchmarkResult[]): BenchmarkSummary {
    const successful = results.filter(r => r.success);
    const durations = successful.map(r => r.duration);
    const memoryDeltas = successful.map(r => r.memoryDelta);

    return {
      totalIterations: results.length,
      successfulIterations: successful.length,
      successRate: successful.length / results.length,
      duration: {
        min: Math.min(...durations),
        max: Math.max(...durations),
        avg: durations.reduce((a, b) => a + b, 0) / durations.length,
        p95: this.percentile(durations, 95),
        p99: this.percentile(durations, 99)
      },
      memory: {
        avgDelta: memoryDeltas.reduce((a, b) => a + b, 0) / memoryDeltas.length,
        maxDelta: Math.max(...memoryDeltas)
      }
    };
  }

  private percentile(values: number[], p: number): number {
    const sorted = values.sort((a, b) => a - b);
    const index = Math.ceil((p / 100) * sorted.length) - 1;
    return sorted[index];
  }

  // Automated performance regression detection
  async detectPerformanceRegression(
    name: string,
    currentBenchmark: BenchmarkSummary,
    thresholds: PerformanceThresholds
  ): Promise<RegressionAnalysis> {
    const historicalData = await this.getHistoricalBenchmarks(name);
    
    if (historicalData.length < 5) {
      return { hasRegression: false, reason: 'Insufficient historical data' };
    }

    const avgHistoricalDuration = this.calculateHistoricalAverage(historicalData, 'duration.avg');
    const regressionThreshold = avgHistoricalDuration * (1 + thresholds.durationIncrease);

    if (currentBenchmark.duration.avg > regressionThreshold) {
      return {
        hasRegression: true,
        reason: 'Average duration increased significantly',
        currentValue: currentBenchmark.duration.avg,
        historicalAverage: avgHistoricalDuration,
        threshold: regressionThreshold,
        severity: this.calculateSeverity(currentBenchmark.duration.avg, regressionThreshold)
      };
    }

    return { hasRegression: false };
  }
}
```

## Business Intelligence

### KPI Dashboard Metrics
```typescript
// src/monitoring/kpi-metrics.service.ts
export class KPIMetricsService {
  
  // Revenue Metrics
  async calculateRevenueMetrics(period: DateRange): Promise<RevenueMetrics> {
    const messages = await this.messageService.getMessagesInPeriod(period);
    const costs = await this.billingService.getCostsInPeriod(period);
    const customers = await this.customerService.getActiveCustomers(period);

    return {
      totalRevenue: costs.totalRevenue,
      revenueGrowth: await this.calculateGrowthRate('revenue', period),
      averageRevenuePerUser: costs.totalRevenue / customers.length,
      messageVolume: messages.length,
      costPerMessage: costs.totalCost / messages.length,
      profitMargin: (costs.totalRevenue - costs.totalCost) / costs.totalRevenue,
      customerAcquisitionCost: await this.calculateCAC(period),
      lifetimeValue: await this.calculateLTV(period)
    };
  }

  // Operational Metrics
  async calculateOperationalMetrics(period: DateRange): Promise<OperationalMetrics> {
    const campaigns = await this.campaignService.getCampaignsInPeriod(period);
    const messages = await this.messageService.getMessagesInPeriod(period);
    
    const deliveryRate = messages.filter(m => m.status === 'delivered').length / messages.length;
    const errorRate = messages.filter(m => m.status === 'failed').length / messages.length;
    const optOutRate = await this.calculateOptOutRate(period);

    return {
      totalCampaigns: campaigns.length,
      activeCampaigns: campaigns.filter(c => c.status === 'approved').length,
      messageDeliveryRate: deliveryRate,
      messageErrorRate: errorRate,
      optOutRate: optOutRate,
      averageResponseTime: await this.calculateAverageResponseTime(period),
      systemUptime: await this.calculateUptime(period),
      complianceScore: await this.calculateComplianceScore(period)
    };
  }

  // Customer Metrics
  async calculateCustomerMetrics(period: DateRange): Promise<CustomerMetrics> {
    const users = await this.userService.getUsersInPeriod(period);
    const brands = await this.brandService.getBrandsInPeriod(period);
    
    return {
      totalUsers: users.length,
      activeUsers: users.filter(u => u.lastLogin > period.start).length,
      newUsers: users.filter(u => u.createdAt > period.start).length,
      churnRate: await this.calculateChurnRate(period),
      userEngagement: await this.calculateUserEngagement(period),
      brandRegistrations: brands.length,
      brandApprovals: brands.filter(b => b.status === 'approved').length,
      averageTimeToApproval: await this.calculateAverageApprovalTime(period)
    };
  }

  // Performance Metrics
  async calculatePerformanceMetrics(period: DateRange): Promise<PerformanceMetrics> {
    const apiMetrics = await this.getAPIMetrics(period);
    const infraMetrics = await this.getInfrastructureMetrics(period);

    return {
      averageResponseTime: apiMetrics.averageResponseTime,
      p95ResponseTime: apiMetrics.p95ResponseTime,
      errorRate: apiMetrics.errorRate,
      throughput: apiMetrics.requestsPerSecond,
      databaseQueryTime: infraMetrics.averageQueryTime,
      cacheHitRate: infraMetrics.cacheHitRate,
      cpuUtilization: infraMetrics.averageCPU,
      memoryUtilization: infraMetrics.averageMemory
    };
  }

  // Automated reporting
  async generateExecutiveDashboard(period: DateRange): Promise<ExecutiveDashboard> {
    const [revenue, operational, customer, performance] = await Promise.all([
      this.calculateRevenueMetrics(period),
      this.calculateOperationalMetrics(period),
      this.calculateCustomerMetrics(period),
      this.calculatePerformanceMetrics(period)
    ]);

    return {
      period,
      generatedAt: new Date(),
      revenue,
      operational,
      customer,
      performance,
      keyInsights: await this.generateKeyInsights({
        revenue,
        operational,
        customer,
        performance
      }),
      recommendations: await this.generateRecommendations({
        revenue,
        operational,
        customer,
        performance
      })
    };
  }

  private async generateKeyInsights(metrics: AllMetrics): Promise<string[]> {
    const insights: string[] = [];

    if (metrics.revenue.revenueGrowth > 0.2) {
      insights.push(`Strong revenue growth of ${(metrics.revenue.revenueGrowth * 100).toFixed(1)}% this period`);
    }

    if (metrics.operational.messageDeliveryRate < 0.95) {
      insights.push(`Message delivery rate below target at ${(metrics.operational.messageDeliveryRate * 100).toFixed(1)}%`);
    }

    if (metrics.customer.churnRate > 0.05) {
      insights.push(`Customer churn rate elevated at ${(metrics.customer.churnRate * 100).toFixed(1)}%`);
    }

    if (metrics.performance.p95ResponseTime > 2000) {
      insights.push(`API performance degraded with P95 response time at ${metrics.performance.p95ResponseTime}ms`);
    }

    return insights;
  }
}
```

### Real-time Analytics Dashboard
```typescript
// src/monitoring/realtime-analytics.service.ts
export class RealTimeAnalyticsService {
  private wsServer: WebSocket.Server;
  private subscribers: Map<string, WebSocket[]> = new Map();

  constructor() {
    this.wsServer = new WebSocket.Server({ port: 8080 });
    this.setupWebSocketHandlers();
    this.startRealTimeUpdates();
  }

  private setupWebSocketHandlers(): void {
    this.wsServer.on('connection', (ws: WebSocket) => {
      ws.on('message', (message: string) => {
        const { type, subscriptions } = JSON.parse(message);
        
        if (type === 'subscribe') {
          this.handleSubscription(ws, subscriptions);
        }
      });

      ws.on('close', () => {
        this.removeSubscriber(ws);
      });
    });
  }

  private handleSubscription(ws: WebSocket, subscriptions: string[]): void {
    for (const subscription of subscriptions) {
      if (!this.subscribers.has(subscription)) {
        this.subscribers.set(subscription, []);
      }
      this.subscribers.get(subscription)!.push(ws);
    }
  }

  private startRealTimeUpdates(): void {
    // Update every 5 seconds
    setInterval(async () => {
      await this.broadcastRealTimeMetrics();
    }, 5000);

    // Update business metrics every minute
    setInterval(async () => {
      await this.broadcastBusinessMetrics();
    }, 60000);
  }

  private async broadcastRealTimeMetrics(): Promise<void> {
    const metrics = {
      timestamp: new Date(),
      messagesPerSecond: await this.calculateMessagesPerSecond(),
      activeConnections: await this.getActiveConnections(),
      queueLength: await this.getQueueLength(),
      errorRate: await this.getCurrentErrorRate(),
      responseTime: await this.getCurrentResponseTime()
    };

    this.broadcast('realtime_metrics', metrics);
  }

  private async broadcastBusinessMetrics(): Promise<void> {
    const metrics = {
      timestamp: new Date(),
      activeCampaigns: await this.getActiveCampaignsCount(),
      totalUsers: await this.getTotalUsersCount(),
      revenueToday: await this.getTodayRevenue(),
      topPerformingCampaigns: await this.getTopPerformingCampaigns(5),
      recentAlerts: await this.getRecentAlerts(10)
    };

    this.broadcast('business_metrics', metrics);
  }

  private broadcast(type: string, data: any): void {
    const subscribers = this.subscribers.get(type) || [];
    const message = JSON.stringify({ type, data });

    subscribers.forEach(ws => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(message);
      }
    });
  }

  // Real-time alert broadcasting
  async broadcastAlert(alert: AlertEvent): Promise<void> {
    const alertData = {
      timestamp: new Date(),
      severity: alert.severity,
      service: alert.service,
      summary: alert.summary,
      description: alert.description,
      labels: alert.labels
    };

    this.broadcast('alerts', alertData);

    // Also send to monitoring channels
    await this.sendToSlack(alertData);
    
    if (alert.severity === 'critical') {
      await this.sendToPagerDuty(alertData);
    }
  }

  private async sendToSlack(alert: any): Promise<void> {
    // Slack webhook integration
    const webhook = process.env.SLACK_WEBHOOK_URL;
    if (!webhook) return;

    await fetch(webhook, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text: `🚨 ${alert.summary}`,
        attachments: [{
          color: alert.severity === 'critical' ? 'danger' : 'warning',
          fields: [
            { title: 'Service', value: alert.service, short: true },
            { title: 'Severity', value: alert.severity, short: true },
            { title: 'Description', value: alert.description, short: false }
          ],
          ts: Math.floor(alert.timestamp.getTime() / 1000)
        }]
      })
    });
  }

  // Predictive analytics
  async getPredictiveInsights(): Promise<PredictiveInsights> {
    const historicalData = await this.getHistoricalMetrics(30); // 30 days
    
    return {
      expectedMessageVolume: await this.predictMessageVolume(historicalData),
      costProjection: await this.predictCosts(historicalData),
      capacityRequirements: await this.predictCapacityNeeds(historicalData),
      riskFactors: await this.identifyRiskFactors(historicalData)
    };
  }
}
```

## Dashboards & Visualization

### Grafana Dashboard Configuration
```json
{
  "dashboard": {
    "title": "10DLC SMS System - Overview",
    "tags": ["sms", "10dlc", "overview"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "panels": [
      {
        "title": "Message Volume",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(messages_processed_total[5m]) * 60",
            "legendFormat": "Messages/min"
          }
        ],
        "gridPos": {"h": 8, "w": 6, "x": 0, "y": 0}
      },
      {
        "title": "API Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          },
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "99th percentile"
          }
        ],
        "yAxes": [
          {
            "label": "Response Time (seconds)",
            "min": 0
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 6, "y": 0}
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total{status_code=~\"5..\"}[5m]) / rate(http_requests_total[5m]) * 100",
            "legendFormat": "Error Rate %"
          }
        ],
        "thresholds": [
          {"color": "green", "value": 0},
          {"color": "yellow", "value": 1},
          {"color": "red", "value": 5}
        ],
        "gridPos": {"h": 8, "w": 6, "x": 18, "y": 0}
      },
      {
        "title": "Campaign Performance",
        "type": "table",
        "targets": [
          {
            "expr": "campaign_performance_score",
            "format": "table"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8}
      },
      {
        "title": "Message Queue Length",
        "type": "graph",
        "targets": [
          {
            "expr": "message_queue_length",
            "legendFormat": "{{queue_name}} - {{priority}}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8}
      },
      {
        "title": "Database Connections",
        "type": "graph",
        "targets": [
          {
            "expr": "database_connections_active",
            "legendFormat": "Active Connections"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 16}
      },
      {
        "title": "External API Status",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(external_api_calls_total{provider=\"telnyx\",status_code=~\"2..\"}[5m]) / rate(external_api_calls_total{provider=\"telnyx\"}[5m]) * 100",
            "legendFormat": "Telnyx Success Rate"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 16}
      }
    ]
  }
}
```

### Business Intelligence Dashboard
```json
{
  "dashboard": {
    "title": "Business Intelligence - 10DLC SMS",
    "tags": ["business", "kpi", "revenue"],
    "refresh": "5m",
    "panels": [
      {
        "title": "Revenue Metrics",
        "type": "row",
        "panels": [
          {
            "title": "Daily Revenue",
            "type": "graph",
            "targets": [
              {
                "expr": "increase(revenue_generated_total[1d])",
                "legendFormat": "Daily Revenue"
              }
            ]
          },
          {
            "title": "Revenue by Customer",
            "type": "piechart",
            "targets": [
              {
                "expr": "topk(10, sum by (customer_id) (revenue_generated_total))",
                "legendFormat": "{{customer_id}}"
              }
            ]
          }
        ]
      },
      {
        "title": "Operational Metrics",
        "type": "row",
        "panels": [
          {
            "title": "Message Delivery Rate",
            "type": "stat",
            "targets": [
              {
                "expr": "rate(messages_processed_total{status=\"delivered\"}[1h]) / rate(messages_processed_total{direction=\"outbound\"}[1h]) * 100"
              }
            ]
          },
          {
            "title": "Compliance Score",
            "type": "gauge",
            "targets": [
              {
                "expr": "avg(compliance_score)"
              }
            ]
          }
        ]
      }
    ]
  }
}
```

## Future Monitoring Enhancements

### AI-Powered Anomaly Detection (Phase 1 - Q2 2024)
```typescript
interface AnomalyDetectionService {
  detectAnomalies: (metrics: MetricSeries) => Promise<Anomaly[]>;
  predictFailures: (historicalData: HistoricalMetrics) => Promise<FailurePrediction>;
  optimizeAlerts: (alertHistory: AlertHistory) => Promise<OptimizedAlertRules>;
}

// Machine learning-based anomaly detection
class MLAnomalyDetector {
  async detectTrafficAnomalies(metrics: TrafficMetrics): Promise<TrafficAnomaly[]> {
    // Use time series analysis to detect unusual patterns
    const model = await this.loadTrafficModel();
    const predictions = await model.predict(metrics.timeSeries);
    
    return this.identifyAnomalies(metrics.actual, predictions);
  }

  async predictSystemFailures(systemMetrics: SystemMetrics): Promise<FailurePrediction> {
    // Predictive analytics for system failures
    const riskScore = await this.calculateRiskScore(systemMetrics);
    const timeToFailure = await this.predictTimeToFailure(systemMetrics);
    
    return {
      riskScore,
      timeToFailure,
      confidence: this.calculateConfidence(systemMetrics),
      recommendedActions: this.generateRecommendations(riskScore)
    };
  }
}
```

### Advanced Business Intelligence (Phase 2 - Q3 2024)
```typescript
interface AdvancedAnalytics {
  customerSegmentation: (data: CustomerData) => Promise<CustomerSegments>;
  campaignOptimization: (campaignData: CampaignData) => Promise<OptimizationRecommendations>;
  revenueForecasting: (historicalRevenue: RevenueData) => Promise<RevenueForecast>;
  marketAnalysis: (marketData: MarketData) => Promise<MarketInsights>;
}

// Real-time data streaming and processing
class RealTimeAnalytics {
  async setupDataPipeline(): Promise<void> {
    // Apache Kafka for real-time data streaming
    await this.setupKafkaStreams();
    
    // Apache Spark for real-time processing
    await this.setupSparkStreaming();
    
    // ClickHouse for real-time analytics
    await this.setupClickHouseCluster();
  }
}
```

## Conclusion

This monitoring and observability guide provides:

- **Comprehensive Metrics**: Application, infrastructure, and business metrics
- **Proactive Alerting**: Smart alerting with reduced false positives
- **Performance Monitoring**: APM integration with custom instrumentation
- **Business Intelligence**: KPI tracking and predictive analytics
- **Real-time Dashboards**: Live visualization and reporting
- **Future-Ready**: Roadmap for AI-powered monitoring and advanced analytics

The monitoring strategy ensures complete visibility into system health, performance, and business metrics while enabling proactive issue resolution and data-driven decision making.
