# Development Quality Prompts

> **Purpose:** Standardized prompts for AI-assisted development ensuring consistent
> code quality, documentation, and review processes.
>
> **Usage:** These prompts establish context, enforce quality standards, and guide systematic
> development workflows.
>
>

**Version:** 1.22  
**Last Updated:** 2026-02-02

---

## Create Supplemental Docs from PRD
```
Read the PRD and use the .cursor/rules/generate.mdc file to create the implementation plan and all other supporting documentation required to start the implementation.
```

## Start Project or Stage

```
Proceed with the implementation, following the PR workflow. At the end of each stage, test, allow me to verify, then commit, push, and merge.
```

## New Employee Orientation

```
Read PRD.md, README.md, .cursor/rules/coding-rules.mdc, all files in the docs folder.

```

## Exit Interview

```
I'm going to create a new session for the next stage, what info should I pass to the new agent?

```

## Create Supporting Docs

```
Use .cursor/rules/generate.mdc file to create the implementation plan and supporting documents for this project. Use PRD.md and README.md as the project definition.
```

## GitIgnore

```
Add hidden macos files, such as .DS_Store, to .gitignore and remove any of these files from the repo, both local and remote.
```

## Update Docs

```
Update the PRD, README, implementation plan, and other supporting documents with this updated information.
```

## Doc Review

```
Can you review the ../rg-platform/docs/product docs, and in this project, the PRD, the README, and all docs for completeness, clarity, and consistency between the docs and with this working implementation?
```

## Code Review

```
Review all of the source code and make a proposal to remove or fix:
- dead code
- deprecated code
- workarounds, backup solutions to the feature requested
- exceptions that prevent an issue from being reported
- fake data
- hard-coded URLs or port numbers. These belong in json config files or *.env files.
```

## Fix Issues

```
Can you write unit tests that expose the issues, fix the root causes, and verify them with the unit tests?
```

## Tech Stack

```
A "tech-stack.mdc" file has been created in .cursor/rules/. After reviewing it, I want you to review this project and make a proposal to:
- Update old versions of components this project is using.
- If updating is a problem, explain why it should not be updated.
- Add new tech stack components that have not been identified to the "tech-stack.mdc" file.
```

## Python Environment

```
source venv/bin/activate
```
