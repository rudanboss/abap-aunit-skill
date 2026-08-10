# Evaluating ABAP Unit Tests

AI agent skills for SAP ABAP development, published for the
[SAP AI Skills Library](https://github.com/SAP/ai-skills-library).

## Skills

| Slug | Description |
|---|---|
| [`evaluating-abap-unit-tests`](skills/evaluating-abap-unit-tests/SKILL.md) | Evaluate, write, and remediate ABAP Unit (AUnit) test suites — isolation frameworks, false-green detection, coverage strategy, testability seams. |

## Installation

Install with the [skills CLI](https://github.com/vercel-labs/skills), which supports Claude Code, Codex,
Cursor, OpenCode and other agents:

```bash
# all skills in this repo
npx skills add rudanboss/abap-aunit-skill

# a specific skill
npx skills add rudanboss/abap-aunit-skill --skill evaluating-abap-unit-tests
```

## What `evaluating-abap-unit-tests` covers

- **Isolation** — choosing and configuring the ABAP SQL (OSQL), CDS, Authority Check, Function Module,
  AMDP, and RAP BO test double frameworks, with the gotchas that cost an activation cycle each.
- **False green** — the catalogue of patterns that make a test pass for the wrong reason, and the empirical
  discriminator (substitute a broken implementation and see whether the suite goes red).
- **Coverage** — the seam ceiling, what is structurally uncoverable in classic ABAP, and how to read
  statement vs. branch vs. procedure coverage honestly.
- **Seams** — the minimal local-interface-plus-optional-constructor-parameter pattern, and why `TEST-SEAM`
  and subclass-to-mock are not it.
- **Review discipline** — separating a coverage fact from a defect claim, and classifying every assertion
  as Confirmed, Inferred, or Unknown.

Target stack: classic ABAP 7.5x+ and ABAP Cloud. Release floors are stated per framework in the skill.

## Scope and limitations

- The skill encodes conventions and framework mechanics, not system-specific knowledge. It will not tell
  you whether a particular function module or DDIC type exists in *your* system — it tells you how to
  check, and insists you do.
- Naming conventions (`i_`/`e_`/`c_` parameter prefixes, `const_` constants) reflect one house style that
  deliberately departs from Clean ABAP on those points. Adapt them if your project differs; everything else
  follows Clean ABAP.
- Coverage thresholds are treated as project policy, not as a value the skill asserts.

## Provenance

This skill was authored with AI assistance and reviewed by a human maintainer, who
takes responsibility for its content. It is grounded in publicly documented SAP
framework behaviour and contains no proprietary, customer-specific, or SAP-internal
material. Disclosed in line with SAP's guidance on AI-generated contributions.

## Related skills

`likweitan/abap-skills` includes an `abap-unit-testing` skill covering assertion
reference, manual mocks, and the CDS, OSQL, and RAP BO environments. This skill is
complementary rather than overlapping: it concentrates on whether a suite discriminates
a broken implementation from a correct one, adds the Function Module and AMDP test
double frameworks, and covers coverage-ceiling analysis and review adjudication.

## Author

Rudramani Pandey

## License

Apache-2.0. See [LICENSE](LICENSE). This repository is
[REUSE](https://reuse.software)-compliant — copyright and licensing are declared for
every file via `REUSE.toml`, verified with `reuse lint` against REUSE specification 3.3.
See [NOTICE](NOTICE) for third-party and trademark statements.

Portions of the guidance derive from publicly documented SAP framework behaviour (ABAP Development Tools
user guide, Clean ABAP style guide). SAP, ABAP, and S/4HANA are trademarks of SAP SE. This repository is not
affiliated with or endorsed by SAP SE.
