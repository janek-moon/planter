# planter 🌱

[English](README.md) · **한국어**

<img width="1536" height="1024" alt="Image" src="https://github.com/user-attachments/assets/b71fc93e-2d49-4869-ac97-e5e1fe1e8769" />

Claude Code 세션의 작업을 이름이 지정된 **tmux** 또는 **cmux** 세션에 위임합니다 —
세션을 찾거나 새로 만들고, 그 안에서 `claude`(기본), `codex`, 또는 일반 셸을
실행한 뒤, 화면을 모니터링하고 결과를 다시 보고합니다.

## 설치

```
/plugin marketplace add janek-moon/planter
/plugin install planter@planter
```

## 스킬

| 스킬 | 역할 |
|---|---|
| `planter:tenant` | 진입점: tmux/cmux를 감지하고, 지정된 이름의 세션을 찾거나 만들어 적절한 러너로 연결 |
| `planter:tenant-claude` | 작업을 `claude` CLI로 실행 (기본 러너) |
| `planter:tenant-codex` | 작업을 `codex` CLI로 실행 |
| `planter:tenant-shell` | 완료 감지를 위한 센티넬과 함께 일반 셸 명령을 실행 |

## 사용법

Claude Code에게 이렇게 요청하면 됩니다:

- "**build** 세션에서 테스트 돌려줘"
- "**deploy**라는 세션을 만들고 거기서 claude로 lint 에러 고쳐줘"
- "`npm run build`를 build 세션에 일반 셸 명령으로 위임해줘"
- "이건 codex로 해줘"
- "보내기만 하고 신경 쓰지 마 (fire and forget)"

## 동작 방식

1. **감지(Detect)** — `$TMUX` / `$CMUX_WORKSPACE_ID` / 실행 중인 서버 탐지로 백엔드를 고릅니다. 아무것도 가정하지 않습니다.
2. **해석(Resolve)** — 지정한 이름의 세션/워크스페이스가 있으면 재사용하고, 없으면 만들어서 이름을 붙입니다.
3. **확인(Inspect)** — 키 입력을 보내기 전에 대상 화면을 먼저 캡처합니다. 사용 중인 창(pane)은 절대 덮어쓰지 않습니다.
4. **실행(Run)** — 러너가 claude/codex를 띄우거나, 센티넬로 감싼 셸 명령을 주입합니다.
5. **모니터링(Monitor)** — 완료 신호가 뜨고 출력이 안정될 때까지 화면을 폴링한 뒤 결과를 요약합니다. fire-and-forget이면 이 단계를 건너뜁니다.

## 안전 규칙

- tmux/cmux 서버를 스스로 부팅하지 않습니다.
- 다른 프로그램이 점유 중인 창 위에 타이핑하지 않습니다.
- 사용자의 명시적 허가 없이는 위임된 세션의 신뢰/권한/로그인 대화상자에 응답하지 않습니다.

## 요구 사항

- tmux 그리고/또는 cmux
- 플러그인을 지원하는 Claude Code, 해당 러너를 쓰려면 PATH에 `claude` / `codex` CLI

## 기여하기

이 저장소의 `main` 브랜치는 보호되어 있습니다. 직접 push할 수 없으며, 변경은 Pull Request로만 반영됩니다.

1. 저장소를 fork 하거나(외부 기여자) 새 브랜치를 만듭니다.
2. 변경 사항을 커밋하고 브랜치를 push합니다.
3. `main`을 대상으로 Pull Request를 엽니다.
4. 코드 오너(@janek-moon)의 승인 1개를 받아야 merge할 수 있습니다.

> 참고: GitHub 정책상 본인이 연 PR은 본인이 승인할 수 없습니다. 저장소 소유자는 관리자 권한으로 merge할 수 있습니다.

## 라이선스

[MIT](LICENSE)
