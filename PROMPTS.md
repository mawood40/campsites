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

## Create the PRD

```
Create a PRD in markdown (PRD.md) that defines a 3-level full-stack web application (front-end, middleware, back-end) for a membership site that displays campsites members have reviewed. I need a React SPA that connects to an API Gateway (edge) via a Python BFF, which connects to a Python backend service with a database. Use the FastAPI framework, Redis, and PostgreSQL. The artifacts will be deployed using Docker images. In the PRD, define the Product Overview, Architecture, Functional and Non-Functional Requirements, the Project Structure, and any other relevant sections. This project is intended to be written solely by an AI without human intervention. There must be no code changes between running the code for ve/test and production. Runtime configuration is only through .env and *.json files. The project must be architected for running on a single dev computer for development and testing. Yet individual components are deployed to separate front-end and back-end servers via SSH.
- The PRD is intended for AI to use when creating supplemental docs and for implementers. Create a README.md intended for human non-technical integrators.
- Users are first presented with a login and signup page. Users must supply their email address.
- Once logged in, they can see the SPA that shows campsites and their ratings. A user can add a new campsite via the “+” button, which lets them name the campsite, specify its location, set campsite pricing, add general info, and post a picture.
- Users can rate a campsite by selecting a thumbs-up or thumbs-down button and leaving comments.
- All user ratings for a campsite can be reviewed by selecting the campsite’s “Review” button
- The average review for a campsite is shown next to the campsite
- Two levels of users can be created, admin and subscriber
- Admin users can remove individual subscribers
- Create scripts for the following actions:
-- setup.sh: installs required programs and dependencies for building and testing the app
-- build.sh: script to build and package the 3 docker images, the default is a dev build  
-- test.sh: script to run unit and integration tests across the entire stack

```

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
