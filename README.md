# Garmin + Claude MCP Guide

Garmin Connect 데이터를 Claude(Code / Desktop)에 MCP로 연결해서 러닝 코치처럼 쓰는 방법을 정리했습니다.
직접 세팅하고 여러 실패·제약을 겪으면서 얻은 내용이라, 공식 문서에 없는 함정 위주로 적었습니다.

## 왜 필요한가

Garmin은 공식 MCP 서버를 제공하지 않습니다. 공식 Garmin Connect API는 개발자 파트너 승인제라 개인이 바로 못 씁니다.
대신 [python-garminconnect](https://github.com/cyberjunky/python-garminconnect)(비공식, Garmin 모바일 앱과 같은 방식으로 로그인)를
기반으로 만든 커뮤니티 MCP 서버가 여러 개 있습니다. 이 가이드는 그중 가장 포괄적인
[Taxuspt/garmin_mcp](https://github.com/Taxuspt/garmin_mcp)(110개+ 도구, MIT 라이선스) 기준입니다.

> 참고로 **Strava는 2026-06-01부터 공식 MCP 커넥터**를 제공합니다(`https://mcp.strava.com/mcp`, 구독자 전용, 읽기 전용).
> Garmin에만 있는 수면·Body Battery·훈련 준비도 같은 지표는 없지만, 별도 설치 없이 모바일까지 바로 됩니다.
> 이런 지표가 필요 없다면 이 가이드 대신 Strava 커넥터만 써도 충분할 수 있습니다.

## 다른 선택지

- [leewnsdud/garmin-connect-mcp](https://github.com/leewnsdud/garmin-connect-mcp) — 더 작은 24개 도구 서버.
  **신발 마일리지 추적**, 응답 PII 자동 필터링이 장점이지만, 마지막 업데이트가 오래됐고(2026-02) 사용자층도 작습니다(⭐5).
  신발 마일리지만 필요하면 서버를 바꾸기보다 Garmin Connect 앱에서 기어(Gear)를 직접 등록하고
  이 가이드가 쓰는 서버의 `get_gear`/`get_activity_gear`로 조회하는 편이 낫습니다.

## 사전 준비

- macOS + [Homebrew](https://brew.sh)
- Garmin Connect 계정
- Claude Code 그리고/또는 Claude Desktop 앱

## 1. `uv` 설치

```bash
brew install uv
```

## 2. Garmin 로그인 (한 번만, 직접 터미널에서 실행)

**절대 이 단계를 AI 어시스턴트에게 대신 시키지 마세요.** 이메일/비밀번호를 직접 입력해야 하는데,
어시스턴트가 이 명령을 실행하면 당신의 자격증명이 대화 로그에 남을 수 있습니다.

```bash
uvx --python 3.12 --from git+https://github.com/Taxuspt/garmin_mcp garmin-mcp-auth
```

이메일/비밀번호(+MFA 코드가 있다면 그것도)를 물어봅니다. 성공하면 OAuth 토큰이
`~/.garminconnect` 에 저장됩니다. **이후로는 비밀번호를 어디에도 입력할 필요가 없습니다.**

- MFA 없는 계정은 `GARMIN_EMAIL`/`GARMIN_PASSWORD` 환경변수로 대체 가능하지만,
  보안상 위 대화형 방식을 권장합니다.
- 인증 중 `mobile+cffi returned 429 ... IP rate limited by Garmin` 경고가 뜰 수 있는데,
  서버가 다른 방식으로 자동 재시도합니다. 마지막에 `SUCCESS`만 나오면 문제없습니다.
- 토큰은 **약 6개월** 유효합니다. 만료되면 `garmin-mcp-auth --force-reauth`로 재인증하세요.

## 3. Claude Code에 등록

```bash
claude mcp add garmin -s user -- uvx --python 3.12 --from git+https://github.com/Taxuspt/garmin_mcp garmin-mcp
```

`-s user`로 등록하면 이 컴퓨터의 어느 프로젝트 폴더에서 Claude Code를 열어도 사용 가능합니다.

**등록·인증 직후에는 세션을 재시작해야 도구가 로드됩니다.** 이미 열려 있던 세션에는 반영되지 않습니다.

확인:

```bash
claude mcp get garmin   # Status: ✔ Connected 이어야 함
```

## 4. Claude Desktop 앱에 등록 (선택)

설정 파일: `~/Library/Application Support/Claude/claude_desktop_config.json`

기존 내용이 있을 수 있으니 **덮어쓰지 말고 `mcpServers` 항목만 병합**하세요:

```json
{
  "mcpServers": {
    "garmin": {
      "command": "/opt/homebrew/bin/uvx",
      "args": [
        "--python", "3.12",
        "--from", "git+https://github.com/Taxuspt/garmin_mcp",
        "garmin-mcp"
      ]
    }
  }
}
```

> ⚠️ **`command`는 반드시 절대경로**(`/opt/homebrew/bin/uvx`, `which uvx`로 확인)여야 합니다.
> Desktop 앱은 로그인 셸 환경 없이 실행되기 때문에 `uvx`라고만 적으면 PATH를 못 찾아 연결이 실패합니다.

Claude Code와 **같은 토큰**(`~/.garminconnect`)을 공유하므로 재인증은 필요 없습니다.
설정 저장 후 Desktop 앱을 **완전히 종료(⌘Q)했다가 재실행**해야 반영됩니다.

## 5. 사용

자연어로 물어보면 Claude가 알아서 관련 도구를 호출합니다.

```
최근 러닝 활동 5개 보여줘
이번 주 훈련 부하랑 회복 상태 어때?
지난 러닝 구간별 페이스랑 심박 분석해줘
예상 10K 기록 알려줘
```

활동/건강/훈련/워크아웃/기어 등 110개 이상의 도구가 등록됩니다. 처음엔 실제로 어떤 도구가 잡히는지
`/mcp` 또는 도구 목록에서 확인해보는 걸 추천합니다.

## 모바일에서는 어떻게 되나 — 이게 가장 큰 함정입니다

**iOS/Android Claude 앱은 로컬 MCP 서버를 쓸 수 없습니다.** Desktop 앱이 안 되는 게 아니라,
모바일 OS 자체가 앱이 임의의 백그라운드 프로세스를 실행하는 걸 허용하지 않기 때문입니다.
Claude 모바일 앱은 **원격(HTTP로 공개된) MCP 서버만** 붙일 수 있습니다. 이 서버는 로컬 stdio 프로세스라 해당 안 됩니다.

### 실제로 뭐가 일어나는지 (직접 겪은 함정)

모바일에서 가민 관련 질문을 하면, Claude가 도구를 호출하지 않고도 **이전 대화의 memory(회상)로 답을 만들어냅니다.**
문제는 이때 Claude가 **"방금 조회했다"고 자신 있게 틀리게 말할 수 있다는 것**입니다. 실제로 겪은 사례:

> 모바일 Claude: "네, 가능합니다... 방금 조회한 러닝 기록도 이 모바일 세션에서 정상적으로 가져온 것입니다."
> → 실제로는 도구를 호출한 적이 없었고, 예전 대화에 있던 숫자를 재구성해서 답한 것이었음

새 대화창을 열어도 재현됩니다. **모바일에서 나온 가민 관련 답변은 숫자가 정확해 보여도 검증 없이 믿지 마세요.**

### 대안 3가지

| 방법 | 설명 | 제약 |
|---|---|---|
| **Remote Control** | 맥에서 `/remote-control`(`/rc`)로 세션을 열고 모바일 앱으로 원격 조종. 실제 실행은 맥에서 일어나므로 로컬 MCP 그대로 사용 가능 | 맥이 켜져 있고 잠자기 상태가 아니어야 함 |
| **Strava 공식 MCP** | Strava가 직접 호스팅하는 원격 서버. `claude.ai` 커넥터로 등록하면 모바일에서 바로 됨 | Strava 구독 필요, 가민 고유 지표(수면·Body Battery 등) 없음, 읽기 전용 |
| **직접 원격 호스팅** | 이 서버를 `streamable-http` 모드로 띄우고 공개 | 비추천 — 아래 보안 항목 참고 |

## 보안 주의사항

- 이 서버엔 **자체 인증 기능이 없습니다.** URL을 아는 사람 누구나 접근 가능합니다.
- **쓰기 도구가 포함**돼 있습니다: `delete_workout`, `add_weigh_in`, `create_manual_activity`,
  `upload_workout` 등 — 단순 조회용 서버가 아닙니다.
- 그래서 **인터넷에 노출하지 않는 것을 기본으로** 하세요. 꼭 원격 호스팅이 필요하면:
  - 리버스 프록시 뒤에 두고 인증(bearer token 등) 계층을 반드시 추가
  - `GARMIN_ENABLED_TOOLS` 환경변수로 읽기 전용 도구만 허용
- `~/.garminconnect`, `~/.garminconnect_base64` 는 계정 접근 자격증명입니다. 커밋·공유 금지.

## 예전 기기(다른 웨어러블) 데이터를 합치고 싶다면

기기를 바꾼 적이 있다면(예: 다른 브랜드 워치 → Garmin), 예전 데이터는 이 MCP 서버로 못 가져옵니다.
그 기간 데이터를 활용하려면:

1. 예전 앱에서 데이터 내보내기(대개 CSV)
2. Claude Code로 직접 파싱해서 요약 마크다운 생성 — **원시 CSV를 Desktop/모바일 Claude가 직접 파싱하게 하지 마세요.**
   중첩 JSON 구조인 경우가 많아서 LLM이 잘못 읽고도 자신 있게 틀린 값을 낼 수 있습니다.
3. 정리된 요약 파일만 Google Drive 등에 올려서, 클라우드/모바일 Claude가 원본 대신 이 파일을 읽게 하기

이렇게 하면 모바일에서도 (memory 회상이 아니라) **실제 파일 내용**으로 정확하게 답하게 만들 수 있습니다.

> Drive API로 파일을 등록할 때 주의: 기존 파일을 그 자리에서 덮어쓰는 기능도, 삭제 기능도 없는 커넥터가 많습니다.
> 내용을 바꿀 때마다 새 파일이 생기니, 옛 버전을 수동으로 지우거나 애초에 자주 갱신하지 않는 편이 낫습니다.

## 유지보수

| 항목 | 방법 |
|---|---|
| 토큰 만료(~6개월) | `garmin-mcp-auth --force-reauth` 재실행 |
| 컨텍스트 절약 | `GARMIN_ENABLED_TOOLS`로 필요한 도구만 allowlist 지정 (오타난 이름은 조용히 무시되니 등록 후 실제 목록 확인) |
| 서버 제거 | `claude mcp remove garmin -s user` |

## 참고 링크

- [Taxuspt/garmin_mcp](https://github.com/Taxuspt/garmin_mcp) — 이 가이드가 사용한 MCP 서버
- [cyberjunky/python-garminconnect](https://github.com/cyberjunky/python-garminconnect) — 기반 라이브러리
- [Strava MCP Connector 안내](https://support.strava.com/en-us/articles/15401531-strava-mcp-connector)
- [Claude Code Remote Control 문서](https://code.claude.com/docs/en/remote-control)

## 라이선스

이 가이드 자체는 자유롭게 사용/수정하세요. 각 도구(garmin_mcp, python-garminconnect)의 라이선스는 해당 저장소를 따릅니다.
