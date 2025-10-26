# LitAssist Verification Mechanisms Analysis

## Overview
This document provides a comprehensive analysis of all verification mechanisms in the LitAssist codebase, including the newly renamed `enforce_citations` flag and all hardcoded verification logic in each command.

## 1. Citation Enforcement Mechanism (`enforce_citations`)

### What It Does
- **Primary Function**: Controls whether citation verification failures trigger automatic retries with enhanced prompts
- **Not**: A flag to enable/disable verification (verification always happens)
- **Location**: Implemented in `LLMClient.complete()` via `process_citation_verification()`

### How It Works
1. `LLMClient.complete()` always calls `process_citation_verification()`
2. `determine_strict_mode()` checks `client._enforce_citations` attribute
3. If `enforce_citations: true` → Strict mode → `CitationVerificationError` raised → Triggers retry
4. If `enforce_citations: false` → Lenient mode → Warnings only, no retry

### Commands with `enforce_citations` Settings

#### Enforce Citations ENABLED (true):
- **extractfacts**: Always enforces for foundational docs
- **~~strategy~~**: Recently changed to `false`
- **~~brainstorm-orthodox~~**: Recently changed to `false`
- **~~brainstorm-unorthodox~~**: Recently changed to `false`
- **~~counselnotes~~**: Recently changed to `false`

#### Enforce Citations DISABLED (false):
- **strategy**: No retry on citation errors
- **brainstorm-orthodox**: No retry on citation errors  
- **brainstorm-unorthodox**: No retry on citation errors
- **counselnotes**: No retry on citation errors
- **lookup**: Lenient citation checking
- **verification**: Avoids double-enforcement
- **digest**: Uses default (false)
- **draft**: Uses default (false)
- **barbrief**: Uses default (false)
- **caseplan**: Uses default (false)
- All CoVe-related commands

## 2. Hardcoded Verification by Command

### extractfacts
```python
# Line 178: Always runs unless --noverify
verify_content_if_needed(client, combined, "extractfacts", verify_flag=True)
```
- **Verification**: Always enabled by default
- **Citation Check**: Via LLMClient.complete() with `enforce_citations: true`
- **Override**: Only `--noverify` flag disables

### strategy
```python
# Line 363: Always validates citations
citation_issues = llm_client.validate_citations(strategy_content)

# Line 387: Always verifies unless --noverify
verify_content_if_needed(llm_client, strategy_content, "strategy", verify_flag=True)
```
- **Citation Validation**: Always runs (hardcoded)
- **Content Verification**: Always runs unless `--noverify`
- **Citation Enforcement**: `enforce_citations: false` (no retry)

### brainstorm
Most complex verification with multiple layers:

#### Orthodox Generator
```python
# Line 74: Always validates
orthodox_citation_issues = orthodox_client.validate_citations(orthodox_content)
```

#### Unorthodox Generator  
```python
# Line 123: Always validates
unorthodox_citation_issues = unorthodox_client.validate_citations(unorthodox_content)

# Line 84: Always verifies Grok output
verification_result, _ = verify_client.verify(unorthodox_content)
```

#### Core Processing
```python
# Line 380: Full verification if --verify flag
correction, _ = verify_client.verify(combined_content)

# Lines 405, 442: Always validates citations
citation_issues = verify_client.validate_citations(combined_content)
citation_issues = analysis_client.validate_citations(combined_content)
```

### counselnotes
```python
# Lines 148, 209, 308: Only with --verify flag
if verify:
    citation_issues = client.validate_citations(content)
```
- **Citation Check**: Only when `--verify` flag is used
- **Citation Enforcement**: `enforce_citations: false`

### draft
```python
# Line 253: Always verifies unless --noverify
verify_content_if_needed(client, content, "draft", verify_flag=True)

# Also runs: detect_factual_hallucinations() - always
```
- **Content Verification**: Always runs unless `--noverify`
- **Hallucination Detection**: Always runs
- **Citation Enforcement**: Uses default (false)

### digest
```python
# Line 157: Only in "issues" mode
if mode == "issues":
    response, _ = validate_citations_if_needed(response, mode, llm_client)

# Line 234: Uses lenient validation
validate_and_verify_citations(content, strict_mode=False)
```
- **Citation Check**: Only in "issues" mode
- **Mode**: Always lenient (`strict_mode=False`)

### barbrief
```python
# Line 176: Always validates format
validate_case_facts(case_facts_content)

# Line 291: Only with --verify flag
if verify:
    verified, unverified = verify_all_citations(content)
```
- **Format Validation**: Always checks 10-heading format
- **Citation Verification**: Only with `--verify` flag

### verify (standalone command)
```python
# Line 356: Core verification
soundness_result, soundness_model = client.verify(content, ...)
```
- **Purpose**: Dedicated verification command
- **Citation Enforcement**: `enforce_citations: false` (avoids loops)

### lookup
- **No hardcoded verification**
- **Citation Enforcement**: `enforce_citations: false`
- Relies only on LLMClient.complete() verification

### caseplan
- **No verification at all**
- Explicitly rejects `--verify` and `--noverify` flags
- Shows warning if flags are used

## 3. Verification Patterns Summary

### Always-On Verification
Commands that always verify (most strict):
- **extractfacts**: Full verification + citation enforcement
- **strategy**: Citation validation + content verification
- **brainstorm**: Multiple layers of verification

### Conditional Verification
Commands that verify only with flags:
- **counselnotes**: Only with `--verify` flag
- **barbrief**: Only with `--verify` flag
- **draft**: Can disable with `--noverify`

### Mode-Specific Verification
- **digest**: Only in "issues" mode

### No Verification
- **lookup**: Minimal verification
- **caseplan**: No verification support

## 4. Types of Verification

### Citation Checking
- `validate_citations()`: Pattern-based validation
- `validate_and_verify_citations()`: Real-time database checking
- Checks citation format and existence

### Content Verification
- `verify_content_if_needed()`: Semantic verification
- `verify()`: Full soundness checking
- Ensures logical consistency and accuracy

### Specialized Checks
- `detect_factual_hallucinations()`: Draft command specific
- `validate_case_facts()`: Barbrief format validation

## 5. Key Insights

1. **Brainstorm is Special**: Has the most aggressive verification with multiple independent layers that cannot be disabled

2. **Two-Level System**:
   - Base level: LLMClient.complete() always does some verification
   - Command level: Additional hardcoded checks

3. **enforce_citations Misleading**: The flag doesn't control whether verification happens, only whether failures trigger retries

4. **Inconsistent Patterns**: Different commands have very different verification philosophies, from aggressive (brainstorm) to none (caseplan)

5. **Recent Changes**: Several commands recently had `enforce_citations` changed from `true` to `false`, reducing retry aggressiveness

## 6. Recommendations

1. **Consider Renaming**: `enforce_citations` could be better named as `retry_on_citation_errors` for clarity

2. **Standardize Behavior**: Commands have inconsistent verification patterns that may confuse users

3. **Document Clearly**: Each command should clearly state its verification behavior in help text

4. **Separate Concerns**: Citation checking and content verification serve different purposes and should be controllable independently