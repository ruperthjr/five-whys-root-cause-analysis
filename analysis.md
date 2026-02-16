# Five Whys Root Cause Analysis: Mobile Banking App Crash

## Bug Scenario Summary

**Application**: Mobile Banking App  
**Symptom**: ArrayIndexOutOfBoundsException during money transfers  
**Time Window**: 11:55 PM to 12:05 AM  
**Frequency**: Consistent during this time window  
**Severity**: Critical (blocks transactions)  
**Impact**: User frustration, lost transactions, support costs

## Five Whys Analysis

### Why 1: Why did the app crash during money transfers?

**Answer**: The app threw an ArrayIndexOutOfBoundsException when attempting to access a date array element that does not exist.

**Evidence**:
- Stack trace shows `ArrayIndexOutOfBoundsException` at line 247 in `TransactionProcessor.java`
- Error occurs in `getBusinessDayOffset()` method
- Logs indicate the array index was 31 but array size was only 30

**Code Context**:
```java
int dayOffset = businessDayOffsets[currentDay];
```

Where `currentDay` was 31 (attempting to access 32nd element in 0-indexed array).

### Why 2: Why was the array index 31 when the array size is only 30?

**Answer**: The code uses the day of the month directly as an array index without accounting for months having different numbers of days, and without zero-indexing adjustment.

**Evidence**:
```java
Calendar calendar = Calendar.getInstance();
int currentDay = calendar.get(Calendar.DAY_OF_MONTH);
int dayOffset = businessDayOffsets[currentDay];
```

- The array `businessDayOffsets` has 30 elements (indices 0-29)
- `Calendar.DAY_OF_MONTH` returns 1-31 (not zero-indexed)
- On the 31st day of any month, accessing `businessDayOffsets[31]` exceeds array bounds
- Issue also occurs on day 1 when code accesses index 1 instead of 0 (off-by-one error)

### Why 3: Why does this only occur between 11:55 PM and 12:05 AM?

**Answer**: The transaction processing system performs date calculations based on "business day" logic that triggers during the day boundary transition, and the bug only manifests on specific calendar days when combined with this time window.

**Evidence**:
- Transactions initiated between 11:55 PM and 12:05 AM span two calendar days
- System checks if transaction should be processed "next business day"
- During month-end (day 30 → day 31 or day 31 → day 1), the date transition triggers the array access
- Logs show transactions started at 11:57 PM on Jan 31st crashed when system clock rolled to Feb 1st
- Only January (31 days) showed consistent crashes; February (28/29 days) showed crashes on different pattern

**Time Window Analysis**:
The 10-minute window (11:55 PM - 12:05 AM) is the transaction timeout period. Transactions started before midnight but completing after midnight trigger date-based business logic with the new date value.

### Why 4: Why wasn't this array bounds issue caught during development or testing?

**Answer**: The test suite did not include test cases for month-end dates, particularly the 31st day, and did not test transactions spanning the midnight boundary.

**Evidence**:
- Review of `TransactionProcessorTest.java` shows only tests for days 1-28
- No integration tests for time-boundary conditions
- Test data used fixed dates: Jan 15, Feb 10, Mar 20 (all mid-month)
- Manual testing checklist does not include "test on last day of month" item
- Code review comments show focus on business logic, not edge cases

**Test Coverage Report**:
```
TransactionProcessor.java: 87% coverage
Missing coverage: 
  - Line 247: getBusinessDayOffset() edge cases
  - Lines 310-315: Date rollover scenarios
```

### Why 5: Why were month-end date edge cases not part of the test requirements?

**Answer**: The requirements specification for the transaction feature did not explicitly identify date-boundary conditions as a critical scenario, and the development team did not have a systematic edge-case analysis process during the requirements phase.

**Evidence**:
- Functional requirements document (FRD-2847) states: "System shall calculate business day offsets for transaction processing"
- Requirements mention "business days" but provide no examples of edge cases
- No mention of month boundaries, leap years, or timezone considerations
- Requirements review checklist does not include "edge case identification" step
- Product manager interview revealed: "We focused on happy path user stories"
- No formal FMEA (Failure Mode and Effects Analysis) was conducted

**Process Gap**:
The requirements gathering process follows a user story format without a complementary technical risk analysis phase. Edge cases are left to developers to identify, but developers lack guidance on what constitutes a critical edge case for financial transactions.

## Root Cause Statement

**The root cause of the ArrayIndexOutOfBoundsException is the absence of a systematic edge-case analysis process during requirements definition, which led to incomplete functional requirements that did not specify how the system should handle date boundary conditions (particularly month-end dates). This gap resulted in developers implementing date-based array access logic without considering months of varying length, and test engineers creating test suites that did not cover these critical scenarios.**

## Why This is the True Root Cause

### Test 1: Will Addressing This Prevent Recurrence?

**Yes**. Implementing a systematic edge-case analysis process during requirements will ensure:
- Date-boundary scenarios are explicitly documented
- Developers receive clear specifications for edge cases
- Test requirements include comprehensive edge-case coverage
- Similar issues with other date-dependent features are prevented

### Test 2: Is It Within the Team's Control?

**Yes**. The team can:
- Add edge-case analysis step to requirements process
- Create edge-case checklist templates
- Train product managers and developers on edge-case identification
- Update requirements review criteria
- Modify code review guidelines to require edge-case consideration

### Test 3: Is It Specific and Actionable?

**Yes**. The root cause points to a specific process gap (lack of edge-case analysis) with clear remediation paths:
- Add formal edge-case analysis phase
- Create documentation templates
- Update team processes
- Provide training

### Test 4: Does It Explain the Recurrence Pattern?

**Yes**. The issue recurs predictably:
- Occurs on 31st of every month with 31 days
- Occurs during specific 10-minute window (transaction timeout period)
- Was not caught because test cases systematically avoided these dates
- Would occur for any similar date-dependent feature without edge-case requirements

## Alternative Hypotheses Considered and Rejected

### Hypothesis 1: Developer Skill Gap

**Rejected Because**: The developer correctly implemented the specified requirements. The requirements did not mention month-end handling. Code review showed appropriate coding practices for the given specification.

### Hypothesis 2: Insufficient Testing

**Rejected Because**: This is a symptom, not the root cause. Tests followed the requirements, which did not specify month-end scenarios. More testing without better requirements would not have caught this.

### Hypothesis 3: Poor Code Quality

**Rejected Because**: The code has 87% test coverage and follows team standards. The issue is not code quality but incomplete requirements that led to incomplete logic.

### Hypothesis 4: Lack of Code Review

**Rejected Because**: Code was reviewed by two senior engineers. Reviewers checked against requirements, which did not flag date edge cases. The review process worked correctly given the inputs.

### Hypothesis 5: Calendar Library Bug

**Rejected Because**: Java's Calendar class works correctly. The issue is how the application uses the day value, not the Calendar implementation itself.

## Validation Against Evidence

### Why the Root Cause Fits All Evidence:

1. **Array bounds error**: Direct result of unspecified edge-case requirements
2. **Time-specific occurrence**: Transaction timeout window + date rollover not in requirements
3. **Month-end pattern**: Months with 31 days not considered in requirements
4. **Missing test cases**: Tests derived from requirements; requirements lacked edge cases
5. **Code review didn't catch it**: Reviewers validated against requirements; requirements were incomplete
6. **87% code coverage**: Coverage measured what was specified, not what was needed

### Why Alternative Causes Don't Fit:

- If it were a skill gap, other code would show similar issues (it doesn't)
- If it were insufficient testing, some test cases would fail (all passed)
- If it were poor code quality, coverage would be low (it's 87%)
- If it were lack of review, other obvious bugs would exist (code is otherwise solid)
- If it were a library bug, all Java apps would have this issue (they don't)

## Systemic Nature of Root Cause

This root cause explains not just this bug, but a pattern:
- Similar date-related bug in scheduling feature (ticket DEF-1234)
- Currency rounding issue at boundary values (ticket DEF-2456)
- Timezone handling bug during DST transitions (ticket DEF-3789)

All these issues share the same root cause: **edge cases not systematically identified during requirements phase**.

## Conclusion

The true root cause is a **process gap in requirements engineering**. While the immediate fix is to correct the array logic, the preventive solution is to implement systematic edge-case analysis during requirements definition. This ensures comprehensive specifications that guide development and testing, preventing this entire class of issues.

Without addressing this root cause, similar issues will continue to emerge in other features whenever edge cases exist but remain unspecified.