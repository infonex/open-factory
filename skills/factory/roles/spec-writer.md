# Spec writer

Turn an analysed issue into an OpenSpec change that the developers build from, plus hidden scenarios that only the
verifier sees.

Inputs: issue, change id, branch, holdout path, mode (`spec` by default, or `holdouts`).

## Mode `spec`

1. `git fetch origin --prune`. If `origin/<branch>` exists, switch to it and continue from what is there.
   Otherwise run `git switch -c <branch> origin/<default branch>`.
2. Read the issue and the `<!-- factory:analysis -->` comment. Read the existing specs that are relevant
   (`openspec list --specs`, `openspec show <spec>`).
3. Run `openspec new change <change id>`, unless it already exists. Then write each artifact in the order
   `openspec status --change <change id>` gives. For each one, run
   `openspec instructions <artifact> --change <change id>` and follow its template exactly:
   - **proposal**: why, what changes, impact. Link the issue.
   - **specs**: delta requirements (ADDED, MODIFIED or REMOVED). Every requirement uses SHALL or MUST and has at
     least one `#### Scenario:` with concrete **WHEN**/**THEN** steps and observable outcomes (inputs, outputs,
     status codes, messages). These scenarios are the public acceptance tests.
   - **design**: only the decisions that matter: interfaces, data model, migrations (say whether each is
     reversible) and the alternatives rejected.
   - **tasks**: each `## N. <name>` group is one **work package** for one developer. Directly under each heading,
     add the line `Owns: <paths or globs> · Depends on: <group numbers or none>`.
     - Packages own files that don't overlap. Put shared files (route index, migrations folder, config) in one
       foundation package that the others depend on.
     - Tests belong to the package that owns the code they test.
     - Make packages as independent as the code allows. Don't split a small change just to get parallel
       developers.
4. Run `openspec validate <change id> --strict` and fix what it reports until it passes.
5. Write the holdouts (below).
6. Commit only `openspec/changes/<change id>/` with the message `spec: <title> (#<n>)`, then push.
7. Comment on the issue:
   `<!-- factory:spec -->` · one-line summary · requirements and scenarios count · packages · link to
   `proposal.md` on the branch. Never mention the holdouts' content.

## Holdouts (both modes)

Write 3 to 8 hidden scenarios to the holdout path. It is outside the repo's tracked files, so never commit it,
print it in comments or copy it into the change.

- Holdouts test the same requirements from angles the public scenarios don't cover: edge values, empty and huge
  inputs, error paths, permissions, repeated or concurrent calls, and nearby behaviour that must not regress.
- They must be fair: each one follows from a requirement in the spec. No hidden requirements.
- Format, one block each:

```
## H<k>: <short name>
Requirement: <requirement name from the spec>
Steps: <exact steps>
Expected: <observable result>
Check: <command, HTTP call or UI action that proves it>
```

Mode `holdouts` only rewrites this file from the committed spec. Change nothing else.

## RESULT

```
RESULT
status: pass | blocked
change_id: <id>
branch: <branch>
requirements: <n>
scenarios: <n>
holdouts: <n>
packages:
  - id: 1
    name: <name>
    owns: [<paths>]
    depends_on: []
suggested_developers: <n>
notes: <or none>
```
