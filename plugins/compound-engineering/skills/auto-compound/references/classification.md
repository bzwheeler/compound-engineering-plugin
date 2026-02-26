# YAML Classification Reference

Maps PR signals to compound-docs YAML schema enum values.

## problem_type

Choose the primary category of the problem the PR solved.

| Value | When to use | Signals in PR |
|-------|------------|---------------|
| `build_error` | Compilation, bundle, build pipeline failures | "build fails", "compile error", CI failures, Dockerfile changes |
| `test_failure` | Broken or flaky tests | "test fails", "flaky", test file changes, fixture updates |
| `runtime_error` | Exceptions, crashes during execution | Stack traces, error handling, rescue blocks, nil checks |
| `performance_issue` | Slow queries, memory, N+1 | "slow", "N+1", "timeout", index additions, eager loading |
| `database_issue` | Migration, query, schema problems | Migration files, schema changes, query rewrites |
| `security_issue` | Auth, XSS, injection vulnerabilities | "vulnerability", "CVE", auth changes, input sanitization |
| `ui_bug` | Frontend visual or interaction issues | CSS/JS changes, template fixes, component updates |
| `integration_issue` | External service or API problems | HTTP client changes, API wrapper fixes, webhook handling |
| `logic_error` | Business logic bugs | Conditional fixes, calculation corrections, state machine |
| `developer_experience` | DX: workflow, tooling, dev setup | Script changes, dev config, seed data, local setup |
| `workflow_issue` | Process gaps, missing steps | Documentation of process, CI config, linting rules |
| `best_practice` | Patterns and practices | Refactoring toward a pattern, adding linting, conventions |
| `documentation_gap` | Missing or wrong docs | README, docstring, comment, or guide updates |

## component

Choose the primary component involved. Use the **generic values**
when the repo is not a Rails application.

### Framework-specific (Rails/Ruby)
| Value | When to use |
|-------|------------|
| `rails_model` | ActiveRecord model changes |
| `rails_controller` | Controller/action changes |
| `rails_view` | Template, ViewComponent changes |
| `service_object` | Service class changes |
| `background_job` | Sidekiq, Active Job, async task changes |
| `frontend_stimulus` | Stimulus JS controller changes |
| `hotwire_turbo` | Turbo Streams, Turbo Drive changes |
| `email_processing` | Mailer, email handler changes |
| `brief_system` | Brief/summary generation changes |
| `assistant` | AI assistant, prompt changes |
| `authentication` | Auth, Devise, session changes |
| `payments` | Stripe, billing, subscription changes |

### Generic (any language/framework)
| Value | When to use |
|-------|------------|
| `database` | Migration, schema, raw SQL, ORM query changes |
| `development_workflow` | Dev process, scripts, local setup |
| `testing_framework` | Test infrastructure, fixtures, helpers |
| `documentation` | README, guides, inline docs |
| `tooling` | Scripts, CLI tools, generators |
| `backend` | Server-side code not matching a specific category above |
| `frontend` | Client-side code not matching a specific category above |
| `api_endpoint` | REST/GraphQL endpoint, route, serializer changes |
| `infrastructure` | CI/CD, Docker, deployment, cloud config changes |
| `configuration` | App config, environment variables, settings changes |

**Mapping heuristic**: Look at the primary changed file paths:
- `app/models/` or `**/models/` -> `rails_model` or `backend`
- `app/controllers/` or `**/views/` or `**/routes` -> `rails_controller` or `api_endpoint`
- `db/migrate/` or `**/migrations/` -> `database`
- `app/services/` or `**/services/` -> `service_object` or `backend`
- `test/` or `spec/` or `tests/` -> `testing_framework`
- `*.js`, `*.ts`, `*.tsx`, `*.vue`, `*.svelte` -> `frontend`
- `.github/`, `Dockerfile`, `docker-compose`, `terraform` -> `infrastructure`
- `config/`, `.env`, `settings` -> `configuration`

## root_cause

Choose the fundamental cause of the problem.

| Value | When to use | Signals |
|-------|------------|---------|
| `missing_association` | Wrong or missing model relationships | Association added/fixed |
| `missing_include` | Missing eager loading (N+1) | `.includes()`, `.preload()` added |
| `missing_index` | Database performance: no index | Index migration added |
| `wrong_api` | Using deprecated or incorrect API | API call replaced |
| `scope_issue` | Incorrect query scope or filtering | Where clause changed |
| `thread_violation` | Thread-unsafe operation | Thread-safety fix |
| `async_timing` | Race condition, timing issue | Async/await fix, lock added |
| `memory_leak` | Memory growth or excessive allocation | Memory fix, GC tuning |
| `config_error` | Wrong configuration or env var | Config file changed |
| `logic_error` | Algorithm or conditional bug | If/else, calculation fix |
| `test_isolation` | Test pollution or fixture issue | Test setup/teardown fix |
| `missing_validation` | Input not validated | Validation added |
| `missing_permission` | Auth check missing | Permission check added |
| `missing_workflow_step` | Undocumented process step | Process docs added |
| `inadequate_documentation` | Docs wrong or missing | Docs updated |
| `missing_tooling` | No script or automation existed | Script/tool added |
| `incomplete_setup` | Missing config, seeds, fixtures | Setup files updated |

**Mapping heuristic**: Look at what the diff *added*:
- Added `.includes()` or `.preload()` -> `missing_include`
- Added `add_index` migration -> `missing_index`
- Changed `if`/`else` logic -> `logic_error`
- Added validation -> `missing_validation`
- Added auth check -> `missing_permission`
- Changed config file -> `config_error`

## resolution_type

Choose the type of fix applied.

| Value | When to use |
|-------|------------|
| `code_fix` | Changed application source code |
| `migration` | Database migration added |
| `config_change` | Configuration file changed |
| `test_fix` | Fixed broken or flaky tests |
| `dependency_update` | Updated a dependency version |
| `environment_setup` | Changed environment or infra config |
| `workflow_improvement` | Improved dev process or CI pipeline |
| `documentation_update` | Updated docs, README, guides |
| `tooling_addition` | Added script, generator, CLI tool |
| `seed_data_update` | Updated seed data or fixtures |

**Mapping heuristic**: What kind of files were changed?
- `*.rb`, `*.py`, `*.ts`, `*.go` (non-test) -> `code_fix`
- `db/migrate/`, `migrations/` -> `migration`
- `config/`, `.env`, `*.yml` (non-CI) -> `config_change`
- `test/`, `spec/`, `*_test.*` -> `test_fix`
- `Gemfile`, `package.json`, `requirements.txt` -> `dependency_update`
- `Dockerfile`, `.github/`, CI config -> `environment_setup`
- `README`, `docs/`, `*.md` -> `documentation_update`
- `scripts/`, `bin/`, `Makefile` -> `tooling_addition`

## severity

| Value | Criteria |
|-------|----------|
| `critical` | Production down, data loss, security breach, build completely broken |
| `high` | Core feature broken, significant security issue, blocking many users |
| `medium` | Specific feature impaired, performance degraded, non-critical bug |
| `low` | Edge case, minor inconvenience, cosmetic issue, dev-only |

**Mapping heuristic**:
- Labels `critical`, `urgent`, `P0`, `incident` -> `critical`
- Labels `P1`, `high-priority`, `security` -> `high`
- Labels `bug` with no priority -> `medium`
- Labels `minor`, `low`, `nice-to-have` -> `low`
- Large diff + many files + production fix -> likely `high`
- Small diff + single file + edge case -> likely `low`

## Category Directory Mapping

| problem_type | Directory |
|-------------|-----------|
| `build_error` | `docs/solutions/build-errors/` |
| `test_failure` | `docs/solutions/test-failures/` |
| `runtime_error` | `docs/solutions/runtime-errors/` |
| `performance_issue` | `docs/solutions/performance-issues/` |
| `database_issue` | `docs/solutions/database-issues/` |
| `security_issue` | `docs/solutions/security-issues/` |
| `ui_bug` | `docs/solutions/ui-bugs/` |
| `integration_issue` | `docs/solutions/integration-issues/` |
| `logic_error` | `docs/solutions/logic-errors/` |
| `developer_experience` | `docs/solutions/developer-experience/` |
| `workflow_issue` | `docs/solutions/workflow-issues/` |
| `best_practice` | `docs/solutions/best-practices/` |
| `documentation_gap` | `docs/solutions/documentation-gaps/` |
