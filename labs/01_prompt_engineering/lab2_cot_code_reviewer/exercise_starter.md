# Instructor Reference Specification: Practical 1.2

---

## 1. Master Chain-of-Thought (CoT) Prompt Reference Solution

```text
You are a Principal Software Engineer performing a rigorous pre-merge code review. Your goal is to identify defects, trace runtime behavior through dry runs, and provide surgical fixes while maintaining 100% backward compatibility.

### CONSTRAINTS & GUARDRAILS:
1. Do NOT suggest external third-party libraries unless strictly requested in the docstring. Use standard Python 3.10+ built-ins only.
2. Do NOT change the function name, parameters, or return structure.
3. Perform dry-run evaluation on all boundary values before writing any code fixes.
4. Output your analysis using EXACTLY the following 5 markdown section headers.

---

### REQUIRED OUTPUT FORMAT:

### 1. Specification & Edge-Case Matrix
Construct a table detailing expected behavior across all boundaries (e.g. 0, exact threshold values, negative values, types).

### 2. Step-by-Step Logic Trace (Chain-of-Thought)
Trace through the code line by line for a customer who spent EXACTLY $5,000 with 24 months account age. Explain why the existing code passes or fails this case.

### 3. Identified Defects & Severity
List every defect found, categorize severity as [High | Medium | Low], and explain the impact.

### 4. Surgical Code Fix
Provide the corrected Python function. Include clean type annotations and input validations. Do not add bloated dependencies.

### 5. Verification Test Suite
Provide 5-7 standalone Python `assert` statements that can be executed directly to verify all boundaries and edge cases.

---

<target_code>
{{SOURCE_CODE}}
</target_code>
```

---

## 2. Ground-Truth Expected Model Output

### 1. Specification & Edge-Case Matrix

| Case Description | `spend_amount` | `account_age_months` | `is_vip` | Expected Tier | Expected Discount |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Zero / Negative Spend | $0 or -$10 | Any | False | `Inactive` | 0.0 |
| Bronze Entry | $100 | 2 | False | `Bronze` | 0.0 |
| Silver Exact Boundary | $500 | 6 | False | `Silver` | 0.10 |
| Gold Exact Boundary | $2,000 | 12 | False | `Gold` | 0.15 |
| Platinum Exact Boundary | $5,000 | 24 | False | `Platinum` | 0.20 |
| VIP Override (Low Spend) | $50 | 1 | True | `Platinum` | 0.20 |

---

### 2. Step-by-Step Logic Trace (Chain-of-Thought)

Dry-run evaluation for `spend_amount = 5000`, `account_age_months = 24`, `is_vip = False`:
1. Check `spend_amount <= 0` (5000 <= 0 -> `False`).
2. Check `is_vip` (False -> `False`).
3. Check `spend_amount > 5000 and account_age_months > 24`:
   - `5000 > 5000` evaluates to `False`.
4. Check `spend_amount > 2000 and account_age_months > 12`:
   - `5000 > 2000` evaluates to `True`, and `24 > 12` evaluates to `True`.
   - Result assigned: `tier = "Gold"`, `discount = 0.15`.
5. **Defect Root Cause**: The customer qualified for Platinum ($5,000 / 24 mo), but was demoted to Gold because strict inequality (`>`) was used instead of inclusive inequality (`>=`).

---

### 3. Identified Defects & Severity

- **Defect 1 [High Severity - Business Logic Error]**: Strict inequality operators (`>`) used for tier boundaries ($5000, $2000, $500, and 24, 12, 6 months) instead of inclusive `>=`.
- **Defect 2 [Medium Severity - Input Validation]**: Missing defensive checks for null (`None`) or negative tenures.
- **Defect 3 [Low Severity - Type Safety]**: Lacks explicit type annotations.

---

### 4. Surgical Code Fix

```python
from typing import TypedDict, Union

class LoyaltyResult(TypedDict):
    tier: str
    discount: float

def calculate_loyalty_tier(
    spend_amount: Union[int, float], 
    account_age_months: int, 
    is_vip: bool = False
) -> LoyaltyResult:
    """
    Calculates customer loyalty tier and discount rate based on spend and account tenure.
    """
    if spend_amount is None or account_age_months is None:
        raise ValueError("spend_amount and account_age_months must not be None")
    
    if spend_amount <= 0 or account_age_months < 0:
        return {"tier": "Inactive", "discount": 0.0}
    
    if is_vip or (spend_amount >= 5000 and account_age_months >= 24):
        return {"tier": "Platinum", "discount": 0.20}
    elif spend_amount >= 2000 and account_age_months >= 12:
        return {"tier": "Gold", "discount": 0.15}
    elif spend_amount >= 500 and account_age_months >= 6:
        return {"tier": "Silver", "discount": 0.10}
    else:
        return {"tier": "Bronze", "discount": 0.0}
```

---

### 5. Verification Test Suite

```python
def test_loyalty_tier_boundaries():
    assert calculate_loyalty_tier(0, 10) == {"tier": "Inactive", "discount": 0.0}
    assert calculate_loyalty_tier(-50, 12) == {"tier": "Inactive", "discount": 0.0}
    assert calculate_loyalty_tier(200, 3) == {"tier": "Bronze", "discount": 0.0}
    assert calculate_loyalty_tier(500, 6) == {"tier": "Silver", "discount": 0.10}
    assert calculate_loyalty_tier(2000, 12) == {"tier": "Gold", "discount": 0.15}
    assert calculate_loyalty_tier(5000, 24) == {"tier": "Platinum", "discount": 0.20}
    assert calculate_loyalty_tier(10, 1, is_vip=True) == {"tier": "Platinum", "discount": 0.20}

test_loyalty_tier_boundaries()
```
