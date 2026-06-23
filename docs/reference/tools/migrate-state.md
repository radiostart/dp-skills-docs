# `migrate-state.py`

> .agent-state.yml migration for existing dp-skills projects.
>
> Detects each project under `workspace/{TEAM}/projects/` (excluding `example/`)
> and creates `.agent-state.yml` (schema v1.3) if missing, or upgrades
> existing v1 / v1.1 / v1.2 files in-place to v1.3.
>
> Auto-detect rules (new file):
> - analyzed: True if `features/*.md` (excluding `.plan.md`) exists
> - tdd: True if `project.md` contains '**TDD 모드**'
> - domain: always null (user must confirm via /dp-skills:analyze)
> - phase: always 'development' (qa 전환은 /dp-skills:project --qa on 으로만)
>
> Upgrade rules:
> - v1   → v1.3 : preserve fields, add `domain: null` + `phase: development`, bump schema
> - v1.1 → v1.3 : preserve fields, add `phase: development`, bump schema
> - v1.2 → v1.3 : preserve fields, add `phase: development` (없을 때만), bump schema.
>                 재실행 시 무변경 (idempotent).
>
> Usage:
>     python3 migrate-state.py [WORKSPACE_PATH] [--apply]
>
>     WORKSPACE_PATH defaults to ./workspace (relative to CWD).
>     Without --apply it runs as dry-run and prints what would be created.

소스: [`tools/migrate-state.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/migrate-state.py)
