# Claims Harness: Agent-Driven Architecture

The harness constrains agents to safely call both legacy COBOL and new TypeScript functions. Functions are tools; the harness determines which tools agents can use.

## Core Model

```typescript
// function-catalog.ts
export interface HarnessFunction {
  id: string;
  name: string;
  description: string;
  category: 'cobol' | 'typescript' | 'hybrid';
  source: string; // "COBOL line 4521" or "ts/bundling.ts:42"
  
  // Agent access control
  requiresAuth: boolean;
  allowedRoles: ('agent' | 'human' | 'system')[];
  maxCallsPerClaim: number;
  timeout: number; // ms
  
  // Function signature for agent
  params: Record<string, { type: string; required: boolean; description: string }>;
  returns: Record<string, { type: string; description: string }>;
  
  // When agent can call this
  preconditions: (claim: Claim, context: Context) => boolean;
  
  // Monitoring
  costPoints: number; // Budget system
  sideEffects: 'none' | 'read-only' | 'modifies-claim' | 'external-call';
}

export interface AgentToolkit {
  claimId: string;
  availableTools: HarnessFunction[];
  callLog: ToolCall[];
  remainingBudget: number; // cost points
}

export interface ToolCall {
  toolId: string;
  claimId: string;
  input: Record<string, any>;
  output: any;
  error?: string;
  durationMs: number;
  timestamp: Date;
  costPoints: number;
  agentId: string;
}
```

## Function Registry (Catalog of All Available Functions)

```typescript
// function-registry.ts
import { HarnessFunction, AgentToolkit, ToolCall } from './function-catalog';

export class FunctionRegistry {
  private functions: Map<string, HarnessFunction> = new Map();
  private callLog: ToolCall[] = [];

  register(fn: HarnessFunction): void {
    this.functions.set(fn.id, fn);
  }

  // Legacy COBOL functions
  registerCOBOLFunctions(): void {
    this.register({
      id: 'cobol_check_eligibility',
      name: 'Check Patient Eligibility',
      description: 'Query COBOL eligibility system for patient coverage status',
      category: 'cobol',
      source: 'COBOL module ELIGL, line 2340',
      requiresAuth: true,
      allowedRoles: ['agent', 'human'],
      maxCallsPerClaim: 1,
      timeout: 5000,
      params: {
        patientId: { type: 'string', required: true, description: 'Patient identifier' },
        serviceDate: { type: 'date', required: true, description: 'Service date for coverage check' },
      },
      returns: {
        eligible: { type: 'boolean', description: 'Is patient eligible?' },
        planType: { type: 'string', description: 'PPO, HMO, etc.' },
        effectiveDate: { type: 'date', description: 'When coverage starts' },
        deductible: { type: 'number', description: 'Remaining deductible' },
      },
      preconditions: (claim, context) => context.eligibility !== null,
      costPoints: 10,
      sideEffects: 'read-only',
    });

    this.register({
      id: 'cobol_apply_bundling',
      name: 'Apply Bundling Rules',
      description: 'COBOL bundling logic for multiple procedures on same date',
      category: 'cobol',
      source: 'COBOL module BUNDLE, line 4521',
      requiresAuth: true,
      allowedRoles: ['agent'],
      maxCallsPerClaim: 1,
      timeout: 3000,
      params: {
        claimAmount: { type: 'number', required: true, description: 'Original claim amount' },
        procedureCodes: { type: 'string[]', required: true, description: 'CPT codes' },
        diagnosisCodes: { type: 'string[]', required: true, description: 'ICD-10 codes' },
      },
      returns: {
        adjustedAmount: { type: 'number', description: 'After bundling adjustment' },
        reductionPercent: { type: 'number', description: 'Percentage reduced' },
        bundlingCode: { type: 'string', description: 'Internal bundling classification' },
      },
      preconditions: (claim, context) => claim.procedureCodes.length > 1,
      costPoints: 15,
      sideEffects: 'read-only',
    });

    this.register({
      id: 'cobol_lookup_policy_rule',
      name: 'Lookup Policy Coverage Rule',
      description: 'Query COBOL policy database for specific coverage rule',
      category: 'cobol',
      source: 'COBOL module POLICY, line 6102',
      requiresAuth: true,
      allowedRoles: ['agent'],
      maxCallsPerClaim: 5, // Can call multiple times for different rules
      timeout: 2000,
      params: {
        ruleCode: { type: 'string', required: true, description: 'Policy rule identifier' },
        planType: { type: 'string', required: true, description: 'PPO, HMO, etc.' },
      },
      returns: {
        exists: { type: 'boolean', description: 'Does rule apply to this plan?' },
        limit: { type: 'number', description: 'Annual/visit limit if applicable' },
        requiresAuth: { type: 'boolean', description: 'Prior auth needed?' },
        copay: { type: 'number', description: 'Patient copay amount' },
      },
      preconditions: (claim, context) => context.eligibility?.planType !== undefined,
      costPoints: 5,
      sideEffects: 'read-only',
    });

    this.register({
      id: 'cobol_store_decision',
      name: 'Store Decision in COBOL',
      description: 'Record claim decision in legacy system for audit trail',
      category: 'cobol',
      source: 'COBOL module AUDIT, line 8823',
      requiresAuth: true,
      allowedRoles: ['system'], // Only system, not agents
      maxCallsPerClaim: 1,
      timeout: 5000,
      params: {
        claimNumber: { type: 'string', required: true, description: 'Claim identifier' },
        decision: { type: 'string', required: true, description: 'approve/deny/escalate' },
        amount: { type: 'number', required: true, description: 'Approved amount' },
        rules: { type: 'string[]', required: true, description: 'Rules applied' },
      },
      returns: {
        recordId: { type: 'string', description: 'COBOL audit record ID' },
        timestamp: { type: 'date', description: 'When stored' },
      },
      preconditions: (claim, context) => claim.id !== undefined,
      costPoints: 20,
      sideEffects: 'modifies-claim',
    });
  }

  // New TypeScript functions (modern replacements)
  registerTypeScriptFunctions(): void {
    this.register({
      id: 'ts_validate_codes',
      name: 'Validate Medical Codes',
      description: 'Validate ICD-10 and CPT codes against current CMS definitions',
      category: 'typescript',
      source: 'ts/validation/codes.ts:142',
      requiresAuth: false,
      allowedRoles: ['agent', 'human'],
      maxCallsPerClaim: 1,
      timeout: 1000,
      params: {
        diagnosisCodes: { type: 'string[]', required: true, description: 'ICD-10 codes to validate' },
        procedureCodes: { type: 'string[]', required: true, description: 'CPT codes to validate' },
      },
      returns: {
        valid: { type: 'boolean', description: 'All codes valid?' },
        invalidCodes: { type: 'string[]', description: 'Any codes not found in CMS database' },
        warnings: { type: 'string[]', description: 'Unusual code combinations' },
      },
      preconditions: (claim, context) => true,
      costPoints: 1,
      sideEffects: 'none',
    });

    this.register({
      id: 'ts_similarity_search',
      name: 'Find Similar Claims',
      description: 'Semantic search for historically similar claims and outcomes',
      category: 'typescript',
      source: 'ts/analytics/similarity.ts:67',
      requiresAuth: true,
      allowedRoles: ['agent'],
      maxCallsPerClaim: 1,
      timeout: 3000,
      params: {
        claimAmount: { type: 'number', required: true, description: 'Claim amount' },
        diagnosisCodes: { type: 'string[]', required: true, description: 'ICD-10 codes' },
        procedureCodes: { type: 'string[]', required: true, description: 'CPT codes' },
      },
      returns: {
        similarClaims: {
          type: 'object[]',
          description: 'Top 10 similar historical claims with outcomes',
        },
        approvalRate: { type: 'number', description: 'What % were approved?' },
        avgApprovedAmount: { type: 'number', description: 'Average approved amount' },
      },
      preconditions: (claim, context) => context.analytics !== undefined,
      costPoints: 25, // More expensive, needs DB search
      sideEffects: 'read-only',
    });

    this.register({
      id: 'ts_apply_new_bundling',
      name: 'Apply 2024 Bundling Rules (New)',
      description: 'TypeScript implementation of new CMS 2024 bundling rules',
      category: 'typescript',
      source: 'ts/rules/bundling2024.ts:12',
      requiresAuth: true,
      allowedRoles: ['agent'],
      maxCallsPerClaim: 1,
      timeout: 1000,
      params: {
        claimAmount: { type: 'number', required: true, description: 'Original claim amount' },
        procedureCodes: { type: 'string[]', required: true, description: 'CPT codes' },
        diagnosisCodes: { type: 'string[]', required: true, description: 'ICD-10 codes' },
      },
      returns: {
        adjustedAmount: { type: 'number', description: 'After bundling' },
        appliedRules: { type: 'string[]', description: 'Which bundling rules matched' },
        confidence: { type: 'number', description: '0-100, how certain?' },
      },
      preconditions: (claim, context) => true,
      costPoints: 5,
      sideEffects: 'read-only',
    });

    this.register({
      id: 'ts_flag_high_risk',
      name: 'ML Risk Scoring',
      description: 'ML model scores claim for fraud/error risk',
      category: 'typescript',
      source: 'ts/ml/risk-scorer.ts:88',
      requiresAuth: true,
      allowedRoles: ['agent'],
      maxCallsPerClaim: 1,
      timeout: 2000,
      params: {
        claimAmount: { type: 'number', required: true, description: 'Claim amount' },
        procedureCodes: { type: 'string[]', required: true, description: 'CPT codes' },
        providerHistory: { type: 'object', required: true, description: 'Provider claims history' },
      },
      returns: {
        riskScore: { type: 'number', description: '0-100, higher = more risky' },
        riskFactors: { type: 'string[]', description: 'Why is it risky?' },
        recommendedAction: {
          type: 'string',
          description: 'approve / review / deny',
        },
      },
      preconditions: (claim, context) => context.providerHistory !== undefined,
      costPoints: 30, // ML models are expensive
      sideEffects: 'none',
    });

    this.register({
      id: 'ts_generate_explanation',
      name: 'Generate Human-Readable Explanation',
      description: 'Convert decision rules into plain-English explanation for patient/provider',
      category: 'typescript',
      source: 'ts/reporting/explanation.ts:45',
      requiresAuth: false,
      allowedRoles: ['agent', 'human', 'system'],
      maxCallsPerClaim: 1,
      timeout: 2000,
      params: {
        decision: { type: 'string', required: true, description: 'approve/deny/escalate' },
        appliedRules: { type: 'string[]', required: true, description: 'Rule IDs that led to decision' },
        claimDetails: { type: 'object', required: true, description: 'Claim context' },
      },
      returns: {
        explanation: {
          type: 'string',
          description: 'Plain-English explanation (1-2 paragraphs)',
        },
        nextSteps: {
          type: 'string',
          description: 'What happens now? Appeal instructions if denied.',
        },
      },
      preconditions: (claim, context) => true,
      costPoints: 2,
      sideEffects: 'none',
    });
  }

  // Build toolkit for an agent processing a claim
  buildToolkit(claim: Claim, context: Context, agentId: string): AgentToolkit {
    const available: HarnessFunction[] = [];

    for (const fn of this.functions.values()) {
      // Check preconditions
      if (!fn.preconditions(claim, context)) continue;

      // Check agent role (agent can call everything except 'system' role only)
      if (!fn.allowedRoles.includes('agent')) continue;

      available.push(fn);
    }

    return {
      claimId: claim.id,
      availableTools: available,
      callLog: [],
      remainingBudget: 100, // cost points per claim
    };
  }

  // Agent calls a function
  async callFunction(
    toolId: string,
    input: Record<string, any>,
    toolkit: AgentToolkit,
    agentId: string
  ): Promise<{ output: any; toolCall: ToolCall }> {
    const fn = this.functions.get(toolId);
    if (!fn) throw new Error(`Function not found: ${toolId}`);

    if (!toolkit.availableTools.some(t => t.id === toolId)) {
      throw new Error(`Function not available for this claim: ${toolId}`);
    }

    if (toolkit.remainingBudget < fn.costPoints) {
      throw new Error(
        `Insufficient cost budget. Need ${fn.costPoints}, have ${toolkit.remainingBudget}`
      );
    }

    const startTime = Date.now();
    let output: any;
    let error: string | undefined;

    try {
      // Dispatch to COBOL or TypeScript implementation
      if (fn.category === 'cobol') {
        output = await this.callCOBOLFunction(toolId, input);
      } else {
        output = await this.callTypeScriptFunction(toolId, input);
      }
    } catch (err) {
      error = (err as Error).message;
      throw err;
    } finally {
      const durationMs = Date.now() - startTime;

      const toolCall: ToolCall = {
        toolId,
        claimId: toolkit.claimId,
        input,
        output,
        error,
        durationMs,
        timestamp: new Date(),
        costPoints: fn.costPoints,
        agentId,
      };

      toolkit.callLog.push(toolCall);
      toolkit.remainingBudget -= fn.costPoints;
      this.callLog.push(toolCall);

      return { output, toolCall };
    }
  }

  private async callCOBOLFunction(
    toolId: string,
    input: Record<string, any>
  ): Promise<any> {
    // Call legacy COBOL via HTTP adapter or message queue
    const response = await fetch('http://cobol-gateway/call', {
      method: 'POST',
      body: JSON.stringify({ function: toolId, params: input }),
    }).then(r => r.json());

    return response.result;
  }

  private async callTypeScriptFunction(
    toolId: string,
    input: Record<string, any>
  ): Promise<any> {
    // Direct TypeScript function call
    const handlers: Record<string, (input: any) => Promise<any>> = {
      ts_validate_codes: async (inp) => {
        // Import and call validation logic
        const { validateCodes } = await import('ts/validation/codes');
        return validateCodes(inp.diagnosisCodes, inp.procedureCodes);
      },
      ts_similarity_search: async (inp) => {
        const { findSimilarClaims } = await import('ts/analytics/similarity');
        return findSimilarClaims(inp.claimAmount, inp.diagnosisCodes, inp.procedureCodes);
      },
      ts_apply_new_bundling: async (inp) => {
        const { applyBundling2024 } = await import('ts/rules/bundling2024');
        return applyBundling2024(inp.claimAmount, inp.procedureCodes, inp.diagnosisCodes);
      },
      ts_flag_high_risk: async (inp) => {
        const { scoreRisk } = await import('ts/ml/risk-scorer');
        return scoreRisk(inp.claimAmount, inp.procedureCodes, inp.providerHistory);
      },
      ts_generate_explanation: async (inp) => {
        const { generateExplanation } = await import('ts/reporting/explanation');
        return generateExplanation(inp.decision, inp.appliedRules, inp.claimDetails);
      },
    };

    const handler = handlers[toolId];
    if (!handler) throw new Error(`No handler for ${toolId}`);

    return handler(input);
  }

  getCallLog(): ToolCall[] {
    return this.callLog;
  }

  async generateAuditReport(claimId: string): Promise<{
    claimId: string;
    functionsCalled: ToolCall[];
    totalCost: number;
    duration: number;
  }> {
    const calls = this.callLog.filter(c => c.claimId === claimId);
    const totalCost = calls.reduce((sum, c) => sum + c.costPoints, 0);
    const totalDuration = calls.reduce((sum, c) => sum + c.durationMs, 0);

    return {
      claimId,
      functionsCalled: calls,
      totalCost,
      duration: totalDuration,
    };
  }
}

// Initialize registry
export const initializeRegistry = (): FunctionRegistry => {
  const registry = new FunctionRegistry();
  registry.registerCOBOLFunctions();
  registry.registerTypeScriptFunctions();
  return registry;
};
```

## Agent with Tool Use (LLM-based)

```typescript
// agent.ts
import Anthropic from '@anthropic-ai/sdk';
import { FunctionRegistry } from './function-registry';
import { Claim, ClaimDecision } from './function-catalog';

export class ClaimsAgent {
  private client = new Anthropic();
  private registry: FunctionRegistry;
  private model = 'claude-opus-4-1-20250805'; // or your preferred model

  constructor(registry: FunctionRegistry) {
    this.registry = registry;
  }

  async processClaim(
    claim: Claim,
    context: any,
    agentId: string = 'agent-001'
  ): Promise<{ decision: ClaimDecision; auditTrail: any }> {
    // Build toolkit for this specific claim
    const toolkit = this.registry.buildToolkit(claim, context, agentId);

    // Convert toolkit to Claude tool definitions
    const tools = toolkit.availableTools.map(fn => ({
      name: fn.id,
      description: fn.description,
      input_schema: {
        type: 'object',
        properties: fn.params,
        required: Object.keys(fn.params).filter(k => fn.params[k].required),
      },
    }));

    // Initial prompt to agent
    const messages: Anthropic.MessageParam[] = [
      {
        role: 'user',
        content: `Process this medical claim and make an approve/deny/escalate decision.

Claim #: ${claim.claimNumber}
Amount: $${claim.amount}
Diagnosis codes: ${claim.diagnosisCodes.join(', ')}
Procedure codes: ${claim.procedureCodes.join(', ')}
Service date: ${claim.serviceDate.toISOString()}

Use the available tools to:
1. Validate the codes
2. Check eligibility
3. Search for similar claims
4. Apply bundling if needed
5. Score risk if needed
6. Make a final decision

Always explain your reasoning.`,
      },
    ];

    let decision: ClaimDecision | null = null;
    let continueLoop = true;

    while (continueLoop) {
      const response = await this.client.messages.create({
        model: this.model,
        max_tokens: 4096,
        tools: tools as Anthropic.Tool[],
        messages,
      });

      // Process response
      for (const block of response.content) {
        if (block.type === 'text') {
          // Agent's reasoning/decision
          if (
            block.text.toLowerCase().includes('approve') ||
            block.text.toLowerCase().includes('deny') ||
            block.text.toLowerCase().includes('escalate')
          ) {
            decision = this.parseDecisionFromText(block.text, claim, toolkit);
          }
        } else if (block.type === 'tool_use') {
          // Agent calling a function
          try {
            const { output, toolCall } = await this.registry.callFunction(
              block.name,
              block.input as Record<string, any>,
              toolkit,
              agentId
            );

            // Add tool result to messages
            messages.push({
              role: 'assistant',
              content: response.content,
            });

            messages.push({
              role: 'user',
              content: [
                {
                  type: 'tool_result',
                  tool_use_id: block.id,
                  content: JSON.stringify(output),
                },
              ],
            });
          } catch (err) {
            messages.push({
              role: 'assistant',
              content: response.content,
            });

            messages.push({
              role: 'user',
              content: [
                {
                  type: 'tool_result',
                  tool_use_id: block.id,
                  is_error: true,
                  content: `Error: ${(err as Error).message}`,
                },
              ],
            });
          }
        }
      }

      // Stop if decision made or max iterations
      if (
        response.stop_reason === 'end_turn' ||
        decision !== null ||
        messages.length > 20
      ) {
        continueLoop = false;
      }
    }

    if (!decision) {
      decision = {
        claimId: claim.id,
        action: 'escalate',
        explanation: 'Agent could not reach a decision',
        reasonRules: [],
        requiresPriorAuth: false,
        timestamp: new Date(),
      };
    }

    // Generate audit report
    const auditTrail = await this.registry.generateAuditReport(claim.id);

    return { decision, auditTrail };
  }

  private parseDecisionFromText(
    text: string,
    claim: Claim,
    toolkit: any
  ): ClaimDecision {
    let action: 'approve' | 'deny' | 'escalate' = 'approve';
    if (text.toLowerCase().includes('deny')) action = 'deny';
    else if (text.toLowerCase().includes('escalate')) action = 'escalate';

    return {
      claimId: claim.id,
      action,
      amount: action === 'approve' ? claim.amount : 0,
      explanation: text,
      reasonRules: toolkit.callLog.map((call: any) => ({
        ruleId: call.toolId,
        ruleName: call.toolId,
        matched: true,
        input: call.input,
        output: call.output,
        section: `Tool call at ${call.timestamp}`,
      })),
      requiresPriorAuth: text.toLowerCase().includes('prior auth'),
      timestamp: new Date(),
    };
  }
}
```

## Usage: Agent Processes Claim

```typescript
// main.ts
import { initializeRegistry } from './function-registry';
import { ClaimsAgent } from './agent';
import { Claim } from './function-catalog';

async function main() {
  const registry = initializeRegistry();
  const agent = new ClaimsAgent(registry);

  const claim: Claim = {
    id: 'claim-001',
    claimNumber: 'CLM123456',
    patientId: 'PAT-789',
    providerId: 'PROV-456',
    amount: 15000,
    diagnosisCodes: ['E11.9', 'I10'],
    procedureCodes: ['99214', '92004'],
    serviceDate: new Date('2024-01-15'),
    submissionDate: new Date('2024-01-20'),
  };

  const context = {
    eligibility: { patientActive: true, planType: 'PPO' },
    providerHistory: { claims_last_year: 150, approval_rate: 0.95 },
    analytics: true,
  };

  console.log('🤖 Agent processing claim...\n');

  const { decision, auditTrail } = await agent.processClaim(
    claim,
    context,
    'agent-claims-001'
  );

  console.log('Decision:', decision.action);
  console.log('Amount:', decision.amount);
  console.log('Explanation:', decision.explanation);
  console.log('\nAudit Trail:');
  console.log(JSON.stringify(auditTrail, null, 2));
}

main().catch(console.error);
```

---

## Key Architecture Points

1. **All functions are tools** — COBOL and TypeScript treated identically as callable functions
2. **Agent decides tool use** — LLM chooses which functions to call and in what order
3. **Budget constraints** — agents get cost-point budgets to prevent runaway calls
4. **Complete audit trail** — every function call logged with input/output/duration
5. **Graceful degradation** — if a COBOL function fails, agent can try alternatives
6. **Gradual migration** — as you replace COBOL functions with TypeScript, you just swap the implementation; agent code doesn't change
7. **Safety preconditions** — each function has guards (preconditions, allowedRoles, maxCallsPerClaim)

This way, agents can leverage 20 years of COBOL logic while you incrementally build better TypeScript replacements.
