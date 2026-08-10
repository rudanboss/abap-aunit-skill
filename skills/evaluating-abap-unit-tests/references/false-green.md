# False green — forbidden patterns

A test that passes for the wrong reason is worse than no test: it consumes runtime, occupies the slot a
real test would fill, and reports safety that does not exist.

## Structural false green

- **`CHECK` in a test body for an expected-failure path.** `CHECK` can silently skip the rest of the method
  and still report green.
- **Swallowed exceptions**, so assertions never run.
- **The only assertion nested behind a condition** that may be false at runtime.
- **Redundant guards** (`IF cut IS BOUND`) when `setup` always instantiates.
- **A test method with no assertion at all** — reachable, green, meaningless.
- **A commented-out "When"** (`" cut->run( ).`) leaving assertions that pass because nothing ran.
- **Live external calls in a `RISK LEVEL HARMLESS` test.** A real RFC or `SELECT` produces a coverage
  number that differs per system and per client. Non-reproducible coverage is worse than low coverage.

## Assertion-shaped false green

**Assert mere presence** — `assert_not_initial`, `assert_bound( cut )`, `assert_true( abap_true )`.
`assert_bound( cut )` only proves `setup` ran.

**Assert quantity instead of content** — `assert_equals( act = lines( messages ) exp = 3 )`. Counts can
match while the content is completely wrong. Prefer asserting the whole table, which covers count *and*
content in one assertion:

```abap
assert_equals( exp = VALUE string_table( ( `line 1` ) ( `line 2` ) ) act = output ).
```

**Overwrite the value under test before asserting it** — the single most common fake-test pattern in
generated code:

```abap
cut->determine_destination( ).
cut->destination = 'R8K_902'.                                                  " <-- overwrite
cl_abap_unit_assert=>assert_equals( act = cut->destination exp = 'R8K_902' ).   " tautology
```

It passes on an empty database, a stubbed-out method, or an unimplemented requirement. Variants:
overwriting the returned table with hardcoded rows before `assert_not_initial`, and asserting a class
constant against itself.

## Fixture-shaped false green

**A fixture that cannot produce the scenario the test name claims.** The double's dispatch silently ignores
the configured value, so the test exercises a different path than its name and message claim:

```abap
" in the double
CASE inject_error.
  WHEN 'E' OR 'A'.  e_return = VALUE #( ( type = inject_error ) ).
ENDCASE.

" in the test
METHOD settle_ok_on_type_s.
  stub->inject_error = 'S'.            " <-- no WHEN 'S', so e_return stays EMPTY
  cl_abap_unit_assert=>assert_true( act = cut->settle_deal( ... )
                                    msg = 'type S success message must not block settlement' ).
ENDMETHOD.
```

Green, real assertion, wrong reason: no S-typed message ever reaches the code under test, and the method is
a duplicate of the empty-return test. Distinguishable from an honest test only by reading the double. When
a double dispatches on a configured value, every value a test sets must have a branch — or the double
should fail loudly on an unknown one.

**Coincident fixture conditions.** When fixture data simultaneously satisfies several validation
conditions, no test can isolate a missing path — removing one check leaves the suite green because another
still fires. Fixtures must be minimally sufficient for each condition **independently**.

## The empirical discriminator

Substitute a deliberately broken implementation — a stub that returns initial values, a swapped key, a
deleted `COMMIT` — and run the suite. If it stays green, the suite is defective regardless of how many
tests it contains, and regardless of whether the assertions look real. This is the practical test for
"vacuous but honest" suites, which are invisible to every other check.

---

## Exception handling in tests

**Not asserting an exception?** Declare the test `FOR TESTING RAISING cx_static_check` and let any
unexpected exception fail the test. Keep the body on the happy path — do **not** wrap in
`TRY/CATCH ... fail( )`:

```abap
METHODS reads_entry FOR TESTING RAISING cx_static_check.

METHOD reads_entry.
  " When
  DATA(entry) = cut->read_something( ).
  " Then
  cl_abap_unit_assert=>assert_not_initial( entry ).
ENDMETHOD.
```

**Asserting an expected exception?** Call, then `fail( )` immediately after so a missing exception is loud;
catch the **specific** exception in the Then:

```abap
METHOD raises_when_display_forbidden.
  " Given
  forbid_display( ).
  " When
  TRY.
      cut->fetch_employee_data( VALUE #( ) ).
      cl_abap_unit_assert=>fail( 'Expected CX_NO_AUTHORIZATION was not raised' ).
      " Then
    CATCH cx_no_authorization.
  ENDTRY.
ENDMETHOD.
```

### Sibling exceptions — the recurring `WRN W020` warning

When the method under test declares **more than one static-check exception** in its `RAISING` clause,
catching only the one you assert is not enough for the syntax checker. It reasons statically over the
*declared* signature, not over which exception your double will actually raise, so every other declared
static-check exception must still be caught or declared:

> *"The exception `LCX_…` is not caught or declared in the RAISING clause of `<test_method>`."*
> (`WRN W020`, cannot be suppressed by pragma.)

**Fix:** add `RAISING cx_static_check` to the test method and catch only the specific exception you assert.
Every uncaught sibling is then *forwarded* — so an unexpected sibling fails the test loudly instead of
being swallowed.

```abap
" onboard_employee RAISING lcx_employee_error cx_no_authorization
METHOD denied_change_raises_no_auth FOR TESTING RAISING cx_static_check.   " forwards the sibling
  " Given - service wired with a denying authorization stub
  " When
  TRY.
      cut->onboard_employee( valid_input( ) ).
      cl_abap_unit_assert=>fail( 'Expected CX_NO_AUTHORIZATION was not raised' ).
      " Then
    CATCH cx_no_authorization.                "  lcx_employee_error is NOT caught here —
  ENDTRY.                                     "  RAISING cx_static_check covers it instead
ENDMETHOD.
```

Do **not** "fix" this by adding a second `CATCH lcx_employee_error.` that does nothing — that swallows the
sibling and reintroduces false green.

Most project exceptions inherit `cx_static_check`, so `RAISING cx_static_check` covers them. A genuinely
dynamic-check sibling (`cx_dynamic_check` / `cx_no_check` lineage) does not trigger W020 and needs no
declaration.

---

## Assertions

- **Few and focused** — assert exactly what the test is about. Asserting too much couples tests to
  unrelated features.
- **Right assert type** — `assert_equals` (matches type and reports diffs), `assert_false`,
  `assert_not_initial`, `assert_initial`. Avoid `assert_true( xsdbool( act = exp ) )` for value comparison.
  The exception is **object reference identity**, which `assert_equals` cannot express because it types
  `act`/`exp` as `SIMPLE` — there, `assert_true( xsdbool( ref1 = ref2 ) )` is correct.
- **Custom asserts** to remove duplication (a `column( )` or `assert_contains( )` helper) instead of
  copy-paste.
- **Assert content, not quantity** — `assert_contains_exactly( )` over a line count.
- **But assert *quality*, not content, when quality is what you mean.** This is the counterweight, and it
  is easy to over-correct past it. If the test is about a meta-property of the result, asserting exact
  content is *fragile*: a harmless rewording breaks a test that was never about the wording.

  ```abap
  " the test is about line length, so assert line length
  assert_all_lines_shorter_than( actual_lines = table expected_max_length = 80 ).
  ```

  **Applying the distinction:** assert exact content when the content *is* the contract — a formatted key,
  a status code, a mapped field. Assert a property when the content is incidental — user-facing message
  wording, log text, ordering the spec does not fix. A test asserting `'Deal 1010/1 does not exist.'`
  verbatim fails the day someone improves the wording; if what you mean is "the output names the deal and
  says it is missing", use `assert_char_cp( exp = '*1010/1*does not exist*' )` and say so.
