# Coverage strategy — the ceiling principle

Classic ABAP programs contain code that ABAP Unit **categorically cannot reach**. Forcing coverage of it
wastes effort and never moves the number. Identify the ceiling first, exclude the uncoverable, then target
the testable components so the *coverable* surface clears the threshold cleanly.

## Structurally uncoverable in a unit test

- **`INITIALIZATION` blocks with selection-text assignments** (`%_<param>_%_app_%-text = '...'`). These are
  selection texts and belong in the program's **text elements**, not in code. Dozens of such statements drag
  whole-program coverage down for no reason.
  *Preferred fix:* move them to text elements and delete the block. The uncoverable statements vanish from
  the denominator.
- **Interactive ALV** — `REUSE_ALV_GRID_DISPLAY_LVC` cannot run headless. Exclude it; leave it to manual or
  integration tests.
- **The ALV happy path of a void `execute( )`** — the branch ending in the ALV call is unreachable headless.
  The *guard branches* of that same method are reachable through the public interface.
- **Event blocks.** ABAP Unit cannot drive `START-OF-SELECTION`, so LUW control sitting there
  (`BAPI_TRANSACTION_COMMIT` / `ROLLBACK`) can never execute under test. Do not create a function
  environment for those modules — the doubles would sit unused and the `create( )` list would imply
  coverage that does not exist. Record the gap. The only fix is a production change: move the LUW boundary
  into the orchestrator method.

## The targeting move

Isolate the program's real logic into testable units and cover those. Typical high-value targets:

- **A pure-logic builder** (a field-catalog builder, a formatter) — no doubles needed; one call exercises
  many branches.
- **A data provider** — isolate with the OSQL and Authority frameworks.
- **A controller with a void `execute( )`** whose logic is private and whose only public entry ends in ALV.
  Cover it through the public interface — no `FRIENDS`, no production change — by driving the early-return
  guard branches: *no input selected*, *authorization denied*, *empty result*. Each issues a `MESSAGE` and
  `RETURN`s **before** the ALV call. Capture `sy-msgty` immediately after the call:

  ```abap
  METHOD stops_when_no_field_chosen.
    " Given - all selection flags cleared in setup
    " When
    cut->execute( ).
    DATA(message_type) = sy-msgty.
    " Then
    cl_abap_unit_assert=>assert_equals( act = message_type exp = 'S' ).
  ENDMETHOD.
  ```

  This is a **behavioural** assertion — it confirms the guard branch ran and returned, not which exact
  message — and it is the most you can assert on a void, ALV-bound method without injecting a display
  double. It lifts the class well off 0%. Drive *authorization denied* with the Authority framework's empty
  restriction; it is deterministic and needs no pass set.

Two such classes are usually enough to clear a 50% floor.

## Read the right number

ADT measures three things, switchable in the *ABAP Coverage* view. They are not interchangeable.

| Measure | Green | Yellow | Red |
|---|---|---|---|
| **Statement** | statement executed | — | statement never executed |
| **Branch** | *all* branches executed | at least one branch not executed | no branch executed |
| **Procedure** | procedure called | — | procedure never called |

Procedure coverage flatters a suite badly: calling `run( )` once makes it green while every branch inside
stays untested. Branch coverage is the honest one for logic. Quote statement coverage for a numeric gate,
and read branch coverage to find the gaps. Hovering a line shows its execution count, and in branch mode
how often it resolved true vs. false — that is how you spot a condition that only ever went one way.

Always **state which measure you are quoting.** The three give very different numbers for the same suite.

## Exclude uncoverable code from the measurement, not just the narrative

ADT supports this directly: *View Menu → Change Coverage Scope…* restricts the measured objects and
packages, and *View Menu → Exclude from Coverage…* drops specific objects. Use it for the presentation
layer rather than letting an untestable ALV class drag the denominator down.

The scope setting is local to whoever ran the tests, so still write a header comment:

```abap
*&  Not unit-tested (presentation / non-isolatable):
*&    - lcl_alv_display  (REUSE_ALV_GRID_DISPLAY_LVC cannot run headless)
*&    - INITIALIZATION   (selection-screen text assignments -> move to text elements)
```

## What the number does and does not mean

Clean ABAP says *don't obsess about coverage*, and warns that padding a suite to hit a number is worse than
leaving code transparently untested. A percentage floor, where a project imposes one, is a floor on the
**coverable** surface, reached by testing behaviour — not by gaming uncoverable code.

Coverage counts alone mislead in the other direction too: 26 tests can produce lower meaningful coverage
than 5 if all 26 resolve to the same hand-rolled double, leaving the real `SELECT` and its `WHERE` clause at
0%. Coverage measures which lines ran, never whether an assertion would have caught them running wrong.

## Where test code is allowed to live

Only three object types are **test containers**: **class pool, function pool, report**. DDL sources, simple
transformations and the like cannot hold test code — to test a CDS view, put the tests in a separate
container object.

- **Unit tests** go in the local test include of the class under test, so they are found during refactoring
  and run with one keypress.
- **Component, integration and system tests** do not belong to any single class. Put them in the local test
  include of a **separate global class**, marked `FOR TESTING` and `ABSTRACT` so production code cannot
  reference it.
- Link tests to the code they exercise with **test relations** (ABAP Doc, 7.53+), so tests are discoverable
  from the production object:

  ```abap
  "! @testing zcl_deal_runner
  "! @testing zcl_deal_service
  CLASS ltc_settlement DEFINITION FOR TESTING RISK LEVEL DANGEROUS DURATION MEDIUM ABSTRACT.
  ```

In ADT: `Ctrl+Shift+F10` runs a class's tests, `F11` adds coverage, `F12` also runs related classes.
