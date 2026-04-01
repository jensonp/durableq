# DurableQ Resume Signal

This document translates DurableQ into a repository signal that works for a resume review.

The goal is not to expand the system scope. The goal is to make the existing scope legible, credible, and defensible to a recruiter, hiring manager, or senior engineer who opens the GitHub repo from a resume link.

## What A Reviewer Looks For

In practice, a reviewer usually checks three things:

1. Can I understand what this project is in under a minute?
2. Can I see evidence that the author built and verified it like a real system?
3. Can I see enough discipline to trust that this was not a toy demo?

GitHub’s own docs support this framing: README files are surfaced prominently and are expected to explain what the project does, why it is useful, how to get started, where to get help, and who maintains it. GitHub also treats CONTRIBUTING, license, security policy, issue templates, and release notes as first-class repository signals.

## Priority 0: Must Be Visible Immediately

These are the highest-signal additions because they affect the first 30 to 60 seconds of review.

### 1. `README.md` Should Be A Landing Page, Not A Spec Dump

Keep the README focused on:

- one-sentence project summary
- what problem the system solves
- what guarantees it provides
- quickstart for local run
- one API example
- one architecture diagram or equivalent text diagram
- one pointer to deeper docs

Why this matters:
GitHub says READMEs are often the first item visitors see, and they should explain what the project does, why it is useful, how to get started, where to get help, and who maintains it. That is the single highest-leverage repo artifact for resume traffic.

Recommendation for DurableQ:
Keep the detailed technical spec in `resources.md` and `fundamentals.md`, but make the README compact, polished, and recruiter-readable.

### 2. CI Must Be Obvious And Passing

Add a GitHub Actions workflow that runs:

- formatting or lint checks
- unit tests
- PostgreSQL integration tests

Why this matters:
GitHub’s CI docs describe CI as continuously building and testing code on commit, and note that results are surfaced in pull requests. A public repo that visibly passes CI reads as materially stronger than one that merely claims to be tested.

Recommendation for DurableQ:
Add a visible CI badge in the README only if the workflow is stable and meaningful. A broken badge is worse than none.

### 3. The Repo Must Show The Core System Contract

Keep a compact, high-signal explanation of:

- at-least-once execution
- idempotent submission
- lease-based recovery
- stale completion prevention
- bounded retries

Why this matters:
A reviewer should not have to infer the engineering contract from code. Durable systems are judged by semantics as much as by implementation.

Recommendation for DurableQ:
Make the guarantee statement explicit in `README.md` and mirror the same terms in `ARCHITECTURE.md`.

## Priority 1: Strong Proof Of Engineering Discipline

These artifacts make the project look like something a serious backend engineer would maintain.

### 4. `ARCHITECTURE.md` Should Explain The Why

Include:

- component responsibilities
- state machine
- invariants
- failure modes
- tradeoffs
- why PostgreSQL is the source of truth
- why the chosen isolation and locking strategy is correct

Why this matters:
This is the document that converts implementation into reasoning. Senior reviewers look for explicit contracts, not just code paths.

Recommendation for DurableQ:
Tie every architectural choice back to one concrete failure mode it prevents.

### 5. `CONTRIBUTING.md` Signals The Repo Is Maintained Like A Real Project

Include:

- how to set up the environment
- how to run tests
- coding and PR expectations
- how to report issues
- any local debugging hints

Why this matters:
GitHub surfaces CONTRIBUTING files in the repository and community profile, and GitHub explicitly says contribution guidelines help collaborators create useful pull requests and issues.

Recommendation for DurableQ:
Even if the repo is solo-maintained, a CONTRIBUTING file signals that the project has a maintainer mindset rather than a throwaway mindset.

### 6. `LICENSE`, `SECURITY.md`, And Issue Templates Raise The Repo Quality Floor

Include:

- `LICENSE`
- `SECURITY.md`
- `.github/ISSUE_TEMPLATE/`
- `.github/pull_request_template.md`

Why this matters:
GitHub recommends adding a license for public repositories, says security policies tell people how to report vulnerabilities, and states that issue templates help produce higher-quality issues. These files are not decorative; they show project hygiene.

Recommendation for DurableQ:
Even if the project is not open to outside contributions immediately, the presence of these files makes the repo look treated, not abandoned.

### 7. Releases And Versioned Milestones Should Be Visible

Add:

- tags or release notes
- a simple changelog
- milestone or roadmap notes

Why this matters:
GitHub releases package software with notes and give a chronological history of versions. That makes progress legible to a reviewer and shows the project has an evolution, not just a snapshot.

Recommendation for DurableQ:
Use release notes to mark the first working submission path, first safe claim path, first recovery path, and first test pass.

## Priority 2: Strong Signal For Senior Engineers

These are especially persuasive to backend and systems reviewers because they prove the system was tested as a system.

### 8. Add A Runbook Or Operations Guide

Include:

- how to inspect `pg_stat_activity`
- how to inspect `pg_locks`
- how to run `EXPLAIN`
- how to diagnose stuck jobs
- how to simulate worker crash recovery

Why this matters:
The repository should show operational literacy, not just implementation literacy. A real backend system needs a diagnostic path.

Recommendation for DurableQ:
Create `RUNBOOK.md` or `OPERATIONS.md` and keep it concrete, command-oriented, and short.

### 9. Add A Test Matrix Or Verification Matrix

Include:

- claim safety
- stale completion rejection
- retry scheduling
- dead-lettering
- crash recovery
- idempotent submission
- queue metrics

Why this matters:
Hiring managers tend to trust projects that can be verified, especially when the behavior is concurrency-sensitive.

Recommendation for DurableQ:
Put the verification matrix in `README.md` or `TESTING.md` so a reviewer can see the proof obligations immediately.

### 10. Add Performance Evidence, But Only If It Is Honest

Include:

- benchmark methodology
- sample throughput or latency results
- what hardware and dataset shape were used
- known caveats

Why this matters:
Systems projects gain credibility from measurements, but only when the measurements are transparent. A synthetic number without methodology is weak signal.

Recommendation for DurableQ:
If you add performance numbers, prefer a small, reproducible benchmark over a flashy but irreproducible claim.

## Priority 3: Nice To Have, But Still Good Resume Material

These are not mandatory, but they sharpen the public repo.

### 11. ADRs Or A Decision Log

Use short architecture decision records for choices like:

- why PostgreSQL instead of Redis
- why `SKIP LOCKED`
- why at-least-once instead of exactly-once
- why no broker in v1

Why this matters:
Decision records show that tradeoffs were deliberate rather than accidental.

### 12. A Diagram Helps More Than Another Paragraph

Add a single diagram for:

- submission flow
- claim flow
- recovery flow
- lease expiration and re-claim

Why this matters:
Public repo visitors often scan visuals faster than prose. A clear diagram improves comprehension without broadening scope.

### 13. A Small Changelog Or Milestone Journal

Add a lightweight development history:

- what shipped in v0
- what was hard
- what was verified
- what remains deferred

Why this matters:
It gives the repo a sense of progression, which looks better than a one-shot dump of files.

## DurableQ-Specific Recommendations

If the focus is resume-driven, the best additions for DurableQ are:

1. Make `README.md` concise and recruiter-readable.
2. Add `ARCHITECTURE.md` with invariants, failure modes, and tradeoffs.
3. Add CI that runs PostgreSQL integration tests.
4. Add `CONTRIBUTING.md`, `LICENSE`, `SECURITY.md`, and issue templates.
5. Add `RUNBOOK.md` showing how to inspect `pg_locks`, `pg_stat_activity`, and `EXPLAIN`.
6. Add a verification matrix covering concurrency and failure cases.
7. Add versioned releases or milestones showing project evolution.

## What I Would Prioritize First

If you want maximum resume signal without changing the technical scope, do these first:

1. rewrite `README.md` into a crisp project landing page
2. add `ARCHITECTURE.md`
3. add CI with PostgreSQL integration tests
4. add `CONTRIBUTING.md`, `LICENSE`, `SECURITY.md`, and issue templates
5. add `RUNBOOK.md`

That sequence gives a reviewer immediate understanding, proof of engineering discipline, and evidence the project was treated like a real system.

## Source Notes

- GitHub README docs: [About the repository README file](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes?apiVersion=2022-11-28)
- GitHub CI docs: [Continuous integration](https://docs.github.com/en/actions/concepts/overview/continuous-integration)
- GitHub Actions docs: [GitHub Actions documentation](https://docs.github.com/en/actions/)
- GitHub contributing docs: [Setting guidelines for repository contributors](https://docs.github.com/en/articles/setting-guidelines-for-repository-contributors)
- GitHub healthy contributions docs: [Setting up your project for healthy contributions](https://docs.github.com/github/building-a-strong-community/setting-up-your-project-for-healthy-contributions)
- GitHub licenses docs: [Adding a license to a repository](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/adding-a-license-to-a-repository)
- GitHub security policy docs: [Adding a security policy to your repository](https://docs.github.com/en/enterprise-server%403.20/code-security/how-tos/report-and-fix-vulnerabilities/configure-vulnerability-reporting/adding-a-security-policy-to-your-repository)
- GitHub issue templates docs: [Using templates to encourage useful issues and pull requests](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests)
- GitHub releases docs: [About releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)
- GitHub Open Source Guides: [github/opensource.guide](https://github.com/github/opensource.guide)
