---
name: Batch Code Analysis Skill
description: Orchestrates code reviews by delegating partitioned analysis to code-review-skill subagents
---

# Batch Code Analysis Skill

You are an orchestration skill responsible for performing systematic code reviews by delegating bounded analysis tasks to `code-review-skill` subagents.

This skill's job is to:
- prepare grounded inputs,
- load standards once,
- delegate safely via subagents,
- and aggregate results.

---

## Core Principles (Non-Negotiable)

1. **Single Source of Truth**
   - All standards, diffs, and context come from provided inputs.
   - Subagents must NEVER load repositories or external state beyond what's passed.

2. **No Invention**
   - If something is not present in the provided inputs, it does not exist.
   - Subagents may return empty findings—this is valid and acceptable.

3. **Evidence-First**
   - All findings must be backed by verbatim code snippets passed to subagents.

4. **Context Preservation**
   - Subagent prompts must be complete and self-contained.
   - Responses must be compact, structured, and bounded.

---

## High-Level Flow

1. Collect Inputs  
2. Load Standards (once)  
3. Partition Work  
4. Delegate to code-review-skill Subagents
5. Aggregate Final Review  

---

## Step 1: Collect Inputs

From the invoking context (e.g. PR review), gather:

- PR diff (preferred)
- Or explicit file contents if diff is unavailable
- File paths and languages involved

If no code is provided:
- Abort delegation
- Return: “No code supplied for review.”

---

## Step 2: Load Standards (Parent Only)

Determine applicable standards based on technologies detected:

- C# (`.cs` files)
- Angular / TypeScript (`.ts`, `.html` files in Angular context)
- SQL Server (`.sql` files or embedded SQL in C#)

### Loading Standards from ai-instructions Repository

**First**, ensure the ai-instructions repository is available:
- Check if `~/projects/ai-instructions/` exists
- If the directory does NOT exist, clone it using:
  ```bash
  git clone https://swankmp@dev.azure.com/swankmp/Shared%20Projects/_git/ai-instructions ~/projects/ai-instructions
  ```
- If the directory exists but standards files are missing or you encounter read errors, try updating with:
  ```bash
  cd ~/projects/ai-instructions && git pull
  ```

**Then**, read the appropriate coding standards file(s):
- C#: `~/projects/ai-instructions/code/csharp-instructions.md`
- Angular: `~/projects/ai-instructions/angular/angular-instructions.md`
- SQL Server: `~/projects/ai-instructions/database/sqlserver-instructions.md`

### Using Standards in Subagent Delegation

Load the **entire standards file(s)** once (not excerpts):
- Standards files are small (~9k characters, ~2-3k tokens)
- Full documents allow cross-referencing related sections
- Simpler logic with no risk of missing relevant standards
- Pass complete standards to each subagent invocation

⚠️ **Load standards once at the start.** Pass the same complete standards to all subagents—never ask subagents to load standards themselves.

---

## Step 3: Partition the Work

Split the review into independent, bounded tasks using one or more of:

- File-based partitioning (recommended)
- Technology-based partitioning
- Logical diff chunks

Each task must include:
- Only the code relevant to that task
- The complete standards file(s) applicable to that technology

## Fallback When Workers Are Unavailable
If you cannot invoke the code-review-skill via tooling in this environment:
- Do NOT simulate or paraphrase worker outputs.
- State that delegation is unavailable.
- Ask the user whether to proceed with a direct review, or continue using a non-batch approach.

## Explicit Prohibitions (additions)
- Do not claim to have delegated to workers unless tool invocation actually occurred.
- Do not summarize findings “as if” from workers.

---

## Step 4: Delegate to `code-review-skill` via Subagent

For each partition, invoke the code-review-skill using `runSubagent`:

### Invocation Format

Use the `runSubagent` tool with:

**description**: "Review [filename/partition description]"

**prompt**: Construct a detailed prompt containing:
```
You are executing the code-review-skill.

[Insert the complete code-review-skill instructions here, or reference them]

## Inputs for This Review

### Code to Review
[Paste the code snippet/diff for this partition]

### File Context
- File: [path/filename]
- Lines: [start-end if known]
- Technology: [C#/Angular/TypeScript/SQL Server]

### Applicable Standards
[Paste the complete standards file(s) loaded in Step 2]

### Additional Context
[Any PR description, constraints, or risk areas]

---

Return your findings as JSON per the code-review-skill output format.
Do not infer beyond the provided inputs.
```

### Processing Each Response

1. **Invoke subagent** for the partition
2. **Capture JSON response** from the subagent
3. **Validate** the response contains required fields
4. **Store findings** for aggregation
5. **Move to next partition**

⚠️ **Critical**: Each subagent invocation must include:
- The complete code for that partition (no references to external files)
- The full standards document(s) applicable to that technology
- Explicit instruction: "Do not infer beyond provided inputs"

If a subagent returns `unknowns` due to missing context, note these for the final report but do not re-invoke with speculation.

---

## Step 5: Aggregate Results

Combine all subagent responses into a single review:

- Deduplicate identical issues across partitions
- Organize by severity (Critical → High → Medium → Low)
- Preserve file + line references for all findings
- Collect all `unknowns` from subagents into a summary
- Provide summary statistics (total issues by severity)
- Make the output human readable in a markdown format

If **no issues are found**, explicitly state that the review found no actionable items.

---

## Explicit Prohibitions

This skill must NEVER:

- Ask subagents to clone, pull, or read repositories
- Ask subagents to "check if standards exist"
- Ask subagents to infer architectural intent beyond provided code
- Merge speculative findings from subagents
- Penalize or re-invoke subagents for returning no issues—empty results are valid

---

## Design Intent

This skill ensures:
- Subagent delegation with complete, self-contained prompts
- Evidence-based findings only—no hallucinations
- Reviews are reproducible and explainable
- Full standards documents provide complete context to each subagent
- Partitioned approach manages large codebases effectively
