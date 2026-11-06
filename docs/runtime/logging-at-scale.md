# Logging for Scale: Millions of Claims

For 1M+ claims/day, detailed per-call logging becomes expensive and unnecessary. Use **adaptive sampling + aggregated metrics + anomaly-based logging**.

## Strategy: Three-Tier Logging

```typescript
// logging.strategy.ts
export enum LogLevel {
  NONE = 0,           // No detail
  METRICS_ONLY = 1,   // Count + duration only
  ANOMALY = 2,        // Log only when decision differs from baseline
  SAMPLE = 3,         // Random sample (e.g., 1%)
  FULL = 4,           // Every call (for dev/debugging)
}

export interface LogConfig {
  defaultLevel: LogLevel;
  sampleRate: number; // 0-1, e.g. 0.01 = 1%
  anomalyThreshold: number; // Flag if decision differs from expected in >N% of similar claims
  highRiskRules: Set<string>; // Always log these
  storeBackend: 'memory' | 'kafka' | 'datadog' | 's3';
}

export class AdaptiveLogger {
  private config: LogConfig;
  private metrics = {
    claimsProcessed: 0,
    claimsApproved: 0,
    claimsDenied: 0,
    claimsEscalated: 0,
    avgDecisionTimeMs: 0,
    totalCostPoints: 0,
  };
  private anomalies: AnomalyRecord[] = [];
  private sampleBuffer: DetailedLog[] = [];

  constructor(config: LogConfig) {
    this.config = config;
  }

  shouldLogDetailed(claim: Claim, decision: ClaimDecision): LogLevel {
    // Always log high-risk
    if (decision.reasonRules?.some(r => this.config.highRiskRules.has(r.ruleId))) {
      return LogLevel.FULL;
    }

    // Always log escalations (humans will review anyway)
    if (decision.action === 'escalate') {
      return LogLevel.FULL;
    }

    // Log denials at sample rate (1% of denies still creates audit trail)
    if (decision.action === 'deny') {
      return Math.random() < this.config.sampleRate
        ? LogLevel.SAMPLE
        : LogLevel.METRICS_ONLY;
    }

    // Approve = default (usually metrics only)
    return this.config.defaultLevel;
  }

  async recordClaim(
    claim: Claim,
    decision: ClaimDecision,
    toolCalls: ToolCall[],
    durationMs: number
  ): Promise<void> {
    const logLevel = this.shouldLogDetailed(claim, decision);

    // Always: update metrics
    this.updateMetrics(claim, decision, durationMs, toolCalls);

    // Conditional: detailed logging
    switch (logLevel) {
      case LogLevel.NONE:
        // Nothing beyond metrics
        break;

      case LogLevel.METRICS_ONLY:
        // Just increment counters (already done above)
        break;

      case LogLevel.SAMPLE:
        // Store in local buffer, flush periodically
        this.sampleBuffer.push({
          claimId: claim.id,
          claimNumber: claim.claimNumber,
          decision: decision.action,
          amount: claim.amount,
          toolCalls: toolCalls.map(t => ({
            toolId: t.toolId,
            durationMs: t.durationMs,
            costPoints: t.costPoints,
          })),
          totalDurationMs: durationMs,
          timestamp: new Date(),
        });

        if (this.sampleBuffer.length >= 1000) {
          await this.flushSampleBuffer();
        }
        break;

      case LogLevel.ANOMALY:
        // Check if decision is unexpected for this claim type
        const isAnomaly = await this.detectAnomaly(claim, decision);
        if (isAnomaly) {
          await this.logAnomaly(claim, decision, toolCalls);
        }
        break;

      case LogLevel.FULL:
        // Log everything (for high-risk, escalations, dev)
        await this.logFull(claim, decision, toolCalls, durationMs);
        break;
    }
  }

  private updateMetrics(
    claim: Claim,
    decision: ClaimDecision,
    durationMs: number,
    toolCalls: ToolCall[]
  ): void {
    this.metrics.claimsProcessed++;

    if (decision.action === 'approve') this.metrics.claimsApproved++;
    else if (decision.action === 'deny') this.metrics.claimsDenied++;
    else if (decision.action === 'escalate') this.metrics.claimsEscalated++;

    this.metrics.avgDecisionTimeMs =
      (this.metrics.avgDecisionTimeMs * (this.metrics.claimsProcessed - 1) +
        durationMs) /
      this.metrics.claimsProcessed;

    this.metrics.totalCostPoints += toolCalls.reduce((sum, t) => sum + t.costPoints, 0);
  }

  private async detectAnomaly(
    claim: Claim,
    decision: ClaimDecision
  ): Promise<boolean> {
    // Query historical similar claims and their approval rates
    const similar = await this.queryHistoricalBaseline(claim);
    if (!similar) return false;

    const { approvalRate, expectedAction } = similar;

    // Flag if this decision is unexpected
    if (decision.action === 'approve' && approvalRate < 0.8) {
      return true; // Usually denied, but we approved
    }
    if (decision.action === 'deny' && approvalRate > 0.8) {
      return true; // Usually approved, but we denied
    }

    return false;
  }

  private async queryHistoricalBaseline(
    claim: Claim
  ): Promise<{ approvalRate: number; expectedAction: string } | null> {
    // Query historical DB for similar claims
    // (This is a quick lookup, not full search)
    const query = `
      SELECT 
        COUNT(*) as total,
        SUM(CASE WHEN decision='approve' THEN 1 ELSE 0 END) as approved,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) as median_amount
      FROM claim_history
      WHERE 
        diagnosis_code = ? 
        AND procedure_code = ?
        AND service_date > NOW() - INTERVAL '90 days'
    `;

    // This returns fast (indexed columns, small result set)
    const result = await this.db.query(query, [
      claim.diagnosisCodes[0],
      claim.procedureCodes[0],
    ]);

    if (result.total < 10) return null; // Not enough history to be confident

    return {
      approvalRate: result.approved / result.total,
      expectedAction: result.approvalRate > 0.8 ? 'approve' : 'deny',
    };
  }

  private async logAnomaly(
    claim: Claim,
    decision: ClaimDecision,
    toolCalls: ToolCall[]
  ): Promise<void> {
    const record: AnomalyRecord = {
      claimId: claim.id,
      claimNumber: claim.claimNumber,
      reason: 'Unexpected decision for claim type',
      decision: decision.action,
      amount: claim.amount,
      toolCallIds: toolCalls.map(t => t.toolId),
      timestamp: new Date(),
    };

    this.anomalies.push(record);

    // Send to monitoring system immediately
    await this.sendToDatadog({
      type: 'anomaly',
      claimId: claim.id,
      decision: decision.action,
      severity: 'medium',
    });
  }

  private async logFull(
    claim: Claim,
    decision: ClaimDecision,
    toolCalls: ToolCall[],
    durationMs: number
  ): Promise<void> {
    // Send to centralized log (Kafka/Datadog/S3)
    const record = {
      timestamp: new Date().toISOString(),
      claimId: claim.id,
      claimNumber: claim.claimNumber,
      decision: decision.action,
      amount: claim.amount,
      diagnosisCodes: claim.diagnosisCodes,
      procedureCodes: claim.procedureCodes,
      toolCalls: toolCalls.map(t => ({
        toolId: t.toolId,
        input: t.input,
        output: t.output,
        durationMs: t.durationMs,
        costPoints: t.costPoints,
      })),
      totalDurationMs: durationMs,
      explanation: decision.explanation,
    };

    await this.sendToBackend(record);
  }

  private async flushSampleBuffer(): Promise<void> {
    if (this.sampleBuffer.length === 0) return;

    // Batch write sample logs (once per 1000 claims)
    await this.sendToBackend({
      type: 'sample_batch',
      timestamp: new Date().toISOString(),
      count: this.sampleBuffer.length,
      records: this.sampleBuffer,
    });

    this.sampleBuffer = [];
  }

  private async sendToBackend(record: any): Promise<void> {
    switch (this.config.storeBackend) {
      case 'kafka':
        // Send to Kafka topic (fire and forget)
        await this.kafka.produce('claims-logs', JSON.stringify(record));
        break;

      case 'datadog':
        // Send to Datadog (with sampling)
        await this.datadog.log(record);
        break;

      case 's3':
        // Batch to S3 (cheapest, ~minutes of latency)
        this.s3Buffer.push(record);
        if (this.s3Buffer.length >= 10000) {
          await this.flushToS3();
        }
        break;

      case 'memory':
        // Dev only
        this.inMemoryLogs.push(record);
        break;
    }
  }

  private async sendToDatadog(event: any): Promise<void> {
    // Send anomalies/high-risk events immediately (sync)
    await fetch('https://api.datadoghq.com/api/v2/events', {
      method: 'POST',
      headers: { 'DD-API-KEY': process.env.DATADOG_API_KEY },
      body: JSON.stringify(event),
    });
  }

  async getMetrics(): Promise<any> {
    return {
      ...this.metrics,
      anomaliesDetected: this.anomalies.length,
      sampledClaims: this.sampleBuffer.length,
    };
  }

  async getAnomalies(hours: number = 24): Promise<AnomalyRecord[]> {
    const cutoff = new Date(Date.now() - hours * 3600 * 1000);
    return this.anomalies.filter(a => a.timestamp > cutoff);
  }
}

interface DetailedLog {
  claimId: string;
  claimNumber: string;
  decision: string;
  amount: number;
  toolCalls: Array<{ toolId: string; durationMs: number; costPoints: number }>;
  totalDurationMs: number;
  timestamp: Date;
}

interface AnomalyRecord {
  claimId: string;
  claimNumber: string;
  reason: string;
  decision: string;
  amount: number;
  toolCallIds: string[];
  timestamp: Date;
}
```

## Configuration for Different Environments

```typescript
// config.ts

// PRODUCTION: Process 1M claims/day
export const productionConfig: LogConfig = {
  defaultLevel: LogLevel.METRICS_ONLY, // Approve = just count
  sampleRate: 0.01, // 1% of denies = ~300 detailed logs/day
  anomalyThreshold: 2, // Flag if >2% deviation from baseline
  highRiskRules: new Set([
    'appeal_override',
    'prior_auth_threshold',
    'coverage_limit_exceeded',
  ]),
  storeBackend: 'kafka', // Stream to Kafka, then S3 nightly
};

// STAGING: More visibility
export const stagingConfig: LogConfig = {
  defaultLevel: LogLevel.SAMPLE,
  sampleRate: 0.1, // 10% sampling
  anomalyThreshold: 1,
  highRiskRules: new Set([...productionConfig.highRiskRules]),
  storeBackend: 's3', // Direct to S3 (cheaper, OK with latency)
};

// DEVELOPMENT: Full debugging
export const devConfig: LogConfig = {
  defaultLevel: LogLevel.FULL, // Everything
  sampleRate: 1.0,
  anomalyThreshold: 0,
  highRiskRules: new Set(),
  storeBackend: 'memory', // Stay in RAM
};
```

## Storage Breakdown for 1M Claims/Day

```
Daily volume: 1,000,000 claims

LOG LEVEL: METRICS_ONLY (default, approved claims)
- Claims: ~800,000 approved
- Storage: ~100 bytes per count update
- Daily: ~80 MB (negligible)

LOG LEVEL: SAMPLE (1% of 200k denials)
- Claims: ~2,000 logged
- Storage: ~10 KB per claim
- Daily: ~20 MB (cheap)

LOG LEVEL: FULL (escalations, high-risk)
- Claims: ~5,000 escalated
- Tool calls per claim: avg 3
- Storage: ~5 KB per call = ~75 KB per claim
- Daily: ~375 MB (acceptable)

LOG LEVEL: ANOMALY (unexpected decisions)
- Claims: ~1,000 flagged
- Storage: ~2 KB per anomaly
- Daily: ~2 MB (trivial)

TOTAL DAILY: ~477 MB (vs. 50+ GB if logging everything)
MONTHLY: ~14 GB (1 SSD)
YEARLY: ~174 GB (fits in single S3 bucket)
```

## Real-Time Dashboards (What You Actually Watch)

```typescript
// dashboard.metrics.ts
import Prometheus from 'prom-client';

export class ProductionMetrics {
  // What you care about (not raw logs)
  
  private approvalRate = new Prometheus.Gauge({
    name: 'claims_approval_rate_percent',
    help: 'Percentage of claims approved (real-time)',
  });

  private avgDecisionTime = new Prometheus.Histogram({
    name: 'claims_decision_time_ms',
    help: 'Decision latency',
    buckets: [100, 200, 500, 1000],
  });

  private anomalyCount = new Prometheus.Counter({
    name: 'claims_anomalies_total',
    help: 'Claims with unexpected decisions',
    labelNames: ['type'],
  });

  private toolCallSuccess = new Prometheus.Counter({
    name: 'tool_calls_success_total',
    help: 'Successful tool calls',
    labelNames: ['toolId'],
  });

  private toolCallError = new Prometheus.Counter({
    name: 'tool_calls_error_total',
    help: 'Failed tool calls',
    labelNames: ['toolId', 'errorType'],
  });

  // These are updated in real-time; dashboard queries Prometheus
  // Historical data stays in S3 for audits/analysis
}

// What you query daily:
// - "What's my approval rate this hour?" (from Prometheus gauge)
// - "Which rules fired most?" (from aggregated metrics)
// - "Any anomalies last 24h?" (from anomaly log)
// - "Which tools are slow?" (from histogram buckets)

// What you query monthly/yearly:
// - "Appeal rate on AI-approved claims vs COBOL?" (S3 + athena)
// - "Which claim types have highest divergence?" (S3 + analysis)
```

## Audit Queries (When You Need Details)

```sql
-- Compliance audit: "Give me all denials for diagnosis E11 last month"
SELECT 
  claim_id, 
  claim_number, 
  decision, 
  amount, 
  rules_applied,
  timestamp
FROM claims_logs_s3  -- Parquet on S3
WHERE 
  service_date >= '2024-01-01' 
  AND service_date < '2024-02-01'
  AND diagnosis_code = 'E11%'
  AND decision = 'deny'
LIMIT 1000;

-- This query runs on parquet files in S3 (cheap, ~seconds)
-- Not on real-time logs (would be slow/expensive)

-- Appeal investigation: "Which tool calls preceded this appeal?"
SELECT 
  tool_calls
FROM claims_logs_s3
WHERE claim_id = 'claim-xyz'
  AND timestamp = '2024-01-15 14:32:00'
LIMIT 1;
-- Full detail available because high-risk claims are logged in full
```

## Implementation in Agent Loop

```typescript
// agent-with-logging.ts
export class ClaimsAgentWithLogging {
  private logger: AdaptiveLogger;

  async processClaim(
    claim: Claim,
    context: any
  ): Promise<{ decision: ClaimDecision; auditTrail: any }> {
    const startTime = Date.now();
    const toolCalls: ToolCall[] = [];

    // ... agent processes claim, makes tool calls ...
    // Each tool call appended to toolCalls[]

    const decision = { /* ... */ };
    const durationMs = Date.now() - startTime;

    // Single call: adaptive logger decides what to store
    await this.logger.recordClaim(claim, decision, toolCalls, durationMs);

    return { decision, auditTrail: { toolCount: toolCalls.length } };
  }
}
```

---

## Summary: What Gets Stored Where

| Log Type | Volume | Storage | Access Pattern |
|----------|--------|---------|-----------------|
| Metrics (counts, rates) | ~100 bytes per 1000 claims | Prometheus (memory) | Real-time dashboards |
| Sampled details (1% of denies) | 20 MB/day | Kafka → S3 parquet | Ad-hoc queries, appeals |
| Full logs (escalations, high-risk) | 375 MB/day | S3 parquet | Compliance audits, investigations |
| Anomalies | 2 MB/day | Datadog alerts | Real-time monitoring |
| **TOTAL** | **~400 MB/day** | **~12 GB/month** | **Tiered by use case** |

This way you have:
- **Real-time visibility** (metrics dashboards)
- **Full audit trail** (S3, queryable, cheap)
- **No operational overhead** (99% of claims = just metrics)
- **Fast anomaly detection** (streaming to Datadog)
- **Compliance ready** (all decisions traceable, especially denies)
