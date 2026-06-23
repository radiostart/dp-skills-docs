# `doctor.py`

> dp-skills doctor — workspace / team / project 정합성 검사.
>
> 본 파일은 thin dispatcher. 실제 검사 로직은 `tools/doctor/` 패키지에 있다.
>
>   - default 모드     → doctor.integrity.run_integrity_check
>   - --fix            → 위에 포함 (run_integrity_check 인자)
>   - --schema         → doctor.schema.run_schema_check
>   - --diagnose       → doctor.diagnose.run_diagnose
>
> Usage:
>     python3 doctor.py [WORKSPACE_PATH] [--project PROJECT] [--fix]
>     python3 doctor.py --schema
>     python3 doctor.py [WORKSPACE_PATH] --diagnose [--project PROJECT]
>
> backward compat:
>     test/외부 코드가 `importlib.util.spec_from_file_location` 으로 본 파일을
>     로드해도 기존 함수명 (`check_gitignore_required_patterns`, `_git_repo_root` 등)
>     이 그대로 노출되도록 4 모듈을 wildcard re-export 한다.

소스: [`tools/doctor.py`](https://github.com/dealicious-inc/deali-skills-plugin/blob/main/dp-skills/tools/doctor.py)
