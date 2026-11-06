# Claims Processing Harness in TypeScript

Modernizing a COBOL claims system by extracting rules into a TypeScript harness, validating against legacy behavior, and migrating incrementally.

## Core Architecture

```typescript
// claims.types.ts
export interface Claim {
  id: string;
  claimNumber: string;
  patientId: string;
  providerId: string;
  amount: number;
  diagnosisCodes: string[]; // ICD-10
  procedureCodes: string[]; // CPT
  serviceDate: Date;
  submissionDate: Date;
}

export interface ClaimDecision {
  claimId: string;
  action: 'approve' | 'deny' | 'escalate';
  amount?: number;
  reasonRules: RuleTrace[];
  explanation: string;
  requiresPriorAuth: boolean;
  reviewerQueue?: 'medical' | 'senior' | 'none';
  timestamp: Date;
}

export interface RuleTrace {
  ruleId: string;
  ruleName: string;
  matched: boolean;
  input: Record<string, any>;
  output: any;
  section: string; // "COBOL line 4521" or "CMS 2024"
}

export interface ComparisonResult {
  claimId: string;
  cobolDecision: ClaimDecision;
  aiDecision: ClaimDecision;
  match: boolean;
  divergences: string[];
  migrationSafe: boolean;
}
```

## Rule Engine (The Harness)

```typescript
// rules.engine.ts
import { Claim, ClaimDecision, RuleTrace } from './claims.types';

type RuleCondition = (claim: Claim, context: RuleContext) => boolean;
type RuleAction = (claim: Claim, context: RuleContext) => Partial<ClaimDecision>;

interface Rule {
  id: string;
  name: string;
  description: string;
  section: string; // where in COBOL or regulations
  priority: number; // 0-100, higher wins
  condition: RuleCondition;
  action: RuleAction;
  testCases?: { claim: Claim; expectedMatch: boolean }[];
}

interface RuleContext {
  eligibility: EligibilityData;
  policyLimits: Map<string, number>;
  priorApprovals: Claim[];
  appeals: Appeal[];
}

export class RulesEngine {
  private rules: Map<string, Rule> = new Map();
  private traces: RuleTrace[] = [];

  register(rule: Rule): void {
    this.rules.set(rule.id, rule);
  }

  async processClaimWithTracing(
    claim: Claim,
    context: RuleContext
  ): Promise<{ decision: ClaimDecision; traces: RuleTrace[] }> {
    this.traces = [];
    const decision: Partial<ClaimDecision> = {
      claimId: claim.id,
      action: 'approve',
      amount: claim.amount,
      reasonRules: [],
      timestamp: new Date(),
    };

    // Sort by priority descending
    const sortedRules = Array.from(this.rules.values())
      .sort((a, b) => b.priority - a.priority);

    for (const rule of sortedRules) {
      const matched = rule.condition(claim, context);
      const trace: RuleTrace = {
        ruleId: rule.id,
        ruleName: rule.name,
        matched,
        input: { claimAmount: claim.amount, codes: claim.procedureCodes },
        output: matched ? rule.action(claim, context) : null,
        section: rule.section,
      };
      this.traces.push(trace);

      if (matched) {
        const ruleOutput = rule.action(claim, context);
        Object.assign(decision, ruleOutput);
        decision.reasonRules!.push(trace);

        // Stop if decision is made (deny/escalate)
        if (decision.action !== 'approve') break;
      }
    }

    return {
      decision: decision as ClaimDecision,
      traces: this.traces,
    };
  }

  getTraces(): RuleTrace[] {
    return this.traces;
  }
}

// Register the rules
export const createHarness = (): RulesEngine => {
  const engine = new RulesEngine();

  // RULE 1: Bundling (extracted from COBOL line 4521)
  engine.register({
    id: 'bundling_e_and_m',
    name: 'E&M Bundling Rule 2024',
    description: 'Combine office visits for diabetes on same date',
    section: 'COBOL line 4521, CMS bundling 2024',
    priority: 90,
    condition: (claim, context) => {
      const hasDiabetes = claim.diagnosisCodes.some(c => c.startsWith('E11'));
      const hasEandM = claim.procedureCodes.some(c =>
        ['99214', '99215', '99203', '99204'].includes(c)
      );
      return hasDiabetes && hasEandM;
    },
    action: (claim, context) => ({
      amount: claim.amount * 0.85, // bundling reduction
      explanation: 'Bundled office visit (diabetes E&M)',
    }),
    testCases: [
      {
        claim: {
          id: 'test1',
          claimNumber: '123',
          patientId: 'P1',
          providerId: 'PR1',
          amount: 500,
          diagnosisCodes: ['E11.9'],
          procedureCodes: ['99214'],
          serviceDate: new Date('2024-01-15'),
          submissionDate: new Date('2024-01-20'),
        },
        expectedMatch: true,
      },
    ],
  });

  // RULE 2: Prior auth threshold (extracted from COBOL line 8823)
  engine.register({
    id: 'prior_auth_threshold',
    name: 'Prior Auth Required > $25K',
    description: 'Any claim over 25000 needs prior authorization',
    section: 'COBOL line 8823',
    priority: 95,
    condition: (claim, context) => claim.amount > 25000,
    action: (claim, context) => ({
      action: 'escalate',
      requiresPriorAuth: true,
      reviewerQueue: 'medical',
      explanation: 'Claim amount exceeds prior auth threshold ($25K)',
    }),
  });

  // RULE 3: Appeal override (new best practice, not in COBOL)
  engine.register({
    id: 'appeal_override',
    name: 'Appeal Always Goes to Senior Review',
    description: 'If patient appealed, escalate to senior reviewer',
    section: 'NEW: Best practice for fairness',
    priority: 100,
    condition: (claim, context) => {
      return context.appeals.some(a => a.claimId === claim.id && a.status === 'pending');
    },
    action: (claim, context) => ({
      action: 'escalate',
      reviewerQueue: 'senior',
      explanation: 'Patient appeal filed; requires senior reviewer',
    }),
  });

  // RULE 4: Coverage limit (extracted from COBOL line 6102)
  engine.register({
    id: 'coverage_limit_pt',
    name: 'PT Visit Limit (30/year)',
    description: 'Physical therapy capped at 30 visits per year',
    section: 'COBOL line 6102',
    priority: 85,
    condition: (claim, context) => {
      const isPT = claim.procedureCodes.some(c =>
        ['97161', '97162', '97163'].includes(c)
      );
      if (!isPT) return false;

      const ptCount = context.priorApprovals.filter(c =>
        c.procedureCodes.some(p => ['97161', '97162', '97163'].includes(p))
      ).length;

      return ptCount >= 30;
    },
    action: (claim, context) => ({
      action: 'deny',
      explanation: 'Patient has reached annual PT visit limit (30)',
    }),
  });

  return engine;
};
```

## Comparison Layer (Legacy vs. New)

```typescript
// cobol.adapter.ts
import { Claim, ClaimDecision } from './claims.types';

export interface COBOLResponse {
  claimNumber: string;
  decision: 'A' | 'D' | 'S'; // Approve, Deny, Suspend
  amount: number;
  codes: string[]; // Decision codes from COBOL
  timestamp: string;
}

export class COBOLAdapter {
  private endpoint: string;
  private timeout: number = 30000;

  constructor(cobolSystemUrl: string) {
    this.endpoint = cobolSystemUrl;
  }

  async processClaimLegacy(claim: Claim): Promise<ClaimDecision> {
    // Call COBOL system (via API wrapper or message queue)
    const cobolResp = await fetch(`${this.endpoint}/process`, {
      method: 'POST',
      body: JSON.stringify({
        claim_number: claim.claimNumber,
        patient_id: claim.patientId,
        amount: claim.amount,
        diag_codes: claim.diagnosisCodes,
        proc_codes: claim.procedureCodes,
        service_date: claim.serviceDate.toISOString(),
      }),
      timeout: this.timeout,
    }).then(r => r.json()) as COBOLResponse;

    return {
      claimId: claim.id,
      action:
        cobolResp.decision === 'A'
          ? 'approve'
          : cobolResp.decision === 'D'
            ? 'deny'
            : 'escalate',
      amount: cobolResp.amount,
      reasonRules: [], // COBOL doesn't give us the rules (black box)
      explanation: `COBOL decision codes: ${cobolResp.codes.join(', ')}`,
      requiresPriorAuth: cobolResp.codes.includes('PA'),
      timestamp: new Date(cobolResp.timestamp),
    };
  }
}

// migration.validator.ts
import { ClaimDecision, ComparisonResult } from './claims.types';

export class MigrationValidator {
  static compareDecisions(
    claim: Claim,
    cobolDecision: ClaimDecision,
    aiDecision: ClaimDecision
  ): ComparisonResult {
    const divergences: string[] = [];

    if (cobolDecision.action !== aiDecision.action) {
      divergences.push(
        `Decision mismatch: COBOL=${cobolDecision.action}, AI=${aiDecision.action}`
      );
    }

    if (cobolDecision.amount !== aiDecision.amount) {
      divergences.push(
        `Amount mismatch: COBOL=${cobolDecision.amount}, AI=${aiDecision.amount}`
      );
    }

    return {
      claimId: claim.id,
      cobolDecision,
      aiDecision,
      match: divergences.length === 0,
      divergences,
      migrationSafe:
        divergences.length === 0 &&
        aiDecision.reasonRules.length > 0, // Must have traceable rules
    };
  }

  static async validateBatch(
    claims: Claim[],
    cobolAdapter: COBOLAdapter,
    aiEngine: RulesEngine,
    context: RuleContext
  ): Promise<{
    total: number;
    matched: number;
    failures: ComparisonResult[];
    matchRate: number;
  }> {
    const results: ComparisonResult[] = [];

    for (const claim of claims) {
      const [cobolDecision, { decision: aiDecision }] = await Promise.all([
        cobolAdapter.processClaimLegacy(claim),
        aiEngine.processClaimWithTracing(claim, context),
      ]);

      const comparison = MigrationValidator.compareDecisions(
        claim,
        cobolDecision,
        aiDecision
      );
      results.push(comparison);
    }

    const failures = results.filter(r => !r.match);
    return {
      total: claims.length,
      matched: results.length - failures.length,
      failures,
      matchRate: ((results.length - failures.length) / claims.length) * 100,
    };
  }
}
```

## Testing & Regression Suite

```typescript
// harness.test.ts
import { describe, it, expect } from 'vitest';
import { createHarness } from './rules.engine';
import { Claim, ClaimDecision, RuleContext } from './claims.types';

describe('Claims Processing Harness', () => {
  const harness = createHarness();

  const mockContext: RuleContext = {
    eligibility: { patientActive: true, planType: 'PPO' },
    policyLimits: new Map([['pt_visits', 30]]),
    priorApprovals: [],
    appeals: [],
  };

  describe('Rule: Bundling E&M', () => {
    it('should bundle diabetes office visits', async () => {
      const claim: Claim = {
        id: 'claim1',
        claimNumber: 'CLM001',
        patientId: 'P123',
        providerId: 'PR456',
        amount: 500,
        diagnosisCodes: ['E11.9'], // Type 2 diabetes
        procedureCodes: ['99214'], // Office visit
        serviceDate: new Date('2024-01-15'),
        submissionDate: new Date('2024-01-20'),
      };

      const { decision, traces } = await harness.processClaimWithTracing(
        claim,
        mockContext
      );

      expect(decision.action).toBe('approve');
      expect(decision.amount).toBe(500 * 0.85); // Bundling reduction
      expect(traces).toContainEqual(
        expect.objectContaining({ ruleId: 'bundling_e_and_m', matched: true })
      );
    });
  });

  describe('Rule: Prior Auth Threshold', () => {
    it('should escalate claims > $25K', async () => {
      const claim: Claim = {
        id: 'claim2',
        claimNumber: 'CLM002',
        patientId: 'P123',
        providerId: 'PR456',
        amount: 50000,
        diagnosisCodes: ['I10'],
        procedureCodes: ['92004'], // Comprehensive eye exam
        serviceDate: new Date('2024-01-15'),
        submissionDate: new Date('2024-01-20'),
      };

      const { decision } = await harness.processClaimWithTracing(
        claim,
        mockContext
      );

      expect(decision.action).toBe('escalate');
      expect(decision.requiresPriorAuth).toBe(true);
      expect(decision.reviewerQueue).toBe('medical');
    });
  });

  describe('Rule: PT Visit Limit', () => {
    it('should deny PT when limit reached', async () => {
      // Mock 30 prior PT approvals
      const priorPTClaims: Claim[] = Array.from({ length: 30 }, (_, i) => ({
        id: `pt${i}`,
        claimNumber: `CLM-PT-${i}`,
        patientId: 'P123',
        providerId: 'PR456',
        amount: 150,
        diagnosisCodes: ['M79.3'],
        procedureCodes: ['97161'], // PT evaluation
        serviceDate: new Date(`2024-${String((i % 12) + 1).padStart(2, '0')}-01`),
        submissionDate: new Date(),
      }));

      const contextWithPT: RuleContext = {
        ...mockContext,
        priorApprovals: priorPTClaims,
      };

      const claim: Claim = {
        id: 'claim3',
        claimNumber: 'CLM003',
        patientId: 'P123',
        providerId: 'PR456',
        amount: 150,
        diagnosisCodes: ['M79.3'],
        procedureCodes: ['97161'],
        serviceDate: new Date('2024-12-15'),
        submissionDate: new Date('2024-12-20'),
      };

      const { decision } = await harness.processClaimWithTracing(
        claim,
        contextWithPT
      );

      expect(decision.action).toBe('deny');
      expect(decision.explanation).toContain('PT visit limit');
    });
  });
});
```

## Migration Orchestrator

```typescript
// migration.orchestrator.ts
import { RulesEngine } from './rules.engine';
import { COBOLAdapter } from './cobol.adapter';
import { MigrationValidator, ComparisonResult } from './cobol.adapter';
import { Claim } from './claims.types';

export interface MigrationPhase {
  phase: number;
  percentageOfTraffic: number;
  validateBefore: boolean;
  description: string;
}

export class MigrationOrchestrator {
  private phases: MigrationPhase[] = [
    { phase: 1, percentageOfTraffic: 1, validateBefore: true, description: 'Canary: 1% traffic' },
    { phase: 2, percentageOfTraffic: 5, validateBefore: false, description: 'Ramp: 5% traffic' },
    { phase: 3, percentageOfTraffic: 25, validateBefore: false, description: 'Ramp: 25% traffic' },
    { phase: 4, percentageOfTraffic: 100, validateBefore: false, description: 'Full cutover' },
  ];

  constructor(
    private aiEngine: RulesEngine,
    private cobolAdapter: COBOLAdapter
  ) {}

  async runPhase(
    phaseNum: number,
    claims: Claim[],
    context: any
  ): Promise<{ phaseResult: MigrationPhase; validation: any; readyForNext: boolean }> {
    const phase = this.phases[phaseNum - 1];
    if (!phase) throw new Error(`Phase ${phaseNum} not found`);

    console.log(`Starting ${phase.description}`);

    // Select subset for this phase
    const subset = claims.slice(
      0,
      Math.ceil((claims.length * phase.percentageOfTraffic) / 100)
    );

    if (phase.validateBefore) {
      console.log(`Validating ${subset.length} claims against COBOL...`);
      const validation = await MigrationValidator.validateBatch(
        subset,
        this.cobolAdapter,
        this.aiEngine,
        context
      );

      console.log(
        `Match rate: ${validation.matchRate.toFixed(2)}% (${validation.matched}/${validation.total})`
      );

      if (validation.matchRate < 99.0) {
        console.error(
          `Match rate below 99%. Failures:\n${validation.failures
            .slice(0, 5)
            .map(f => `  ${f.claimId}: ${f.divergences.join('; ')}`)
            .join('\n')}`
        );
        return {
          phaseResult: phase,
          validation,
          readyForNext: false,
        };
      }
    }

    return {
      phaseResult: phase,
      validation: null,
      readyForNext: true,
    };
  }

  async runFullMigration(allClaims: Claim[], context: any): Promise<void> {
    for (const phase of this.phases) {
      const result = await this.runPhase(
        phase.phase,
        allClaims,
        context
      );

      if (!result.readyForNext) {
        console.error(`❌ Phase ${phase.phase} failed validation. Stopping migration.`);
        process.exit(1);
      }

      console.log(`✅ Phase ${phase.phase} complete. Ready to proceed.`);
    }

    console.log('\n✅ Full migration complete. COBOL system is now read-only.');
  }
}
```

## Daily Monitoring Dashboard

```typescript
// monitoring.ts
import prom from 'prom-client';

export class ClaimsMetrics {
  private approvalRate = new prom.Gauge({
    name: 'claims_approval_rate',
    help: 'Percentage of claims approved',
    labelNames: ['system'], // 'cobol' vs 'ai'
  });

  private decisionTime = new prom.Histogram({
    name: 'claims_decision_time_ms',
    help: 'Time to process a claim (ms)',
    labelNames: ['system'],
    buckets: [50, 100, 500, 1000, 5000],
  });

  private appealRate = new prom.Gauge({
    name: 'claims_appeal_rate',
    help: 'Percentage of approved claims that are appealed',
    labelNames: ['system'],
  });

  private ruleMatches = new prom.Counter({
    name: 'claims_rule_matches_total',
    help: 'Count of times each rule matched',
    labelNames: ['rule_id'],
  });

  recordDecision(
    system: 'cobol' | 'ai',
    approved: boolean,
    timeMs: number,
    matchedRules: string[]
  ) {
    this.approvalRate.set({ system }, approved ? 1 : 0);
    this.decisionTime.observe({ system }, timeMs);
    matchedRules.forEach(rule => {
      this.ruleMatches.inc({ rule_id: rule });
    });
  }

  recordAppeal(claimId: string, system: 'cobol' | 'ai') {
    this.appealRate.inc({ system });
  }

  async exportMetrics(): Promise<string> {
    return prom.register.metrics();
  }
}
```

## Usage Example

```typescript
// index.ts
import { createHarness } from './rules.engine';
import { COBOLAdapter } from './cobol.adapter';
import { MigrationOrchestrator } from './migration.orchestrator';
import { ClaimsMetrics } from './monitoring';

async function main() {
  const harness = createHarness();
  const cobol = new COBOLAdapter('http://mainframe.legacy.internal:8080');
  const orchestrator = new MigrationOrchestrator(harness, cobol);
  const metrics = new ClaimsMetrics();

  // Load historical claims for validation
  const historicalClaims = await loadClaimsFromDatabase();

  // Run migration
  await orchestrator.runFullMigration(historicalClaims, {
    eligibility: { /* ... */ },
    policyLimits: new Map([['pt_visits', 30]]),
    priorApprovals: [],
    appeals: [],
  });

  console.log('Migration complete. Metrics:');
  console.log(await metrics.exportMetrics());
}

main().catch(console.error);
```

---

## Key Benefits

1. **Traceable decisions** — every approve/deny/escalate cites the rule and source
2. **Testable rules** — each rule has regression tests; new rules must pass before deployment
3. **Safe migration** — validation against COBOL before moving traffic
4. **Rule evolution** — divergences reveal missing rules; you codify them and re-test
5. **Compliance** — auditors see exactly which rule made each decision and where it came from
