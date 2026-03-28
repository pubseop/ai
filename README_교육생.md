# IDR 교육용 Cursor 규칙 패키지 — 적용 안내

이 폴더는 **강의·실습용**으로 묶은 규칙 레이어입니다. 전체 `lis_cursor` 저장소가 아니며, **비밀·운영 키·`.env` 실값은 포함하지 않습니다.**

## 버전

- `VERSION.txt`에 `규칙 패키지 rYYYYMMDD` 형식으로 빌드 날짜가 적혀 있습니다.
- 강의 안내 URL과 **같은 회차**인지 확인하세요.

## 대상 환경

- **Cursor** 기반 에디터(강의에서 안내하는 버전 권장).
- 적용 위치: **실습용 프로젝트 루트**(본인이 클론하거나 만든 폴더의 최상위).

## 압축 해제 후 구조(요지)

- `.cursor/rules/` — Cursor Rules (`.mdc`)
- `.cursor/skills/idr-session-workflow/` — 세션 워크플로 요약 스킬
- `cursorrules.example` — 저장소 루트 `.cursorrules` 예시(프로젝트에 맞게 발췌·수정 후 사용)
- `docs/rules/` — 원문 규칙 MD(참고·오프라인 열람)

## 적용 절차

1. ZIP을 풀면 보통 **`idr-cursor-rules-student`** 한 폴더가 생깁니다. 그 안이 패키지 루트입니다.
2. 실습 프로젝트 루트에서 **백업** 후 진행합니다(이미 `.cursorrules` 등이 있을 수 있음).
3. **Rules**
   - `idr-cursor-rules-student/.cursor/rules/git-commit-korean.mdc` → 실습 프로젝트의 `.cursor/rules/`에 복사합니다.
   - `.cursor/rules` 폴더가 없으면 만듭니다.
4. **Skill(요약본)**
   - 실습 프로젝트에 `.cursor/skills/idr-session-workflow/` 디렉터리를 만들고, 같은 이름으로 `SKILL.md`를 복사합니다.
5. **프로젝트 루트 규칙(선택)**
   - `cursorrules.example` 내용을 검토한 뒤, 필요한 부분만 실습 프로젝트 루트의 `.cursorrules`에 반영하거나, 전체를 복사해 이름을 `.cursorrules`로 바꿉니다.
   - 일부 문단은 **IDR 전용 저장소**를 전제로 하므로, 팀·강사 지시에 맞게 줄이는 것이 좋습니다.
6. **문서 참고(선택)**
   - `docs/rules/` 아래 MD는 통째로 실습 프로젝트의 `docs/rules/` 등에 복사해 두면 오프라인으로 읽기 좋습니다. 경로는 팀 규칙에 맞게 조정해도 됩니다.
7. **Cursor 재시작** 또는 규칙/스킬이 다시 로드되도록 한 번 창을 닫았다가 엽니다.

## 하지 말 것

- 이 패키지에 **API 키·비밀번호·실서버 URL에 붙인 토큰**을 적어 넣어 공유하지 마세요.
- `.env` 실파일은 Git에 올리지 마세요(강의에서 안내하는 템플릿 파일명만 사용).

## 무결성(선택)

- 같은 경로에 제공되는 `idr-cursor-rules-student.zip.sha256`으로 ZIP 해시를 확인할 수 있습니다.

## 문제가 있을 때

- 브라우저에서 내려받은 ZIP이 깨졌는지, 압축 해제 도구 오류인지 확인합니다.
- 규칙이 반영되지 않으면 **경로**(프로젝트 루트인지), **파일명**(`.cursorrules` 등), **Cursor 재시작** 여부를 다시 봅니다.
- 그래도 안 되면 **강사**에게 질문하세요.
