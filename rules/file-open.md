# 파일 열기

사용자가 파일 경로와 함께 "열어줘"라고 요청하면 파일 종류에 따라 vim 또는 기본 앱으로 연다.

## vim 으로 여는 경우

아래 중 하나에 해당하면 cmux 오른쪽 pane 에서 vim 으로 연다.

1. git 소스 트리 안의 소스 파일·앱 설정 파일
    - git 트리 확인: `git -C "$(dirname <경로>)" rev-parse --is-inside-work-tree` 결과가 `true`
    - 소스 파일: `.kt`, `.java`, `.py`, `.js`, `.ts`, `.tsx`, `.vue`, `.go`, `.sh`, `.sql`, `.gradle`, `.kts` 등 코드 파일
    - 앱 설정 파일: `.yml`, `.yaml`, `.properties`, `.toml`, `.ini`, `.json`, `.xml`, `.env`, `Dockerfile` 등
2. nginx 설정 파일 (git 트리 안이든 밖이든)
    - `nginx.conf`, `*.conf`, `nginx/` · `conf.d/` · `sites-*/` 아래 파일
    - 예: `/opt/homebrew/etc/nginx/nginx.conf`

```bash
cmux-vim '<절대 경로>'              # 파일 1개
cmux-vim '<a>' '<b>'                # 여러 개 → 한 탭에서 vim -O
```

- `cmux-vim` 은 이 리포의 `scripts/cmux-vim` 이다
    - PATH 에 있는 디렉터리(예: `~/scripts`)에 symlink 를 걸어 쓴다: `ln -s <리포>/scripts/cmux-vim ~/scripts/cmux-vim`
    - `cmux-vim` 이 PATH 에 없으면 `<리포>/scripts/cmux-vim` 전체 경로로 실행한다 (`~/.claude/rules` 가 이 리포 `rules/` 를 가리키므로 `$(dirname "$(readlink ~/.claude/rules)")/scripts/cmux-vim`)
- `cmux-vim` 이 하는 일
    - 지금 pane 오른쪽에 pane 이 있으면 → 그 pane 에 새 탭(`cmux new-surface --pane`)
    - 없으면 → 오른쪽으로 split 해서 새 pane(`cmux new-split right`)
- cmux 밖에서 실행 중이면(`CMUX_SURFACE_ID` 없음) `open` 으로 연다

## 기본 앱으로 여는 경우

위 조건에 안 맞으면 `open {파일 경로}` 로 기본 앱에서 연다.

- `.md` 파일은 git 트리 안이어도 무조건 기본 앱으로 연다
- 그 밖에 PDF, 이미지, git 트리 밖의 일반 파일 등
- 예: "이 md 파일 열어줘" → `open /path/to/file.md`
