# create-feature 위임 트림 — analyze/SKILL.md 경유 제거

> **For agentic workers:** task 단위 순차 실행, step 체크박스 추적. 동작 보존
> 최우선 — 지시 내용 불변, 위임 포인터만 직접 reference 링크로 교체.

**Goal:** `create-feature` step 4·5 의 `analyze/SKILL.md`(227줄) 위임을 없애고
동일 detail 을 담은 `analyze/references/*` 직접 링크로 바꾼다. 호출당 ~4k 토큰과
직렬 Read hop 1 을 줄이며, diff 는 neutral.

**무손실 근거:** ① step 4 의 bullet(5-1·5-2·6-1~6-3·6-4)이 analyze 5~6
스켈레톤을 이미 중복 보유한다. ② progressive-disclosure 이후 `analyze/SKILL.md`
도 detail 을 `project-md-update.md`·`agents-update.md` 로 링크만 하는 thin
오케스트레이터다(중간자). 따라서 중간자를 빼고 동일 reference 를 직접 링크하면
정보 동일, `analyze/SKILL.md` read 만 소거된다.

**Scope:** `create-feature` 단독. `project` step 7·8 의 analyze 위임은 `--url`
경로에서 analyze 전체 파이프라인(1~7)을 실제 실행하는 정당한 로드 — 제외.

**Tech:** markdown, `tools/docs_build.py`(reference 동기화), markdownlint,
pytest(tests/tools 회귀).

## 기준선

- plugin.json: 0.20.1 → 0.20.2 (patch)
- `create-feature` → `analyze/SKILL.md` 링크: line 101·119 (2건) → **0건** 목표.
  line 105 의 `domain-decision.md` 직접 링크는 유지.

## 검증 절차 (커밋 직전)

```bash
cd dp-skills
# 1. 상대 링크 유효성
python3 - <<'EOF'
import re, pathlib, sys
bad = []
for md in pathlib.Path("skills").rglob("*.md"):
    for m in re.finditer(r"\]\(([^)\s#]+)", md.read_text()):
        t = m.group(1)
        if t.startswith(("http", "mailto:", "${")):
            continue
        if not (md.parent / t).resolve().exists():
            bad.append(f"{md}: {t}")
print("\n".join(bad) or "links OK"); sys.exit(1 if bad else 0)
EOF
# 2. lint
npx -y markdownlint-cli "skills/**/*.md"
# 3. reference 재생성 (drift 방지 — 커밋 포함)
python3 tools/docs_build.py
# 4. 회귀
/opt/homebrew/bin/pytest tests/tools -q
```

## Task 1: create-feature step 4·5 위임 제거

**Files:** Modify `skills/create-feature/SKILL.md` · Regen
`docs/reference/skills/create-feature.md` · Modify `.claude-plugin/plugin.json`

- [ ] **Step 1** — step 4 opening(line 101): `analyze/SKILL.md` "5~6단계 수행"
  포인터를 제거하고 "features/ 전체 기준 갱신 + 항목별 reference 직접 따름,
  analyze/SKILL.md 는 읽지 않는다(동일 절차·소스만 features/)" 로 교체.
- [ ] **Step 2** — 5-1 bullet 말미에
  `갱신 예시: [project-md-update.md](../analyze/references/project-md-update.md).` 추가.
- [ ] **Step 3** — 5-2 bullet 말미에
  `표 기입·갱신 규칙: [project-md-update.md](../analyze/references/project-md-update.md).` 추가.
- [ ] **Step 4** — 6-1~6-3 bullet 말미에
  `상세 절차: [agents-update.md](../analyze/references/agents-update.md).` 추가.
- [ ] **Step 5** — step 5(line 119): `analyze/SKILL.md 의 6-5 단계 와 동일 —` 를
  `무결성 자동 검증 —` 으로. bash·ERROR/WARN 처리 불변.
- [ ] **Step 6** — `grep -n "analyze/SKILL.md" skills/create-feature/SKILL.md`
  → 0건 확인(`domain-decision.md` 직접 링크만 잔존).
- [ ] **Step 7** — 위 검증 절차 전체 통과.
- [ ] **Step 8** — plugin.json `0.20.2` + commit.

```bash
git add skills/create-feature/SKILL.md docs/reference .claude-plugin/plugin.json
git commit -m "refactor(create-feature): analyze/SKILL.md 위임 제거 → references 직접 링크 (0.20.2)"
```
