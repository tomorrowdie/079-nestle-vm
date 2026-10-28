# Fixtures — Claims Guard

12 fictional test cases (6 Vietnamese, 6 English; 7 images, 5 text) with known expected answers.
Brands **DemoMalt / DemoGrow / DemoBrew are fictional**. No real Nestlé packaging, logos or claims.
Approved claims for these brands: `data/claims/demo_approved_claims.yaml`.

## Why fictional, not real product photos
A test case needs a known correct answer. Each image here has violations planted on purpose,
so the pipeline can be scored automatically. Real pack photos have no answer key.

## Files per case
```
cases/C01/input.txt | input.png
cases/C01/expected.yaml
index.yaml               list of all cases
```

## Expected fields per claim
- `expected_match`: APPROVED | VARIANT | NOT_IN_LIBRARY (step 4, library match)
- `expected_reg`: OK | REG_RISK | NEEDS_HUMAN (step 5, rule check)
- `expected_reg_tags`: risk pattern tags. Real `rule_id`s are filled in only after the regulatory corpus is VERIFIED.
- `expected_final`: worst of the two, by precedence
  `REG_RISK > NEEDS_HUMAN > NOT_IN_LIBRARY > VARIANT > APPROVED`

## Overall status rule
- any `REG_RISK` → `BLOCK`
- else any `NEEDS_HUMAN`, `NOT_IN_LIBRARY` or `VARIANT` → `REVIEW`
- else → `PASS`

## Coverage
| Case | Lang | Input | Pattern | Overall |
|---|---|---|---|---|
| C01 | en | text | all approved | PASS |
| C02 | vi | image | "smarter" outcome claim for kids | BLOCK |
| C03 | en | image | wrong % (30 vs 25) + immunity | BLOCK |
| C04 | vi | text | doctor endorsement | BLOCK |
| C05 | en | image | research-institute backing + height outcome | BLOCK |
| C06 | vi | image | right words, condition fails (calcium 8% NRV) | REVIEW |
| C07 | en | text | "cures" | BLOCK |
| C08 | vi | image | infant formula promo, breast-milk comparison | BLOCK |
| C09 | en | image | "zero side effects" | BLOCK |
| C10 | vi | text | "number 1 in Vietnam" needs evidence | REVIEW |
| C11 | en | text | all approved | PASS |
| C12 | vi | image | "100% natural" with added sugar/flavour | REVIEW |

Text on images is not part of any claim: the footer line `FICTIONAL TEST FIXTURE · Claims Guard demo`
(listed under `non_claim_text`).

Vietnamese wording is machine-drafted: TODO-VERIFY-VI with a native speaker before any external demo.

## Adding real examples later
Put real screenshots in `data/fixtures/real_private/` (gitignored). Use them only in a live demo,
never commit them to a public repo.
