# Separate repos with shared docs pushed down from kitty

Kitty's specs will grow with its features, and we want each service's specs readable alone and agent context small. So the code lives in one repo per service, `kitty-backend` and `kitty-telegram-bot`, and any later Channel gets its own repo. The `kitty` repo holds anything about the business or affecting more than one repo. Each service repo keeps its own copy of the glossary terms and ADRs it uses, with definitions word for word, and agents working in it never read another repo. Shared changes merge in `kitty` first and then reach each affected service through a ticket that carries the full requirement. This costs two or more PRs for a contract change, duplicated CI and deploy config, and copies that can drift, which we check by a PR template checklist and not by automation. In return, a service's repo is enough to work in.

## Considered Options

- A monorepo with one folder per service. It makes a contract change one PR, but all specs and docs sit in one tree.
- Two repos, with `kitty` becoming the backend. Shared docs would sit next to backend specs.
- Side-by-side clones with `CLAUDE.md` pointers to `kitty`, or a git submodule. Both keep one copy of each doc, but a service would depend on another repo to be understood.
