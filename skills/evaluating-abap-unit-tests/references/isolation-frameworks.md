# Test isolation frameworks

Isolate every external dependency with SAP-supported tooling. Confirm exact API signatures against live
documentation before writing — signatures and release availability change.

Release floors: CDS/SQL TDF **7.52+** · `TEST-SEAM` **7.5+** · Function Module TDF **7.56+** ·
Authority Check test helper ≈ S/4HANA 2022+.

## Front door / back door

A test interacts with the object under test through the **front door** (its public API — direct input,
direct output) or the **back door** (calls to depended-on components — *indirect* input and output).
Controlling the front door is easy; the back door needs a double.

|  | Input | Output |
|---|---|---|
| **Direct** (front door) | calls to the object under test and their parameters | return values, exceptions |
| **Indirect** (back door) | responses *from* the depended-on component | calls *to* it and their parameters |

### The four roles a double can play

| Role | Purpose |
|---|---|
| **Stub** | control indirect *input* — make the dependency return what the test needs |
| **Mock** | verify indirect *output* — assert the dependency was called as expected |
| **Spy** | log indirect output — record calls for later inspection |
| **Dummy** | pretend existence — satisfy a parameter that is never used |

Not every technique supports every role. Verifying and logging indirect output is well supported by
hand-made doubles, the OO test double framework, and the FM framework — but **not** by the CDS or SQL
frameworks, because a doubled table has no call log. Consequence: to assert *that*
`BAPI_TRANSACTION_COMMIT` was called you need the FM framework's `verify( )` or a hand-rolled spy. No
database framework can tell you.

---

## 1. Database tables — ABAP SQL (OSQL) Test Double Framework

Create the environment once per class, clear doubles before each test, destroy at class end.

```abap
CLASS-DATA database TYPE REF TO if_osql_test_environment.

METHOD class_setup.
  database = cl_osql_test_environment=>create(
               i_dependency_list = VALUE #( ( 'ZEMPLOYEE_TABLE' ) ) ).
ENDMETHOD.

METHOD setup.
  database->clear_doubles( ).      " reset between tests
ENDMETHOD.

METHOD class_teardown.
  database->destroy( ).
ENDMETHOD.

" inside a test, Given:
database->insert_test_data( employees ).
```

- **Assert behaviour, not arrival.** Insert a matching *and* a non-matching row, then assert the
  non-matching row was filtered out. That proves the `WHERE`; "a row came back" does not.
- `create( )` is called **once per test class**, in `class_setup`. `clear_doubles( )` in `setup` gives
  each test fresh data.
- The parameter is `i_dependency_list`. A data source may be a table, view, CDS view, CDS table function,
  CDS view entity, CDS table entity, CDS projection view, or external view.
- **Do not populate `MANDT` in insert helpers.** Seeding the client field produces *"No component exists
  with the name MANDT"*. Declare `DATA rows TYPE STANDARD TABLE OF <table>.`, set only the business key
  fields, and let the framework handle the client.
- **Known limitation:** OSQL doubles do not reliably enforce primary-key uniqueness. An
  `INSERT ... raise-on-duplicate` path is therefore not coverable via the double — document it as a gap.
- **Double-redirection gotcha (silently wrong).** OSQL redirects *ABAP SQL* statements. If the same table
  is also read through a **CDS view**, that access still hits the real database — half the test runs
  against production data. When both access paths exist, create through the CDS framework and enable
  double redirection:

  ```abap
  environment = cl_cds_test_environment=>create(
                  i_for_entity      = 'CDS_VIEW'
                  i_dependency_list = VALUE #( ( type = 'TABLE' name = 'DB_TABLE' ) ) ).
  environment->enable_double_redirection( ).
  environment->insert_test_data( test_rows ).
  ```

- **Read vs. write:** the CDS framework covers read access only; the SQL framework covers read and write.
  A `MODIFY`/`INSERT`/`UPDATE` under test needs the SQL framework.

---

## 2. CDS views and table entities — ABAP CDS Test Double Framework

```abap
CLASS-DATA cds_environment TYPE REF TO if_cds_test_environment.

cds_environment = cl_cds_test_environment=>create( 'ZI_SOME_VIEW' ).
cds_environment->insert_test_data( double_data ).
cds_environment->clear_doubles( ).
cds_environment->destroy( ).
```

If the view carries DCL access control, the creation call has a flag to disable DCL for the test.

---

## 3. Authority checks — Authority Check Test Double Framework

Get the controller, build an authorization object set, restrict to it.

```abap
" Deny everything (negative path):
DATA(no_objects)  = VALUE cl_aunit_auth_check_types_def=>role_auth_objects( ).
DATA(restricted)  = VALUE cl_aunit_auth_check_types_def=>user_role_authorizations(
                      ( role_authorizations = no_objects ) ).
DATA(controller)  = cl_aunit_authority_check=>get_controller( ).
controller->restrict_authorizations_to(
  cl_aunit_authority_check=>create_auth_object_set( restricted ) ).
```

When the `AUTHORITY-CHECK` sits inline with no injectable checker, tests steer it by restricting to a
precisely built object set:

- **Pass set** — the *exact* authorization the production check requires (object plus every field and
  value). Restricting to it lets the real check pass.
- **Fail set** — the same object with a wrong value, so the real check fails and the production exception
  is raised.

```abap
DATA(pass_set) = cl_aunit_authority_check=>create_auth_object_set(
  VALUE cl_aunit_auth_check_types_def=>user_role_authorizations(
    ( role_authorizations = VALUE #(
        ( object         = 'S_TABU_DIS'
          authorizations = VALUE #( ( VALUE #(
              ( fieldname = 'ACTVT'     fieldvalues = VALUE #( ( lower_value = '02' ) ) )
              ( fieldname = 'DICBERCLS' fieldvalues = VALUE #( ( lower_value = '&NC&' ) ) ) ) ) ) ) ) ) ) ).
controller->restrict_authorizations_to( pass_set ).
controller->authorizations_expected_to( pass_execution = pass_set ).
" ... call CUT ...
controller->assert_expectations( ).
```

- Expectations can be positive and negative in the same call — the cleanest way to pin a check that must
  pass for one object and fail for another:

  ```abap
  auth_controller->authorizations_expected_to( pass_execution = objset_with_display_auth
                                               fail_execution = objset_with_create_auth ).
  " ... call CUT ...
  auth_controller->assert_expectations( ).
  ```

- `check_expectations( IMPORTING failed_expectations = ... )` is the non-failing variant.
  `get_auth_check_execution_log( )` then exposes `get_execution_status( )`, `get_failed_expectations( )`
  and `get_unexpected_executions( )`. Prefer `assert_expectations( )`; reach for the log only when diagnosing.
- The framework supports the `FOR USER` variant and multi-user configuration.

### Critical limitation — deny only, never grant

The framework can *remove* authorizations but cannot *grant* them. A pass set only takes effect if the
runner already holds that authorization; otherwise `restrict_authorizations_to` raises and the happy-path
test runs the **real** `AUTHORITY-CHECK` against the real user.

**The `skip( )` smell.** A test that `skip( )`s when the user lacks the role is not deterministic — it
silently asserts nothing on most runners. This is a symptom of the missing seam, not an acceptable pattern.

**Deterministic fix:** put the `AUTHORITY-CHECK` behind a local interface and inject the checker via an
optional constructor parameter; pass a granting stub in the test. The happy path then no longer depends on
the runner's roles.

Watch for the inverse defect in production: a `check_authorization` method that uses `RETURN` instead of
`RAISE cx_no_authorization` makes the declared exception dead and the subsequent `SELECT` unconditional.

---

## 4. Function modules — Function Module Test Double Framework

**ABAP 7.56+.** The alternatives are test seams (restricted) or wrapping every FM in an interface method
(prohibitive refactoring). This framework is API-driven, so **no production change is required** to isolate
a `CALL FUNCTION`.

### Lifecycle

```abap
CLASS-DATA function_env TYPE REF TO if_function_test_environment.

METHOD class_setup.
  " Naming every FM here is itself a check: the framework must resolve each one.
  function_env = cl_function_test_environment=>create(
                   VALUE #( ( 'ENQUEUE_E_VTBFHA' )
                            ( 'DEQUEUE_E_VTBFHA' )
                            ( 'BAPI_FTR_SETTLE' )
                            ( 'BAPI_TRANSACTION_COMMIT' )
                            ( 'BAPI_TRANSACTION_ROLLBACK' ) ) ).
ENDMETHOD.

METHOD setup.
  function_env->clear_doubles( ).      " reset ALL configurations before every test
ENDMETHOD.
```

Doubles created by `create( )` are valid **for the entire test session**. `clear_doubles( )` belongs in
**`setup`**, not only in `class_teardown` — otherwise configuration and invocation counts leak between
test methods.

**Configuration objects must be built inline.** The interface types returned by
`create_input_configuration( )` and `create_output_configuration( )` are not documented, so they cannot be
declared as `CLASS-DATA` or as a helper method's `RETURNING` type. Build them with `DATA(...)` inside the
test method. When one test needs the same configuration for both `when( )` and `verify( )`, hold it in a
single local variable rather than constructing it twice.

**Two environments in one include.** Two test classes (orchestrator and adapter) each need their own
`CLASS-DATA function_env`, so `create( )` runs twice. Because doubles are session-wide, the two FM lists
must be **disjoint**. The framework documents no `destroy( )`, so the first environment is still live when
the second is created — disjoint lists are the reason to expect this to work, not evidence that it does.
Verify on activation in your own system.

### Configuring behaviour

```abap
DATA(double) = function_env->get_double( 'BAPI_FTR_SETTLE' ).

DATA(input)  = double->create_input_configuration(
                 )->set_importing_parameter( name = 'COMPANYCODE'          value = '1010'
                 )->set_importing_parameter( name = 'FINANCIALTRANSACTION' value = '0000000001' ).

DATA(output) = double->create_output_configuration(
                 )->set_exporting_parameter( name = 'RETURNCOMPANYCODE' value = '1010'
                 )->set_table_parameter(     name = 'RETURN'            value = messages ).

double->configure_call( )->when( input )->then_set_output( output ).
```

| Parameter kind | Method | Config |
|---|---|---|
| IMPORTING | `set_importing_parameter( name = value = )` | input |
| EXPORTING | `set_exporting_parameter( name = value = )` | output |
| **TABLES** | **`set_table_parameter( name = value = )`** — singular `TABLE` | output |
| CHANGING | `set_changing_parameter( name = value = )` | either |

| Goal | Call |
|---|---|
| Return configured output | `->when( input )->then_set_output( output )` |
| Raise a class-based exception | `->when( input )->then_raise_exception( exception )` |
| Raise a classic exception (`EXCEPTIONS foreign_lock = 1`) | `->when( input )->then_raise_classic_exception( name )` |
| Custom logic per call | `->when( input )->then_answer( answer )` — implements `if_ftd_invocation_answer` |
| Different result on later calls | `->then_set_output( out )->then_raise_exception( ex )` |
| Repeat a behaviour n times | `->then_set_output( out )->for_times( n )` |
| Match any input | `->ignore_all_parameters( )->then_set_output( output )` |

`then_raise_classic_exception` is the one for `ENQUEUE_*`, BAPIs, and other classic-exception FMs — it
covers a lock-contention or error branch without a real lock.

**No configuration at all is a valid configuration:** naming an FM in `create( )` intercepts it, so
`BAPI_TRANSACTION_COMMIT` stops committing for free. That is what keeps the suite `RISK LEVEL HARMLESS`.
A `RISK LEVEL HARMLESS` suite that issues a real `BAPI_TRANSACTION_COMMIT` because the LUW modules were
never doubled is a serious defect.

### Verifying the double was called — use this for COMMIT / ROLLBACK

```abap
" Then - exactly one COMMIT, and no ROLLBACK
function_env->get_double( 'BAPI_TRANSACTION_COMMIT'   )->verify( )->is_called_once( ).
function_env->get_double( 'BAPI_TRANSACTION_ROLLBACK' )->verify( )->is_called_times( 0 ).

double->verify( input_config )->is_called_once( ).   " scoped to one argument set
double->verify( )->is_called_times( 2 ).             " any argument set
```

This is the Mock role, and the only way to prove a side-effecting FM ran. Covering the
`CALL FUNCTION 'BAPI_TRANSACTION_COMMIT'` line proves it executed — not that it executed on the right
branch or in the right order.

### What this framework does and does not prove

| Question | Answered? |
|---|---|
| Does the FM name exist? | **Yes.** No double intercepts an unregistered name, so the real call runs and a non-existent FM dumps with `CX_SY_DYN_CALL_ILLEGAL_FUNC`. |
| Do the parameter names bind? | **Yes** — an unknown name fails when configuring the double, and `CALL FUNCTION` binding fails at runtime. |
| Was COMMIT called on success and ROLLBACK on error? | **Yes**, via `verify( )`. |
| Did production call a *different but real* FM? | **Only via `verify( )`** — no dump occurs, so assert the expected double *was* invoked. |
| Did production pass the right parameter **values**? | **Only via `when( )` plus a scoped `verify( )`.** A swapped key or a hardcoded literal where a variable belongs is type-correct and binds cleanly. `ignore_all_parameters( )` leaves this defect class entirely invisible. |
| Does the FM do the right business thing? | **No.** That is integration testing. |

### Three defect classes, three different catchers

| Injected defect | Caught by | Needs an input configuration? |
|---|---|---|
| Fabricated FM name | `create( )` cannot resolve it, or the real call dumps `CX_SY_DYN_CALL_ILLEGAL_FUNC` | No |
| Wrong or missing parameter **name** | the ABAP runtime binds against the real interface *before* the double is consulted → `CX_SY_DYN_CALL_PARAM_*` | No — fires even under `ignore_all_parameters( )` |
| Wrong parameter **value** | the test, and only the test | **Yes** |

**A value mismatch does not raise.** When no configuration matches, the double does not error — it returns
initial values. The test then fails somewhere downstream ("expected 1 row, got 0"), which reads in the
failure log like a production logic defect rather than a wrong argument. Pair every `when( )` with a scoped
verify so the failure reports as what it is:

```abap
DATA(expected_call) = double->create_input_configuration(
                        )->set_importing_parameter( name = 'COMPANYCODE'          value = key-company_code
                        )->set_importing_parameter( name = 'FINANCIALTRANSACTION' value = key-deal_number ).

" ... drive the CUT ...

double->verify( expected_call )->is_called_once( ).
```

Pin values from the **same variable the test hands to the CUT**, so fixture and expectation cannot drift
apart. An input configuration constrains only the parameters it lists; unlisted ones stay free, which is
the right default for a parameter the requirement does not fix — copying a lock mode out of production into
the test would only prove production agrees with itself.

### Two conditions, both easy to get wrong

1. **The test must execute the production method.** Creating doubles proves nothing on its own — call
   `cut->settle( )`.
2. **Do not change the test to match suspect production code.** If configuring `RETURN_TABLE` fails because
   the real parameter is `RETURN`, the *production* `CALL FUNCTION` is wrong. Fix production, not the test.

### Verifying BAPI parameter names

`FUPARAREF` (SE16, `FUNCNAME = '<fm>'`) is the cheapest single-screen check for an unfamiliar FM: every
parameter with its kind (I/E/T/C) and optional flag in one view, faster than SE37 for this purpose.
BAPI families follow naming conventions consistently enough that a deviation is a signal — but a
convention is Inferred, not Confirmed. Check the specific FM before asserting a defect, and remember that
custom-coded FMs are bound by no convention at all.

---

## 5. General-purpose doubles — `cl_abap_testdouble`

For plain collaborator interfaces (no DB, auth, or FM), prefer the standard tool over hand-written doubles:

```abap
DATA(reader) = CAST <some_interface>( cl_abap_testdouble=>create( '<some_interface>' ) ).
cl_abap_testdouble=>configure_call( reader )->returning( expected_data ).
```

Keep it **data-in / data-out**. Do not build a "test-case-ID → `CASE`" framework inside doubles.

---

## 6. Hand-rolled doubles (`ltd_*`)

`cl_abap_testdouble` needs a **global** interface. Report-local `lif_*` interfaces therefore require a
hand-written double, which is fine and often clearer.

- **Name it `ltd_<thing>`**, not `ltc_*`. In a single-source report, keep `DEFINITION FOR TESTING` on it so
  it is generated only in test mode.
- **Data-in / data-out.** Public attributes the test sets; the double returns them.
- **A double can also be a spy.** Recording interactions is often the only way to assert ordering:

  ```abap
  DATA call_log TYPE string_table.       " in the double
  METHOD lif_deal_service~commit.
    APPEND `COMMIT` TO call_log.
  ENDMETHOD.
  ```

  ```abap
  " Then - one assertion covering sequence, skipping and exclusivity
  cl_abap_unit_assert=>assert_equals(
      exp = VALUE string_table( ( `EXISTS` ) ( `LOCK` ) ( `SETTLE` ) ( `COMMIT` ) ( `UNLOCK` ) )
      act = double->call_log
      msg = 'the LUW must be closed before the lock is released' ).
  ```

  This replaces a scatter of boolean `*_called` flags with one content assertion.

- **Spy counters prevent flag-shaped implementations from passing.** A connection lifecycle suite needs
  `create_count` / `close_count` attributes on the double; a Boolean `is_open` flag would otherwise satisfy
  every assertion.
- **A spy at the service boundary cannot see below it.** If `COMMIT` happens *inside* the adapter rather
  than being called through the interface, the spy is blind to it — use the FM framework instead. Diagnose
  which layer the call lives on before choosing.

> **Don't mock what isn't needed.** Pass `VALUE #( )` for unused dependencies. Value objects with no side
> effects (a transient log, for instance) can use their production version directly.

---

## 7. RAP business objects — BO Test Double Framework

For code that issues EML against a RAP BO. Two **mutually exclusive** variants:

| | `cl_botd_txbufdbl_bo_test_env` (transactional buffer double) | `cl_botd_mockemlapi_bo_test_env` (mock EML API) |
|---|---|---|
| Maintain instances across calls | yes | no |
| Partial / unknown EML input | yes | via `when_partial_input` / `ignore_input` |
| Negative tests beyond duplicate / not-found | only via complementary APIs | yes |
| Full control of each EML call | restricted — cannot force an operation to pass if the buffer state says it fails | yes |
| Internal `CREATE` | `insert_test_data` | yes |
| Internal numbering | `set_fields_handler` | yes |

Rule of thumb: **transactional buffer double** when the test is about state built across several EML
statements; **mock EML API** when it is about one call's behaviour, especially a failure you cannot reach
through real buffer state.

---

## 8. AMDP methods — AMDP Test Double Framework

SQLScript inside an AMDP is invisible to every framework above — an OSQL double does not intercept it,
because the code runs in the database, not in ABAP. `cl_amdp_test_environment` is the only way to test it,
and it is required whenever a **CDS table function** is in scope, since a table function's implementation
*is* an AMDP.

Build a **configuration**, then create the **environment** from it. The environment generates a test double
for each dependency and a test clone of each method under test.

```abap
CLASS-DATA amdp_environment TYPE REF TO if_amdp_test_environment.

METHOD class_setup.
  DATA(environment_config) = cl_amdp_test_environment=>create_test_configuration( ).

  environment_config->add_amdp_class( 'CL_AMDPTD_DEMO_FLIGHT_BOOKING'
                   )->add_methods_for_unit_test( VALUE #( ( 'read_flight_booking_status' )
                                                          ( 'set_booking_status_accepted' ) ) ).

  amdp_environment = cl_amdp_test_environment=>create( environment_config ).
ENDMETHOD.
```

- `add_methods_for_unit_test( )` removes *first-level* dependencies — the method's own logic in isolation.
  `add_methods_for_hierarchy_test( )` keeps the call hierarchy intact and doubles only the leaves. Both can
  be chained on the same class, and several classes can be configured in one test class.
- Declare the test methods `RAISING cx_static_check`; environment creation and AMDP calls are
  static-check-raising.

---

## In-system examples worth reading

- `CL_FTD_EXPENSE_MANAGER` — function module TDF, complete supported use cases.
- `CL_AMDPTD_DEMO_FLIGHT_BOOKING` — AMDP TDF.

Read these before inventing a configuration call.
