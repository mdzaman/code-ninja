# Chapter 5: Deployment, CI/CD, and Infrastructure Automation

## Table of Contents
1. Infrastructure as Code (IaC) Setup
2. CI/CD Pipeline Implementation
3. Monitoring and Observability
4. Security Automation
5. Performance Optimization
6. Disaster Recovery

## 1. Infrastructure as Code (IaC) Setup

### 1.1 CDK with TypeScript
```typescript
// infrastructure/app.ts
import * as cdk from 'aws-cdk-lib';
import { SaasStack } from './stacks/saas-stack';
import { MonitoringStack } from './stacks/monitoring-stack';
import { SecurityStack } from './stacks/security-stack';

const app = new cdk.App();

// Environmental configurations
const environmentConfig = {
  dev: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: 'us-east-1'
  },
  prod: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: 'us-east-1'
  }
};

// VPC and Network Stack
new NetworkStack(app, 'NetworkStack', {
  env: environmentConfig.dev,
  crossStackReferences: true
});

// Main SaaS Application Stack
new SaasStack(app, 'SaasStack', {
  env: environmentConfig.dev
});

// Monitoring and Observability Stack
new MonitoringStack(app, 'MonitoringStack', {
  env: environmentConfig.dev
});

// Security and Compliance Stack
new SecurityStack(app, 'SecurityStack', {
  env: environmentConfig.dev
});
```

### 1.2 Terraform Alternative Configuration
```hcl
# infrastructure/terraform/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
  
  backend "s3" {
    bucket = "terraform-state-saas"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

module "vpc" {
  source = "./modules/vpc"
  
  environment = var.environment
  cidr_block  = var.vpc_cidr
}

module "serverless" {
  source = "./modules/serverless"
  
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
}
```

## 2. CI/CD Pipeline Implementation

### 2.1 GitHub Actions Workflow
```yaml
# .github/workflows/main.yml
name: SaaS CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run tests
        run: npm test
        
      - name: Run security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Build application
        run: npm run build
        
      - name: Run CDK synth
        run: npx cdk synth
        
      - name: Store artifacts
        uses: actions/upload-artifact@v3
        with:
          name: cdk-artifacts
          path: cdk.out

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
          
      - name: Deploy to AWS
        run: npx cdk deploy --all --require-approval never
```

### 2.2 ArgoCD Kubernetes Deployment
```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: saas-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/saas-app.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: saas
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 3. Monitoring and Observability

### 3.1 Prometheus Configuration
```yaml
# monitoring/prometheus/config.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'lambda-metrics'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'api-gateway'
    static_configs:
      - targets: ['localhost:9091']

  - job_name: 'dynamodb'
    static_configs:
      - targets: ['localhost:9092']
```

### 3.2 Grafana Dashboard Configuration
```json
{
  "dashboard": {
    "id": null,
    "title": "SaaS Metrics Dashboard",
    "tags": ["saas", "production"],
    "timezone": "browser",
    "panels": [
      {
        "title": "API Response Time",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(api_response_time_seconds_sum[5m])",
            "legendFormat": "{{method}} {{path}}"
          }
        ]
      }
    ]
  }
}
```

## 4. Security Automation

### 4.1 OWASP ZAP Automation
```typescript
// security/zap-automation.ts
import { ZapClient } from 'zaproxy';

export async function runSecurityScan(targetUrl: string) {
  const zaproxy = new ZapClient({
    apiKey: process.env.ZAP_API_KEY,
    proxy: process.env.ZAP_PROXY_URL
  });

  // Start Spider scan
  const scanId = await zaproxy.spider.scan(targetUrl);
  await waitForSpiderToComplete(zaproxy, scanId);

  // Start Active scan
  const activeScanId = await zaproxy.ascan.scan(targetUrl);
  await waitForActiveScanToComplete(zaproxy, activeScanId);

  // Generate report
  const report = await zaproxy.core.htmlreport();
  return report;
}
```

## Learning Resources for Tools

### Infrastructure as Code
1. **AWS CDK**
   - Official Documentation: https://docs.aws.amazon.com/cdk/
   - egghead.io Course: https://egghead.io/courses/build-an-aws-cdk-app
   - Matt Coulter's CDK Patterns: https://github.com/cdk-patterns/serverless

2. **Terraform**
   - HashiCorp Learn: https://learn.hashicorp.com/terraform
   - Terraform Up & Running (Book): https://www.terraformupandrunning.com/
   - freeCodeCamp Course: https://www.youtube.com/watch?v=SLB_c_ayRMo

### CI/CD Tools
1. **GitHub Actions**
   - GitHub Learning Lab: https://lab.github.com/githubtraining/github-actions
   - Official Documentation: https://docs.github.com/en/actions
   - DevOps Journey Course: https://www.youtube.com/watch?v=R8_veQiYBjI

2. **ArgoCD**
   - Official Tutorials: https://argoproj.github.io/argo-cd/
   - Cloud Native TV Course: https://www.youtube.com/watch?v=MeU5_k9ssrs
   - ArgoCD Best Practices: https://www.youtube.com/watch?v=ESQLqjbM8h0

### Monitoring Tools
1. **Prometheus**
   - Official Documentation: https://prometheus.io/docs/
   - Prometheus Tutorial: https://www.youtube.com/watch?v=h4Sl21AKiDg
   - Cloud Native Monitoring: https://www.youtube.com/watch?v=QoDqxm7ybLc

2. **Grafana**
   - Getting Started: https://grafana.com/tutorials/
   - Grafana Fundamentals: https://www.youtube.com/watch?v=CjABEnRn6-Y
   - Dashboard Design: https://grafana.com/blog/2019/09/23/how-to-design-grafana-dashboards/

### Security Tools
1. **OWASP ZAP**
   - Official Documentation: https://www.zaproxy.org/docs/
   - ZAP in Ten: https://www.youtube.com/playlist?list=PLEBitBW-Hlsv8cEIUntAO8st2UGhmrjUB
   - Automated Security Testing: https://www.youtube.com/watch?v=dqKGGCVFTvI

2. **Snyk**
   - Snyk Learn: https://learn.snyk.io/
   - Security Testing: https://www.youtube.com/watch?v=Kx4tOodJBR0
   - Container Security: https://www.youtube.com/watch?v=QjZHeSr6wHE

### Additional Resources
1. **Cloud Native Computing Foundation**
   - CNCF Landscape: https://landscape.cncf.io/
   - KubeCon Talks: https://www.youtube.com/c/CloudNativeComputingFoundation

2. **AWS Well-Architected Labs**
   - Official Labs: https://www.wellarchitectedlabs.com/
   - AWS Solutions Library: https://aws.amazon.com/solutions/

3. **DevOps Communities**
   - DevOps Subreddit: https://www.reddit.com/r/devops/
   - DevOps Weekly Newsletter: https://www.devopsweekly.com/
   - The New Stack: https://thenewstack.io/

Would you like me to dive deeper into any specific tool or create the next chapter focusing on advanced features and integrations?
