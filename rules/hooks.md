# 훅 시스템

## 훅 유형

- **PreToolUse**: 도구 실행 전 (검증, 파라미터 수정)
- **PostToolUse**: 도구 실행 후 (자동 포맷, 검사)
- **Stop**: 세션 종료 시 (최종 검증)

## 현재 훅 (in ~/.claude/settings.json)

### PreToolUse
- **tmux 알림**: 장기 실행 명령어에 tmux 제안 (gradlew, mvn 등)
- **git push 검토**: push 전 Zed에서 검토 열기
- **문서 차단기**: 불필요한 .md/.txt 파일 생성 차단

### PostToolUse
- **PR 생성**: PR URL 및 GitHub Actions 상태 로깅
- **ktfmt**: 편집 후 Kotlin 파일 자동 포맷
- **Kotlin 컴파일 검사**: .kt 파일 편집 후 컴파일 확인
- **println 경고**: 편집된 파일에 println 경고

### Stop
- **println 감사**: 세션 종료 전 모든 수정된 파일에서 println 검사

## 자동 승인 권한

주의해서 사용:
- 신뢰할 수 있고 잘 정의된 계획에 활성화
- 탐색적 작업에는 비활성화
- dangerously-skip-permissions 플래그 절대 사용 금지
- 대신 `~/.claude.json`에서 `allowedTools` 설정

## TodoWrite 모범 사례

TodoWrite 도구 용도:
- 다단계 작업 진행 추적
- 지시 이해도 확인
- 실시간 조정 활성화
- 세부 구현 단계 표시

Todo 목록이 드러내는 것:
- 순서가 잘못된 단계
- 누락된 항목
- 불필요한 추가 항목
- 잘못된 세분화
- 잘못 해석된 요구사항
