# Deployment & Operations Guide - 10DLC SMS System

## Table of Contents
1. [Overview](#overview)
2. [Infrastructure Architecture](#infrastructure-architecture)
3. [Container Strategy](#container-strategy)
4. [CI/CD Pipeline](#cicd-pipeline)
5. [Environment Management](#environment-management)
6. [Auto-Scaling & Load Balancing](#auto-scaling--load-balancing)
7. [Database Operations](#database-operations)
8. [Monitoring & Observability](#monitoring--observability)
9. [Disaster Recovery](#disaster-recovery)
10. [Operational Procedures](#operational-procedures)
11. [Performance Optimization](#performance-optimization)
12. [Future Scaling Strategies](#future-scaling-strategies)

## Overview

This document provides comprehensive guidance for deploying, operating, and scaling the 10DLC SMS system across development, staging, and production environments.

### Deployment Objectives
- **High Availability**: 99.9% uptime SLA
- **Scalability**: Handle 1M+ messages per day
- **Performance**: <200ms API response time
- **Security**: Zero-trust network architecture
- **Compliance**: SOC 2 Type II ready infrastructure
- **Cost Optimization**: Efficient resource utilization

### Technology Stack
- **Container Platform**: Docker + Kubernetes
- **Cloud Provider**: AWS (multi-AZ deployment)
- **CI/CD**: GitHub Actions + ArgoCD
- **Infrastructure as Code**: Terraform + Helm
- **Service Mesh**: Istio
- **Monitoring**: Prometheus + Grafana + ELK Stack

## Infrastructure Architecture

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                     Internet Gateway                        │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    Application Load Balancer               │
│  • SSL Termination                                         │
│  • WAF Integration                                         │
│  • Health Checks                                           │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼──────┐    ┌────────▼────────┐    ┌──────▼──────┐
│   Public     │    │   Private       │    │  Database   │
│   Subnet     │    │   Subnet        │    │   Subnet    │
│              │    │                 │    │             │
│ • NAT GW     │    │ • EKS Nodes     │    │ • RDS       │
│ • Bastion    │    │ • App Pods      │    │ • ElastiC   │
│              │    │ • Redis         │    │ • Backup    │
└──────────────┘    └─────────────────┘    └─────────────┘
```

### Multi-Region Setup
```yaml
# Primary Region: us-east-1
Primary:
  Region: us-east-1
  Availability Zones:
    - us-east-1a
    - us-east-1b
    - us-east-1c
  Services:
    - EKS Cluster (Production)
    - RDS Primary
    - ElastiCache Primary
    - S3 Buckets

# Secondary Region: us-west-2 (DR)
Secondary:
  Region: us-west-2
  Availability Zones:
    - us-west-2a
    - us-west-2b
  Services:
    - EKS Cluster (Standby)
    - RDS Read Replica
    - ElastiCache Replica
    - S3 Cross-Region Replication
```

### Terraform Infrastructure
```hcl
# terraform/main.tf
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = "10dlc-sms-system"
      ManagedBy   = "terraform"
    }
  }
}

# VPC Configuration
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "${var.environment}-10dlc-vpc"
  cidr = var.vpc_cidr
  
  azs             = var.availability_zones
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs
  database_subnets = var.database_subnet_cidrs
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  enable_dns_hostnames = true
  enable_dns_support = true
  
  tags = {
    Environment = var.environment
  }
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = "${var.environment}-10dlc-cluster"
  cluster_version = "1.27"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = true
  
  node_groups = {
    main = {
      desired_capacity = 3
      max_capacity     = 10
      min_capacity     = 3
      
      instance_types = ["t3.medium"]
      
      k8s_labels = {
        Environment = var.environment
        NodeGroup   = "main"
      }
    }
    
    high_memory = {
      desired_capacity = 2
      max_capacity     = 5
      min_capacity     = 2
      
      instance_types = ["r5.large"]
      
      k8s_labels = {
        Environment = var.environment
        NodeGroup   = "high-memory"
      }
      
      taints = [{
        key    = "workload"
        value  = "memory-intensive"
        effect = "NO_SCHEDULE"
      }]
    }
  }
}

# RDS Database
resource "aws_db_instance" "main" {
  identifier = "${var.environment}-10dlc-db"
  
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = var.db_instance_class
  
  allocated_storage     = var.db_allocated_storage
  max_allocated_storage = var.db_max_allocated_storage
  storage_encrypted     = true
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "Sun:04:00-Sun:05:00"
  
  deletion_protection = var.environment == "production"
  skip_final_snapshot = var.environment != "production"
  
  tags = {
    Name = "${var.environment}-10dlc-database"
  }
}
```

## Container Strategy

### Docker Configuration
```dockerfile
# Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production image
FROM node:18-alpine AS production

# Security: Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy built application
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

# Security: Run as non-root user
USER nodejs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["npm", "start"]
```

### Multi-Stage Build Strategy
```dockerfile
# Development image
FROM node:18-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["npm", "run", "dev"]

# Testing image
FROM development AS testing
RUN npm run test
RUN npm run lint
RUN npm audit --audit-level=high

# Production image (as above)
FROM node:18-alpine AS production
# ... production configuration
```

### Kubernetes Manifests

#### Deployment
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sms-api
  namespace: production
  labels:
    app: sms-api
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: sms-api
  template:
    metadata:
      labels:
        app: sms-api
        version: v1
    spec:
      serviceAccountName: sms-api
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: sms-api
        image: 10dlc-sms-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: password
        - name: TELNYX_API_KEY
          valueFrom:
            secretKeyRef:
              name: telnyx-secret
              key: api-key
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: logs
          mountPath: /app/logs
      volumes:
      - name: config
        configMap:
          name: sms-api-config
      - name: logs
        emptyDir: {}
```

#### Service
```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sms-api-service
  namespace: production
  labels:
    app: sms-api
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
  selector:
    app: sms-api
```

#### Ingress
```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sms-api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - api.10dlc-sms.com
    secretName: sms-api-tls
  rules:
  - host: api.10dlc-sms.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sms-api-service
            port:
              number: 80
```

## CI/CD Pipeline

### GitHub Actions Workflow
```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linting
      run: npm run lint
    
    - name: Run tests
      run: npm test
    
    - name: Run security audit
      run: npm audit --audit-level=high
    
    - name: Code coverage
      run: npm run test:coverage
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.image.outputs.image }}
    steps:
    - uses: actions/checkout@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
    
    - name: Build and push image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    - name: Output image
      id: image
      run: echo "image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}" >> $GITHUB_OUTPUT

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
    
    - name: Update kubeconfig
      run: aws eks update-kubeconfig --name staging-10dlc-cluster
    
    - name: Deploy to staging
      run: |
        kubectl set image deployment/sms-api sms-api=${{ needs.build.outputs.image }} -n staging
        kubectl rollout status deployment/sms-api -n staging --timeout=600s
    
    - name: Run smoke tests
      run: |
        npm run test:smoke -- --baseUrl=https://staging-api.10dlc-sms.com

  deploy-production:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
    
    - name: Update kubeconfig
      run: aws eks update-kubeconfig --name production-10dlc-cluster
    
    - name: Deploy to production
      run: |
        kubectl set image deployment/sms-api sms-api=${{ needs.build.outputs.image }} -n production
        kubectl rollout status deployment/sms-api -n production --timeout=600s
    
    - name: Run production health checks
      run: |
        sleep 30
        curl -f https://api.10dlc-sms.com/health
    
    - name: Notify deployment
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### ArgoCD GitOps Configuration
```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sms-api-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/10dlc-sms-system
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Kustomization for Environment Overlays
```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- ../../base

patchesStrategicMerge:
- deployment-patch.yaml
- service-patch.yaml

configMapGenerator:
- name: sms-api-config
  files:
  - config/production.yaml

secretGenerator:
- name: database-secret
  envs:
  - secrets/database.env
- name: telnyx-secret
  envs:
  - secrets/telnyx.env

images:
- name: sms-api
  newName: ghcr.io/your-org/10dlc-sms-system
  newTag: latest
```

## Environment Management

### Environment Configuration
```typescript
// config/environments/production.ts
export default {
  app: {
    name: 'SMS API',
    version: process.env.APP_VERSION || '1.0.0',
    environment: 'production',
    port: parseInt(process.env.PORT || '3000', 10),
    logLevel: 'warn',
    debug: false
  },
  database: {
    host: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT || '3306', 10),
    database: process.env.DB_NAME,
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    ssl: true,
    pool: {
      min: 5,
      max: 20,
      idle: 10000,
      acquire: 60000
    }
  },
  redis: {
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT || '6379', 10),
    password: process.env.REDIS_PASSWORD,
    tls: true,
    retryDelayOnFailover: 100,
    maxRetriesPerRequest: 3
  },
  telnyx: {
    apiKey: process.env.TELNYX_API_KEY,
    webhookSecret: process.env.TELNYX_WEBHOOK_SECRET,
    baseUrl: 'https://api.telnyx.com/v2',
    timeout: 30000
  },
  monitoring: {
    enabled: true,
    metricsPort: 9090,
    healthCheckInterval: 30000
  }
};
```

### Secret Management
```yaml
# k8s/secrets/sealed-secret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: database-secret
  namespace: production
spec:
  encryptedData:
    host: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEQAx...
    password: AgAKAoiQm7UWrLGv4kV6wRs5Q2GZx9kN...
    username: AgAjbQ9XnH+5G7Zy2QvFd8Ht9KmL1Np...
  template:
    metadata:
      name: database-secret
      namespace: production
```

### Configuration Management with Helm
```yaml
# helm/values.yaml
global:
  environment: production
  registry: ghcr.io/your-org
  imageTag: latest

app:
  name: sms-api
  replicaCount: 3
  image:
    repository: 10dlc-sms-system
    pullPolicy: Always
  
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"

database:
  host: "production-10dlc-db.cluster-xyz.us-east-1.rds.amazonaws.com"
  port: 3306
  name: "sms_system"

redis:
  enabled: true
  host: "production-redis.cache.amazonaws.com"
  port: 6379

ingress:
  enabled: true
  className: "nginx"
  hostname: "api.10dlc-sms.com"
  tls:
    enabled: true
    secretName: "sms-api-tls"

monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
    interval: 30s

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

## Auto-Scaling & Load Balancing

### Horizontal Pod Autoscaler
```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sms-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sms-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: custom_queue_length
      target:
        type: AverageValue
        averageValue: "10"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### Vertical Pod Autoscaler
```yaml
# k8s/vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: sms-api-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: sms-api
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: sms-api
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 1000m
        memory: 1Gi
      controlledResources: ["cpu", "memory"]
```

### Load Balancer Configuration
```yaml
# k8s/aws-load-balancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: sms-api-nlb
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "6"
    service.beta.kubernetes.io/aws-load-balancer-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-unhealthy-threshold: "2"
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 3000
    protocol: TCP
  selector:
    app: sms-api
```

### Application Performance Monitoring
```typescript
// src/middleware/performance.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { performance } from 'perf_hooks';
import { promisify } from 'util';

interface PerformanceMetrics {
  responseTime: number;
  memoryUsage: NodeJS.MemoryUsage;
  cpuUsage: NodeJS.CpuUsage;
  activeConnections: number;
}

class PerformanceMonitor {
  private metrics: Map<string, PerformanceMetrics[]> = new Map();

  middleware() {
    return (req: Request, res: Response, next: NextFunction): void => {
      const startTime = performance.now();
      const startCPU = process.cpuUsage();

      res.on('finish', () => {
        const endTime = performance.now();
        const endCPU = process.cpuUsage(startCPU);
        
        const metrics: PerformanceMetrics = {
          responseTime: endTime - startTime,
          memoryUsage: process.memoryUsage(),
          cpuUsage: endCPU,
          activeConnections: this.getActiveConnections()
        };

        this.recordMetrics(req.route?.path || req.path, metrics);
        this.checkPerformanceThresholds(metrics);
      });

      next();
    };
  }

  private checkPerformanceThresholds(metrics: PerformanceMetrics): void {
    // Alert if response time > 2 seconds
    if (metrics.responseTime > 2000) {
      this.alertingService.sendAlert({
        type: 'performance',
        severity: 'warning',
        message: `High response time: ${metrics.responseTime}ms`
      });
    }

    // Alert if memory usage > 80%
    const memoryUsagePercent = (metrics.memoryUsage.heapUsed / metrics.memoryUsage.heapTotal) * 100;
    if (memoryUsagePercent > 80) {
      this.alertingService.sendAlert({
        type: 'memory',
        severity: 'critical',
        message: `High memory usage: ${memoryUsagePercent}%`
      });
    }
  }
}
```

## Database Operations

### Database Migration Strategy
```typescript
// src/migrations/production-migration.ts
import { DataSource } from 'typeorm';

export class ProductionMigrationManager {
  constructor(private dataSource: DataSource) {}

  async runMigration(migrationName: string): Promise<void> {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // Check current schema version
      const currentVersion = await this.getCurrentSchemaVersion(queryRunner);
      
      // Validate migration prerequisites
      await this.validateMigrationPrerequisites(migrationName, currentVersion);
      
      // Create migration backup
      const backupId = await this.createPreMigrationBackup();
      
      // Run migration with monitoring
      await this.executeMigrationWithMonitoring(queryRunner, migrationName);
      
      // Validate post-migration state
      await this.validatePostMigrationState(queryRunner);
      
      // Update schema version
      await this.updateSchemaVersion(queryRunner, migrationName);
      
      await queryRunner.commitTransaction();
      
      logger.info(`Migration ${migrationName} completed successfully`);
      
    } catch (error) {
      await queryRunner.rollbackTransaction();
      
      // Alert on migration failure
      await this.alertMigrationFailure(migrationName, error);
      
      // Optionally restore from backup
      await this.considerBackupRestore(backupId);
      
      throw error;
    } finally {
      await queryRunner.release();
    }
  }

  private async createPreMigrationBackup(): Promise<string> {
    const backupId = `migration_backup_${Date.now()}`;
    
    await this.backupService.createSnapshot({
      id: backupId,
      type: 'migration_backup',
      retentionDays: 7
    });
    
    return backupId;
  }
}
```

### Database Connection Pool Management
```typescript
// src/config/database-pool.config.ts
export const createDatabasePool = () => {
  return new DataSource({
    type: 'mysql',
    host: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT || '3306'),
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    
    // Connection pooling configuration
    extra: {
      connectionLimit: 20,
      acquireTimeout: 60000,
      timeout: 60000,
      reconnect: true,
      
      // Pool monitoring
      onConnection: (connection: any) => {
        logger.debug('New database connection established', {
          connectionId: connection.threadId
        });
      },
      
      onRelease: (connection: any) => {
        logger.debug('Database connection released', {
          connectionId: connection.threadId
        });
      },
      
      onEnqueue: () => {
        logger.warn('Database connection request queued');
      }
    },
    
    // Health check query
    healthCheck: async (connection: any) => {
      try {
        await connection.query('SELECT 1');
        return true;
      } catch (error) {
        logger.error('Database health check failed', error);
        return false;
      }
    }
  });
};
```

### Database Backup and Recovery
```bash
#!/bin/bash
# scripts/backup-database.sh

set -e

ENVIRONMENT=${1:-production}
BACKUP_TYPE=${2:-full}
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)

# Configuration
BACKUP_BUCKET="s3://10dlc-database-backups"
BACKUP_PATH="${BACKUP_BUCKET}/${ENVIRONMENT}/${BACKUP_DATE}"
RETENTION_DAYS=30

# Database credentials from environment
DB_HOST=${DB_HOST}
DB_NAME=${DB_NAME}
DB_USER=${DB_USERNAME}
DB_PASS=${DB_PASSWORD}

echo "Starting ${BACKUP_TYPE} backup for ${ENVIRONMENT} environment..."

if [ "$BACKUP_TYPE" = "full" ]; then
    # Full backup
    mysqldump \
        --host=${DB_HOST} \
        --user=${DB_USER} \
        --password=${DB_PASS} \
        --single-transaction \
        --routines \
        --triggers \
        --all-databases \
        --compress | \
    gzip > backup_${BACKUP_DATE}.sql.gz
    
elif [ "$BACKUP_TYPE" = "incremental" ]; then
    # Incremental backup using binary logs
    LAST_BACKUP=$(aws s3 ls ${BACKUP_BUCKET}/${ENVIRONMENT}/ | tail -1 | awk '{print $4}')
    LAST_BACKUP_TIME=$(echo $LAST_BACKUP | cut -d'_' -f1-2 | tr '_' ' ')
    
    mysqlbinlog \
        --start-datetime="$LAST_BACKUP_TIME" \
        --host=${DB_HOST} \
        --user=${DB_USER} \
        --password=${DB_PASS} \
        --read-from-remote-server \
        --raw | \
    gzip > incremental_backup_${BACKUP_DATE}.sql.gz
fi

# Upload to S3
aws s3 cp backup_${BACKUP_DATE}.sql.gz ${BACKUP_PATH}/

# Cleanup local files
rm -f backup_${BACKUP_DATE}.sql.gz incremental_backup_${BACKUP_DATE}.sql.gz

# Cleanup old backups
aws s3 ls ${BACKUP_BUCKET}/${ENVIRONMENT}/ | \
while read -r line; do
    backup_date=$(echo $line | awk '{print $1" "$2}')
    backup_file=$(echo $line | awk '{print $4}')
    
    if [[ $(date -d "$backup_date" +%s) -lt $(date -d "$RETENTION_DAYS days ago" +%s) ]]; then
        aws s3 rm ${BACKUP_BUCKET}/${ENVIRONMENT}/${backup_file}
        echo "Deleted old backup: $backup_file"
    fi
done

echo "Backup completed successfully: ${BACKUP_PATH}"

# Verify backup integrity
aws s3 ls ${BACKUP_PATH}/ && echo "Backup verified in S3"

# Send notification
curl -X POST "${SLACK_WEBHOOK_URL}" \
    -H 'Content-type: application/json' \
    --data "{\"text\":\"Database backup completed: ${ENVIRONMENT} - ${BACKUP_TYPE} - ${BACKUP_DATE}\"}"
```

## Monitoring & Observability

### Prometheus Metrics Configuration
```yaml
# k8s/monitoring/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sms-api-metrics
  namespace: production
  labels:
    app: sms-api
spec:
  selector:
    matchLabels:
      app: sms-api
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    honorLabels: true
```

### Custom Metrics Implementation
```typescript
// src/monitoring/metrics.ts
import { register, Counter, Histogram, Gauge } from 'prom-client';

export class MetricsService {
  private httpRequestsTotal = new Counter({
    name: 'http_requests_total',
    help: 'Total number of HTTP requests',
    labelNames: ['method', 'route', 'status_code']
  });

  private httpRequestDuration = new Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests in seconds',
    labelNames: ['method', 'route'],
    buckets: [0.1, 0.5, 1, 2, 5, 10]
  });

  private messagesProcessed = new Counter({
    name: 'messages_processed_total',
    help: 'Total number of messages processed',
    labelNames: ['direction', 'status', 'campaign_id']
  });

  private activeConnections = new Gauge({
    name: 'active_connections',
    help: 'Number of active database connections',
    collect() {
      this.set(getDatabaseConnectionCount());
    }
  });

  private queueLength = new Gauge({
    name: 'message_queue_length',
    help: 'Current length of message processing queue'
  });

  constructor() {
    register.registerMetric(this.httpRequestsTotal);
    register.registerMetric(this.httpRequestDuration);
    register.registerMetric(this.messagesProcessed);
    register.registerMetric(this.activeConnections);
    register.registerMetric(this.queueLength);
  }

  recordHttpRequest(method: string, route: string, statusCode: number, duration: number): void {
    this.httpRequestsTotal.inc({ method, route, status_code: statusCode });
    this.httpRequestDuration.observe({ method, route }, duration);
  }

  recordMessageProcessed(direction: string, status: string, campaignId: string): void {
    this.messagesProcessed.inc({ direction, status, campaign_id: campaignId });
  }

  updateQueueLength(length: number): void {
    this.queueLength.set(length);
  }

  getMetrics(): string {
    return register.metrics();
  }
}
```

### Grafana Dashboard Configuration
```json
{
  "dashboard": {
    "title": "10DLC SMS System - Production",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{route}}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          }
        ]
      },
      {
        "title": "Message Processing Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(messages_processed_total[5m])",
            "legendFormat": "{{direction}} - {{status}}"
          }
        ]
      },
      {
        "title": "System Resources",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total[5m])",
            "legendFormat": "CPU Usage"
          },
          {
            "expr": "container_memory_usage_bytes / container_spec_memory_limit_bytes",
            "legendFormat": "Memory Usage"
          }
        ]
      }
    ]
  }
}
```

## Disaster Recovery

### Multi-Region Failover Strategy
```typescript
// src/services/disaster-recovery.service.ts
export class DisasterRecoveryService {
  private primaryRegion = 'us-east-1';
  private secondaryRegion = 'us-west-2';
  private failoverThreshold = 5; // minutes

  async monitorPrimaryRegion(): Promise<void> {
    setInterval(async () => {
      const primaryHealth = await this.checkRegionHealth(this.primaryRegion);
      
      if (!primaryHealth.isHealthy && primaryHealth.downTime > this.failoverThreshold) {
        await this.initiateFailover();
      }
    }, 30000); // Check every 30 seconds
  }

  private async initiateFailover(): Promise<void> {
    logger.critical('Initiating disaster recovery failover');
    
    try {
      // 1. Stop accepting new requests in primary region
      await this.drainPrimaryTraffic();
      
      // 2. Promote read replica to primary in secondary region
      await this.promoteSecondaryDatabase();
      
      // 3. Update DNS to point to secondary region
      await this.updateDNSFailover();
      
      // 4. Scale up secondary region infrastructure
      await this.scaleSecondaryRegion();
      
      // 5. Notify operations team
      await this.notifyFailover();
      
      logger.info('Disaster recovery failover completed');
      
    } catch (error) {
      logger.error('Failover failed', error);
      await this.alertFailoverFailure(error);
    }
  }

  private async promoteSecondaryDatabase(): Promise<void> {
    const rds = new AWS.RDS({ region: this.secondaryRegion });
    
    await rds.promoteReadReplica({
      DBInstanceIdentifier: 'secondary-10dlc-db'
    }).promise();
    
    // Wait for promotion to complete
    await this.waitForDatabaseAvailability('secondary-10dlc-db');
  }

  private async updateDNSFailover(): Promise<void> {
    const route53 = new AWS.Route53();
    
    await route53.changeResourceRecordSets({
      HostedZoneId: process.env.HOSTED_ZONE_ID,
      ChangeBatch: {
        Changes: [{
          Action: 'UPSERT',
          ResourceRecordSet: {
            Name: 'api.10dlc-sms.com',
            Type: 'CNAME',
            TTL: 60,
            ResourceRecords: [{
              Value: `secondary-lb.${this.secondaryRegion}.elb.amazonaws.com`
            }]
          }
        }]
      }
    }).promise();
  }
}
```

### Backup Recovery Procedures
```bash
#!/bin/bash
# scripts/disaster-recovery.sh

RECOVERY_TYPE=${1:-full}
BACKUP_DATE=${2:-latest}
TARGET_ENVIRONMENT=${3:-production}

echo "Starting disaster recovery: ${RECOVERY_TYPE} from ${BACKUP_DATE}"

case $RECOVERY_TYPE in
    "full")
        # Full system recovery
        echo "Performing full system recovery..."
        
        # 1. Restore database
        ./scripts/restore-database.sh $BACKUP_DATE $TARGET_ENVIRONMENT
        
        # 2. Restore application state
        kubectl apply -f k8s/overlays/$TARGET_ENVIRONMENT/
        
        # 3. Verify system health
        ./scripts/health-check.sh $TARGET_ENVIRONMENT
        ;;
        
    "database")
        # Database-only recovery
        echo "Performing database recovery..."
        ./scripts/restore-database.sh $BACKUP_DATE $TARGET_ENVIRONMENT
        ;;
        
    "application")
        # Application-only recovery
        echo "Performing application recovery..."
        kubectl rollout restart deployment/sms-api -n $TARGET_ENVIRONMENT
        kubectl rollout status deployment/sms-api -n $TARGET_ENVIRONMENT
        ;;
        
    *)
        echo "Unknown recovery type: $RECOVERY_TYPE"
        exit 1
        ;;
esac

echo "Disaster recovery completed"
```

## Operational Procedures

### Daily Operations Checklist
```bash
#!/bin/bash
# scripts/daily-operations.sh

echo "=== Daily Operations Checklist ==="
echo "Date: $(date)"

# 1. System Health Check
echo "1. Checking system health..."
kubectl get pods -A | grep -v Running
kubectl top nodes
kubectl top pods -A

# 2. Database Health
echo "2. Checking database health..."
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS -e "SHOW PROCESSLIST;"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS -e "SHOW GLOBAL STATUS LIKE 'Threads_%';"

# 3. Message Processing Stats
echo "3. Message processing stats..."
curl -s https://api.10dlc-sms.com/admin/stats | jq .

# 4. Error Rates
echo "4. Checking error rates..."
kubectl logs -l app=sms-api --since=24h | grep ERROR | wc -l

# 5. Security Alerts
echo "5. Checking security alerts..."
curl -s https://security-api.10dlc-sms.com/alerts/today | jq .

# 6. Backup Verification
echo "6. Verifying recent backups..."
aws s3 ls s3://10dlc-database-backups/production/ | tail -5

# 7. Certificate Expiry
echo "7. Checking certificate expiry..."
echo | openssl s_client -servername api.10dlc-sms.com -connect api.10dlc-sms.com:443 2>/dev/null | openssl x509 -noout -dates

echo "=== Operations check completed ==="
```

### Incident Response Procedures
```typescript
// src/procedures/incident-response.ts
export class IncidentResponseProcedures {
  
  async handleHighCPUUsage(): Promise<void> {
    logger.warn('High CPU usage detected, initiating response procedure');
    
    // 1. Scale up replicas
    await this.scaleReplicas(this.getCurrentReplicas() * 2);
    
    // 2. Enable circuit breaker
    await this.enableCircuitBreaker();
    
    // 3. Alert engineering team
    await this.alertEngineering('high_cpu_usage');
    
    // 4. Collect diagnostics
    await this.collectDiagnostics();
  }

  async handleDatabaseConnectionFailure(): Promise<void> {
    logger.error('Database connection failure detected');
    
    // 1. Attempt connection pool restart
    await this.restartConnectionPool();
    
    // 2. If that fails, switch to read-only mode
    if (!await this.testDatabaseConnection()) {
      await this.enableReadOnlyMode();
    }
    
    // 3. Alert DBA team
    await this.alertDBA('database_connection_failure');
    
    // 4. Initiate database failover if needed
    if (await this.shouldFailoverDatabase()) {
      await this.initiateDatabaseFailover();
    }
  }

  async handleTelnyxAPIFailure(): Promise<void> {
    logger.error('Telnyx API failure detected');
    
    // 1. Enable message queuing
    await this.enableMessageQueuing();
    
    // 2. Implement exponential backoff
    await this.enableExponentialBackoff();
    
    // 3. Alert operations team
    await this.alertOperations('telnyx_api_failure');
    
    // 4. Consider backup provider if available
    if (this.hasBackupProvider()) {
      await this.switchToBackupProvider();
    }
  }
}
```

## Performance Optimization

### Application Performance Tuning
```typescript
// src/optimization/performance-tuner.ts
export class PerformanceTuner {
  
  async optimizeNodeJS(): Promise<void> {
    // Optimize V8 flags
    process.env.NODE_OPTIONS = [
      '--max-old-space-size=1024',
      '--max-new-space-size=1024',
      '--optimize-for-size',
      '--gc-interval=100'
    ].join(' ');
    
    // Enable clustering for CPU-intensive tasks
    if (cluster.isMaster && process.env.NODE_ENV === 'production') {
      const numCPUs = os.cpus().length;
      for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
      }
    }
  }

  async optimizeDatabase(): Promise<void> {
    // Connection pooling optimization
    const pool = mysql.createPool({
      connectionLimit: 20,
      queueLimit: 0,
      acquireTimeout: 60000,
      timeout: 60000,
      reconnect: true,
      
      // MySQL-specific optimizations
      ssl: {
        rejectUnauthorized: false
      },
      charset: 'utf8mb4',
      timezone: 'UTC'
    });

    // Query optimization
    await this.optimizeQueries();
    await this.updateIndexes();
    await this.analyzeSlowQueries();
  }

  async optimizeRedis(): Promise<void> {
    const redis = new Redis({
      host: process.env.REDIS_HOST,
      port: 6379,
      
      // Connection optimization
      maxRetriesPerRequest: 3,
      retryDelayOnFailover: 100,
      connectTimeout: 10000,
      commandTimeout: 5000,
      
      // Memory optimization
      lazyConnect: true,
      keepAlive: 30000,
      
      // Clustering for high availability
      enableReadyCheck: true,
      maxClusterRedirections: 3
    });
  }
}
```

### Caching Strategy
```typescript
// src/caching/cache-strategy.ts
export class CacheStrategy {
  private redis: Redis;
  private localCache: NodeCache;

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL);
    this.localCache = new NodeCache({ 
      stdTTL: 300, // 5 minutes
      checkperiod: 60 // 1 minute
    });
  }

  async get<T>(key: string): Promise<T | null> {
    // L1 Cache: Local memory
    let data = this.localCache.get<T>(key);
    if (data) {
      return data;
    }

    // L2 Cache: Redis
    const cached = await this.redis.get(key);
    if (cached) {
      data = JSON.parse(cached);
      this.localCache.set(key, data, 300); // Cache locally for 5 minutes
      return data;
    }

    return null;
  }

  async set<T>(key: string, value: T, ttl: number = 3600): Promise<void> {
    // Store in both caches
    this.localCache.set(key, value, Math.min(ttl, 300));
    await this.redis.setex(key, ttl, JSON.stringify(value));
  }

  // Cache warming for frequently accessed data
  async warmCache(): Promise<void> {
    const popularCampaigns = await this.getPopularCampaigns();
    
    for (const campaign of popularCampaigns) {
      await this.set(`campaign:${campaign.id}`, campaign, 7200); // 2 hours
    }
  }

  // Cache invalidation strategy
  async invalidatePattern(pattern: string): Promise<void> {
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
    
    // Invalidate local cache entries
    const localKeys = this.localCache.keys();
    localKeys.forEach(key => {
      if (key.match(pattern)) {
        this.localCache.del(key);
      }
    });
  }
}
```

## Future Scaling Strategies

### Microservices Architecture (Phase 1 - Q2 2024)
```typescript
// Future microservices breakdown
interface MicroserviceArchitecture {
  authService: {
    responsibilities: ['user authentication', 'authorization', 'session management'];
    database: 'auth_db';
    scaling: 'horizontal';
  };
  
  brandService: {
    responsibilities: ['brand management', 'TCR integration'];
    database: 'brand_db';
    scaling: 'horizontal';
  };
  
  campaignService: {
    responsibilities: ['campaign management', 'compliance validation'];
    database: 'campaign_db';
    scaling: 'horizontal';
  };
  
  messageService: {
    responsibilities: ['message processing', 'queue management'];
    database: 'message_db';
    scaling: 'horizontal + vertical';
  };
  
  telnyxService: {
    responsibilities: ['Telnyx API integration', 'webhook processing'];
    database: 'none';
    scaling: 'horizontal';
  };
  
  analyticsService: {
    responsibilities: ['reporting', 'metrics aggregation'];
    database: 'analytics_db';
    scaling: 'horizontal';
  };
}
```

### Event-Driven Architecture (Phase 2 - Q3 2024)
```typescript
// Event sourcing implementation
interface EventDrivenArchitecture {
  eventStore: {
    technology: 'Apache Kafka';
    partitions: 12;
    replicationFactor: 3;
    retentionPeriod: '7 days';
  };
  
  eventProcessors: {
    messageProcessor: {
      consumerGroup: 'message-processors';
      parallelism: 10;
      scaling: 'auto';
    };
    
    analyticsProcessor: {
      consumerGroup: 'analytics-processors';
      parallelism: 5;
      scaling: 'scheduled';
    };
    
    complianceProcessor: {
      consumerGroup: 'compliance-processors';
      parallelism: 3;
      scaling: 'manual';
    };
  };
  
  eventTypes: [
    'MessageSent',
    'MessageDelivered',
    'MessageFailed',
    'UserOptedOut',
    'CampaignApproved',
    'BrandRegistered'
  ];
}
```

### Global Multi-Region Deployment (Phase 3 - Q4 2024)
```yaml
# Global deployment strategy
regions:
  primary:
    region: us-east-1
    zones: [us-east-1a, us-east-1b, us-east-1c]
    traffic: 60%
    services: [all]
    
  secondary:
    region: us-west-2  
    zones: [us-west-2a, us-west-2b]
    traffic: 25%
    services: [api, messaging]
    
  tertiary:
    region: eu-west-1
    zones: [eu-west-1a, eu-west-1b]
    traffic: 15%
    services: [api, messaging]

traffic_routing:
  strategy: geolocation + latency
  failover: automatic
  health_checks: multi-layer
  
data_strategy:
  primary_writes: us-east-1
  read_replicas: all_regions
  cache_strategy: regional
  backup_strategy: cross_region
```

## Conclusion

This deployment and operations guide provides:

- **Comprehensive Infrastructure**: Production-ready Kubernetes deployment
- **Automated CI/CD**: Full pipeline from code to production
- **Scalability**: Auto-scaling and load balancing strategies
- **Reliability**: Disaster recovery and backup procedures
- **Observability**: Complete monitoring and alerting setup
- **Future-Ready**: Roadmap for scaling to global deployment

The architecture supports current requirements while providing clear paths for future growth including microservices, event-driven architecture, and global multi-region deployment.
