---
name: evaluating-abap-unit-tests
description: Evaluate, write, and remediate ABAP Unit (AUnit) test suites so they actually fail when the implementation breaks. Use when judging whether an existing AUnit suite discriminates a broken implementation from a correct one, when adjudicating a review finding about ABAP tests, when creating or fixing ABAP unit tests, when isolating dependencies with the ABAP SQL (OSQL), CDS, Authority Check, Function Module, AMDP, or RAP BO test double frameworks, when choosing a testability seam for code with no injection point, or when planning realistic coverage for ABAP reports, classes, and function groups. Triggers on "review this AUnit suite", "the tests pass but do they test anything", "false green ABAP test", "is this ABAP test suite any good", "write ABAP unit tests", "mock a BAPI in ABAP Unit", "cl_function_test_environment", "cl_osql_test_environment", "cl_aunit_authority_check", "ABAP test coverage below target", "how do I test this without a seam".
license: Apache-2.0
---

# ABAP Unit Testing

Standards for writing and reviewing ABAP Unit tests. The organising idea is **discriminability**: a
suite is only worth its runtime if a broken implementation makes it go red. Test count, coverage
percentage, and "all green" are all proxies that can be satisfied by a suite that proves nothing.

## When to use this skill

- Writing an AUnit suite for a report, class pool, or function group.
- Reviewing an existing suite — especially generated or inherited code — for false green.
- Deciding which isolation framework fits a dependency.
- A dependency has no injection point and you need a seam.
- Coverage is below target and you need to know whether that is a defect or a ceiling.

## Operating rule: verify before you write

1. **Confirm framework API signatures against live documentation.** Do not generate framework calls
   from memory — a plausible-looking method name is not confirmation. (Real example: `set_tables_parameter`
   read plausibly from a blog; the actual method is `set_table_parameter`, singular. One letter, one
   failed activation.) If the documentation does not show the exact call, write it, activate, and let
   the compiler adjudicate.
2. **Map the coverage surface before writing test methods** — decide what is testable, what is
   structurally uncoverable, and which components you will target. See `references/coverage-strategy.md`.
3. **Touch production code only where testability genuinely requires it**, and only as a minimal seam.
4. **Know what each check catches** before claiming a defect is or is not detectable:

| Stage | Catches a wrong or non-existent function module name? |
|---|---|
| Standard syntax check | **No.** A literal name in `CALL FUNCTION` is not resolved — the call may be remote. Activates cleanly. |
| Extended check (SLIN) / ATC | **Yes.** Resolves the name and checks the interface. Cheapest bulk screen. |
| Runtime | **Yes** — `CX_SY_DYN_CALL_ILLEGAL_FUNC`. |
| Unit test with FM doubles | **Yes**, and it is the only stage that also proves the parameters bind and the LUW behaves. |

Practical consequence: to triage many programs for hallucinated FM names, run ATC first. Write FM
doubles for behaviour you need to assert, not to discover whether a name is real.

## Test class skeleton

```abap
CLASS ltc_<unit> DEFINITION FINAL FOR TESTING
  DURATION SHORT
  RISK LEVEL HARMLESS.

  PRIVATE SECTION.
    DATA cut TYPE REF TO <interface_under_test>.   " interface, not class

    METHODS setup.
    METHODS <given>_<expected> FOR TESTING.        " name = given + expected, never the "when"
ENDCLASS.

CLASS ltc_<unit> IMPLEMENTATION.
  METHOD setup.
    cut = NEW <class_under_test>( ).
  ENDMETHOD.

  METHOD <given>_<expected>.
    " Given
    " When
    " Then
  ENDMETHOD.
ENDCLASS.
```

- `DURATION SHORT RISK LEVEL HARMLESS` for pure-logic and doubled tests.
- `cut` is the default name for the code under test. Use a descriptive name only when several setups
  coexist (`empty_blog_post`, `very_long_blog_post`).
- **No `teardown`** unless you genuinely release external resources — `setup` overwrites `cut` and the
  doubles anyway. OSQL/CDS environments are the exception: `destroy( )` them in `class_teardown`.
- `setup` and `teardown` must be declared in the `PRIVATE SECTION` of each test class **directly**.
  ABAP Unit does not pick them up from an inherited `lth_*` parent.
- No redundant guards (`IF cut IS BOUND`), no commented-out test code.

## Given / When / Then

- **Given** — set up state and doubles, as precisely as possible.
- **When** — **exactly one** call to the code under test. More than one and you cannot tell which call failed.
- **Then** — a small number of focused assertions.

If Given or Then grow until the three sections blur, extract helper sub-methods (`given_two_employees( )`).
When the call under test needs many parameters, extract it behind a helper with defaults so the test reads
as one line of intent.

Name test methods for *given + expected*, never the "when": `reads_existing_entry`,
`throws_on_invalid_key`, `raises_when_display_forbidden`. Not `test_1`, `test_loop`.

## Test against interfaces; inject via the constructor

- Type `cut` as an **interface**: `DATA cut TYPE REF TO <some_interface>.`
- **Test publics, not privates.** A strong urge to test a private method is a design signal — usually a
  concept wanting its own class, or domain logic tangled with glue code. Drive the behaviour through the
  public entry point instead of widening scope.
- **Dependency inversion:** hand every dependency to the **constructor**. No setter injection, no
  `FRIENDS` injection (which builds the production dependency first and breaks on renames).
- Prefer instance methods behind an interface over `CLASS-METHODS`, so doubles can be injected.
- For `CREATE PRIVATE` classes, `CLASS ... DEFINITION LOCAL FRIENDS unit_tests` is acceptable **only** to
  reach the dependency-inverting constructor — never to poke private members with mock data.
- **`LOCAL FRIENDS` gotcha in single-source reports:** re-opening a production local class late in the
  source (after its implementation and `START-OF-SELECTION`) can fail activation with *"A PUBLIC class can
  only be defined within a global CLASSPOOL."* Prefer driving the class through its public interface. If
  logic genuinely must be reached and there is no interface seam, move those methods to the
  `PUBLIC SECTION` rather than forcing `FRIENDS`.
- **Object reference identity** cannot be asserted with `assert_equals` (it types `act`/`exp` as `SIMPLE`).
  Use `assert_true( xsdbool( ref1 = ref2 ) )`.

## Choosing a double

Pick by **what the production code actually calls**, not by what is convenient.

| Production code contains | Use | Notes |
|---|---|---|
| `SELECT` on a transparent table | `cl_osql_test_environment` | Also covers `UPDATE`/`INSERT`/`DELETE`. No PK-uniqueness enforcement. |
| `SELECT` on a CDS view / table entity | `cl_cds_test_environment` | Read access only. Flag available to disable DCL. |
| `AUTHORITY-CHECK` | `cl_aunit_authority_check` | **Deny only** — cannot grant. Happy path needs an injected stub. |
| `CALL FUNCTION` (BAPIs, `ENQUEUE_*`, `BAPI_TRANSACTION_COMMIT`) | `cl_function_test_environment` | ABAP 7.56+. Local calls only. |
| Collaborator behind a **global** interface | `cl_abap_testdouble` | Data-in / data-out. |
| Collaborator behind a **local** (`lif_*`) interface | hand-rolled `ltd_*` class | `cl_abap_testdouble` needs a global interface. |
| Dependency with no interface and no injection point | add a seam first | Then double it as above. |
| RAP business object (EML) | `cl_botd_txbufdbl_bo_test_env` **or** `cl_botd_mockemlapi_bo_test_env` | Mutually exclusive — one per test. |
| AMDP method (SQLScript) | `cl_amdp_test_environment` | Also the route for a CDS table function. |
| `CALL FUNCTION ... DESTINATION` | **none of the above** | A remote call. `cl_function_test_environment` cannot intercept it — extract a `lif_*` seam. |
| A genuinely unforceable system call | document the gap | Do not fake it. |

Full lifecycle code, configuration vocabulary, and gotchas for every framework:
**`references/isolation-frameworks.md`**.

**Two doubles are usually needed, not one.** One at the collaborator interface for orchestration tests,
one at the framework layer beneath the adapter for adapter tests. See the seam ceiling below.

Using a double is always three steps: **create → configure (optional) → inject**. Skipping "inject" is
the most common isolation failure — the double exists but production still calls the real thing.

## The seam ceiling — the dominant failure mode

A seam (`lif_*` interface plus constructor injection) and a double (`ltd_*`) are **complementary**: the
seam creates the injection point, the double exploits it. Neither covers the class *below* the seam.

When every test injects a double at the service boundary, the real adapter — the BAPI calls, the LUW
control, the lock lifecycle — has **0% coverage**. The suite may have honest assertions and still be
**vacuous with respect to the requirement**. Symptoms:

- Adding logic to the adapter makes coverage go **down** while the test count goes up.
- A wrong FM name, swapped parameters, a deleted `COMMIT`, or an invalid enqueue mode all leave the suite green.

```
lcl_runner
    |
    +-- lif_service        <-- hand-rolled double sits here (orchestration tests)
    |
lcl_service                <-- must be instantiated directly by its own test class
    |
    +-- CALL FUNCTION ...  <-- cl_function_test_environment doubles here
    +-- SELECT ...         <-- cl_osql_test_environment doubles here
```

**The fix is not fewer doubles — it is doubling at the right layer.** Add a second test class that
instantiates the **real** adapter and doubles the layer beneath it.

Coverage counts alone mislead: 26 tests can yield lower meaningful coverage than 5 if all 26 resolve to
the same hand-rolled double and leave the real `SELECT` and its `WHERE` clause at 0%.

Coverage targets, uncoverable code, and how to read statement vs. branch vs. procedure coverage:
**`references/coverage-strategy.md`**.

## False green

A test that passes for the wrong reason is worse than no test. The full catalogue of forbidden patterns,
with examples, is in **`references/false-green.md`**. The short list:

- `CHECK` in a test body — it can silently skip the rest and still report green.
- Swallowed exceptions, or the only assertion nested behind a condition that may be false.
- **The value under test overwritten between the When and the Then** — the single most common fake-test
  pattern in generated code.
- A test method with no assertion; a commented-out "When".
- Asserting mere presence (`assert_not_initial`, `assert_bound( cut )`, `assert_true( abap_true )`).
- Asserting quantity (`lines( ) = 3`) where content is assertable.
- A fixture that cannot produce the scenario the test name claims — e.g. a double dispatching on a
  configured value with no branch for the value the test sets.
- Live external calls in a `RISK LEVEL HARMLESS` test.

**The empirical discriminator:** substitute a deliberately broken implementation and run the suite. If it
still passes, the suite is defective regardless of how many tests it contains.

## Testability seams

When a dependency is hard-instantiated with no interface and no injection point:

1. Extract a **local interface** for the dependency (e.g. `lif_authorization_checker`).
2. Add an **optional constructor parameter** defaulting to the real implementation in production and
   accepting a stub in tests.

Production behaviour is unchanged; the test gains a deterministic injection point. This is the smallest
change that satisfies dependency inversion.

Avoid:

- **`TEST-SEAM`** — invasive, entangled with private internals, hard to keep alive. Only as a temporary
  workaround for legacy code or older stacks, and only en route to a real seam. **It cannot be used in
  reports at all** — class pools and function groups only. Test injections can only refer to seams in the
  same function pool. Available since NetWeaver 7.5.
- **Sub-classing plus method redefinition** to mock — fragile, and it changes the inheritance contract.
- **`IF is_unit_test_running = abap_true`** branches in production — never.

## Documenting genuine coverage gaps

Some branches are honestly unreachable: a system call that cannot be forced to fail, an OSQL
insert-raise path (OSQL doubles do not reliably enforce PK uniqueness). Record the gap and name the
refactor that would close it, so it reads as a decision rather than an oversight:

```abap
METHOD catch_uuid_error_gap.
  " The CATCH cx_uuid_error branch cannot be covered: cl_system_uuid=>create_uuid_x16_static
  " is a system call that cannot be forced to raise from a unit test without injecting the
  " UUID provider. Fix: wrap it behind lif_uuid_provider so a failing double can be injected.
  cl_abap_unit_assert=>fail(
    'COVERAGE GAP: cx_uuid_error branch requires injecting cl_system_uuid behind an interface.' ).
ENDMETHOD.
```

Use sparingly — most "can't test this" cases are a missing seam. Where an all-green run is a hard
requirement, name the uncoverable paths in the file header comment instead of using `fail( )`, which
would break the gate.

**LUW control in an event block is structurally undrivable.** ABAP Unit cannot drive
`START-OF-SELECTION`, so a `BAPI_TRANSACTION_COMMIT` sitting there can never execute under test.
Do not add a function environment for it — the doubles would sit unused and the `create( )` list would
imply coverage that does not exist. Record the gap. The only fix is a production change: move the LUW
boundary into the orchestrator method.

## Naming and style

- **Name the test class by its purpose, not by the class under test** — either the *when*
  (`ltc_<public method name>`) or the *given* (`ltc_<common setup semantics>`). Prefer `ltc_settlement_luw`,
  `ltc_locked_deal` over `ltc_deal_runner`. `ltc_test` says nothing.
- **Prefixes:** `ltc_` test class · `ltd_` test double · `lth_` test helper. Shared helpers belong in an
  `lth_*` class, not copy-pasted across suites.
- **Method names ≤ 30 characters** — ABAP caps method identifiers at 30. A longer test method name is a
  hard syntax error, not a warning. Shorten by dropping the implied "when"/"test". If names keep hitting
  the ceiling, the class is covering several givens — split it.
- **snake_case** throughout, consistent with the object.
- **Meaningful test data:** give meaningless values obviously meaningless names and values
  (`DATA(alert_id) = '42'.`). Do not make fake data look like real customizing. Make differences easy to
  spot — suffix `...END1` / `...END2` rather than burying them mid-string.

## Review discipline

When adjudicating a suite or another reviewer's finding:

1. **Open the method before agreeing.** Most bad findings come from pattern-matching the shape of a test
   file rather than reading the production code.
2. **Separate the coverage fact from the defect claim.** "This class has 0% coverage" and "this class is
   wrong" are different statements needing different evidence.
   - *Good:* "The suite tests only `lcl_runner` through an injected double; `lcl_service` is entirely
     uncovered, so the commit-before-unlock ordering and the enqueue mode both survive an all-green run."
   - *Bad:* "The production code calls the wrong API and omits commit/rollback" — asserted about code
     where the FM was correct and the commit was present but misplaced.
3. **"All tests pass" is not "the tests are fake."** A suite can have real assertions and still be vacuous
   with respect to the requirement. Say which.
4. **Confirm what is right, not only what is wrong.** If three prior reviewers claimed a missing COMMIT
   and it is present, say so — the withdrawal is part of the finding.
5. **Classify every claim** as Confirmed (read it, or the compiler/ATC said so), Inferred (consistent with
   how SAP normally works, not checked here), or Unknown (no basis — do not claim it). Custom objects
   break every convention; if an FM is custom-coded, standard parameter semantics do not apply.

## Checklist

**Structure**
- [ ] `FINAL FOR TESTING DURATION SHORT RISK LEVEL HARMLESS`
- [ ] `cut` typed as an interface; instantiated in `setup`
- [ ] `setup`/`teardown` declared in each test class's own `PRIVATE SECTION`
- [ ] Given / When / Then present; exactly one call in When
- [ ] Test class named by purpose; method names given + expected, ≤ 30 chars
- [ ] Shared helpers in an `lth_*` class
- [ ] No `teardown` unless releasing real resources; no redundant guards; no commented-out test code

**Correctness**
- [ ] Assertions always execute — no `CHECK`, no nesting that can skip them
- [ ] No value overwritten between the When and the Then
- [ ] Every test method has at least one assertion; no commented-out When
- [ ] Expected exceptions forced via `fail( )` after the call; the specific exception caught
- [ ] Unexpected and sibling exceptions forwarded via `RAISING cx_static_check` — never swallowed
- [ ] Behaviour asserted (a filtered-out row proven), not mere arrival or quantity
- [ ] Exact-content assertions only where the content *is* the contract; incidental wording asserted as a property
- [ ] A deliberately broken implementation makes the suite go red

**Isolation**
- [ ] DB via `cl_osql_test_environment` (create in `class_setup` / clear in `setup` / destroy in `class_teardown`)
- [ ] CDS via `cl_cds_test_environment`; RAP BO via one `cl_botd_*` variant, never both
- [ ] Auth via `cl_aunit_authority_check` (deny-only); happy path uses an injected granting stub — no `skip( )`-on-missing-auth guard
- [ ] FM via `cl_function_test_environment` — `clear_doubles( )` in `setup`; `set_table_parameter` (singular) for TABLES
- [ ] COMMIT / ROLLBACK asserted via `verify( )->is_called_once( )` / `is_called_times( 0 )`, not merely covered
- [ ] The adapter class itself is instantiated by at least one test class, not only replaced by a double
- [ ] Genuine gaps documented with the refactor that would close them
- [ ] No `COMMIT WORK`; no `cl_abap_unit_assert` in production code; no leftover locks

**Gates**
- [ ] Coverage target met on the coverable surface, with uncoverable code excluded and explained
- [ ] ATC clean
- [ ] Pretty Printer applied
- [ ] ABAP Cleaner run
