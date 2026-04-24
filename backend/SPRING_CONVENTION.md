# dev-conventions

개인 개발 컨벤션 모음 레포지토리.
새 프로젝트 시작 시 해당 스택의 규칙 파일을 프로젝트 루트에 복사해서 사용한다.

---

## 구조

```
dev-conventions/
└── spring/
    ├── CLAUDE.md     # Spring Boot 코딩 규칙 (AI 코드 생성용)
    └── PROJECT.md    # 프로젝트별 정보 템플릿 (예정)
```

---

## 사용 방법

### AI 코딩 어시스턴트에 컨텍스트로 제공

**Claude Code**
```bash
# 프로젝트 루트에 복사
cp path/to/dev-conventions/spring/CLAUDE.md ./CLAUDE.md
```

Claude Code는 프로젝트 루트의 `CLAUDE.md`를 자동으로 읽는다.

**그 외 AI 도구 (Gemini, Codex 등)**
```bash
cp path/to/dev-conventions/spring/CLAUDE.md ./GEMINI.md
cp path/to/dev-conventions/spring/CLAUDE.md ./CODEX.md
```

### PROJECT.md 함께 사용하기

`CLAUDE.md`는 스택 공통 규칙이고, `PROJECT.md`는 프로젝트마다 새로 작성한다.
두 파일을 함께 AI에 제공하면 프로젝트 맥락까지 반영된 코드를 생성할 수 있다.

---

## 스택별 규칙 파일

| 스택 | 파일 | 설명 |
|---|---|---|
| Spring Boot | `spring/CLAUDE.md` | 디렉토리 구조, 레이어 규칙, 예외 처리, 테스트 전략, 코딩 컨벤션 |

---

## 업데이트 기록

프로젝트 경험이 쌓이면서 규칙이 바뀌는 경우 커밋 메시지에 변경 이유를 함께 기록한다.