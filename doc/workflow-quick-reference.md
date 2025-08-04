# Workflow Quick Reference

This document provides a concise overview of the system's key workflows for quick reference during development and troubleshooting.

## System Overview

```mermaid
graph LR
    A[用户触发] --> B[GitHub Actions]
    B --> C[Step Functions]
    C --> D[Aurora克隆]
    C --> E[EMR作业]
    C --> F[DynamoDB操作]
    C --> G[数据验证]
    C --> H[资源清理]
    
    style C fill:#fff3e0
    style E fill:#ffcc02
```

## Step Functions Main Flow

```mermaid
flowchart TD
    Start([开始]) --> Init[初始化变量]
    Init --> Clone[克隆Aurora数据库]
    Clone --> Bootstrap[运行Bootstrap作业]
    Bootstrap --> Dynamo[DynamoDB导入导出]
    Dynamo --> Compare[数据对比验证]
    Compare --> Cleanup[清理资源]
    Cleanup --> End([结束])
    
    style Bootstrap fill:#ffcc02
    style Compare fill:#ff9800
```

## Data Flow Summary

```mermaid
graph LR
    Aurora[Aurora分区数据库] --> Clone[克隆数据库]
    Clone --> EMR[EMR Spark作业]
    EMR --> S3[S3存储]
    S3 --> DDB[DynamoDB表]
    DDB --> Kinesis[Kinesis流]
    DDB --> Verify[数据验证]
```

## Key Parameters

| Parameter | Purpose | Default | Example |
|-----------|---------|---------|---------|
| `env` | Environment | - | `test`, `prod` |
| `processDomain` | Data domain | - | `rates`, `availability` |
| `io` | Official run flag | `false` | `true`, `false` |
| `readParallelism` | Read concurrency | `100` | `50`, `200` |
| `writeParallelism` | Write concurrency | `6` | `3`, `10` |

## Common Commands

### Manual Trigger
```bash
# Via GitHub Actions UI
# Go to Actions → Run Step Function Workflow
# Select parameters and click "Run workflow"
```

### Status Monitoring
```bash
# Step Functions execution ARN will be shown in GitHub Actions logs
# Monitor in AWS Console: Step Functions → State machines → Executions
```

### Troubleshooting
```bash
# Check CloudWatch logs for EMR jobs
# Check S3 for output files and logs
# Check DynamoDB tables for import status
```

---

*For detailed diagrams and comprehensive documentation, see [Architecture Diagrams](architecture-diagrams.md)*