
# 🧪 Hypotheses & Attack Paths — PROJECT_NAME


> Test-driven structure to keep track of your active audit guesses.

| ID    | Description                                | Status     | Confirmed? | Notes                     |
| ----- | ------------------------------------------ | ---------- | ---------- | ------------------------- |
| H-001 | Share price manipulation via harvest delay | 🧪 Testing | ⏳          | See [[talking-to-myself]] |
| H-002 | Reentrancy on `withdraw()` via strategy    | ✅          | ❌          | State updates first       |
| H-003 | ERC4626 violation if strategy returns 0    | 🚩         | ⏳          | Needs fuzz test           |

---

## New Hypothesis Ideas

- Can `deposit()` be front-run to cause overminting?
- Is `totalAssets()` manipulable via bad strategy accounting?



