### What's changed in v1.3.0

* feat: promotion config refactor + pr mode detection + cleanup (#26) (by @patrickleet)

  * feat: preview_mode pr or push + dry run + test

  * fix: needs

  * fix: conflict

  * feat: summary diff + config

  * fix: improve dry run output

  * fix: coderabbit suggestions

  * fix: branch name sanitization

  * feat: default branch + native git push

  * fix: dry run file output

  * fix: branch_name tag

  * chore: more test scenarios

  * feat: native git push

  * feat: native git push + native yq

  * feat: multiple workflow comment support

  * fix: show dry run comments for all job types on PR events

  - Renamed comment steps from [Preview][PR] to [PR] since they now apply
    to all dry run scenarios, not just preview mode
  - Updated Set Comment Identifier condition to run when dry_run OR preview
    with pr mode
  - Updated Find Comment with same condition
  - Updated Dry Run Comment condition to simply check dry_run and PR event
  - This ensures all 4 test jobs create separate dry run comments

  Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>

  * feat: detect pr mode, config step

  * chore: dry run comment cleanup

  * feat: refactor - config script for branching logic in js, and simpler steps after

  * chore: update readme and job names

  * chore: readme table structure

  * chore: job name cleanup

  * chore: more branch sanitizing for length, default branch detection error handling

  * fix: image.tag preview vs. release

  * fix: set some env values in tests, not image tag, pipeline should handle that

  * fix: test promotion chart doesnt need quotes here

  * fix: cleanup

  * fix: env name test cleanup

  * feat: e2e tests

  * fix: job names

  * feat: auth mode: app/pat

  * fix: only one test can push to main cause race conditions

  ---------

  Co-authored-by: Claude Opus 4.5 <noreply@anthropic.com>

* fix: permissions (by @patrickleet)

* fix: permissions (by @patrickleet)


See full diff: [v1.2.0...v1.3.0](https://github.com/unbounded-tech/workflows-gitops/compare/v1.2.0...v1.3.0)
