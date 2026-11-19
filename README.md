# JARVIS — AI 클라이언트가 함께 쓰는 클라우드 기억 서버

> 한성대학교 캡스톤 디자인 (2026.03–06), 3인 팀 프로젝트
> 정진환 담당: 기억 캡처 파이프라인, Bitemporal 데이터 모델

Claude, ChatGPT 같은 여러 AI 클라이언트가 **MCP(Model Context Protocol)** 로 하나의 장기 기억을 같이 쓰게 해 주는 서버입니다.
AI가 사용자와 나눈 대화를 "일기"처럼 서버에 기록하고, 나중에 "그때 뭐였지?"라고 물으면 관련 사실과 대화 맥락을 찾아 돌려줍니다.

## 핵심 아이디어

- **AI가 직접 일기를 쓴다**: 원본 대화 로그를 통째로 올리지 않습니다. AI 클라이언트가 자기 맥락에서 대화 턴, 요약, 키워드, 주제, 엔티티·사실·관계를 정리해 `jarvis_log_diary` 한 번으로 올립니다. 그래서 로컬 파일에 접근할 수 없는 앱에서도 똑같이 동작합니다.
- **사실을 덮어쓰지 않는다 (Bitemporal)**: 생각이 바뀌어도 옛 사실을 지우지 않습니다. 사실마다 `valid_from` / `valid_to` / `recorded_at`을 두어 "언제부터 언제까지 참이었는지"와 "언제 기록됐는지"를 따로 관리합니다. 마음이 바뀐 것 자체가 정보이기 때문입니다.
- **하이브리드 검색**: 벡터 검색(pgvector), 한국어 전문 검색(PGroonga), 엔티티 그래프 탐색 결과를 **RRF(Reciprocal Rank Fusion, k=60)** 로 합칩니다. 엔티티 그래프는 Leiden 커뮤니티로 묶어 recall 결과가 한쪽으로 쏠리지 않게 합니다.

## MCP 도구

| 용도 | 도구 |
|---|---|
| 기록 | `jarvis_log_diary` (기본), `jarvis_store_memory` (구버전) |
| 워크스페이스 | `jarvis_manage_workspace`, `jarvis_initialize_memory` |
| 회상 | `jarvis_recall_memory`, `jarvis_explore_topic` |
| 깊게 찾기 | `jarvis_search_passages`, `jarvis_search_episodes`, `jarvis_get_episode_excerpt`, `jarvis_follow_relation` |
| 화면 | `jarvis_open_ui` |

같은 기능을 REST API(`/api/v1/memory/...`)와 CLI(`jarvis init`, `jarvis recall <질문>`, `jarvis workspace ...`)로도 쓸 수 있습니다.

## 기술 스택

| 영역 | 사용 기술 |
|---|---|
| 서버 | Python 3.11, FastAPI, MCP Python SDK (FastMCP, streamable HTTP) |
| DB | PostgreSQL, pgvector, PGroonga, SQLAlchemy (async), Alembic |
| 임베딩 | `dragonkue/multilingual-e5-small-ko` (384차원, ONNX) |
| 그래프 | python-igraph, leidenalg |
| 실행 | Docker Compose |

## 폴더 구조

```
Jarvis/
├── src/jarvis/
│   ├── main.py            # FastAPI 앱, /health, /mcp 마운트
│   ├── mcp_adapter.py     # MCP 도구 정의
│   ├── api/v1/            # REST API (memory, workspaces, users)
│   ├── core/              # 기록·검색·회상 로직 (store, recall, retrieval, entity_resolution …)
│   ├── models/tables.py   # DB 스키마 (Bitemporal 사실·관계 테이블 포함)
│   ├── cli.py             # jarvis CLI
│   └── web/index.html     # 기억 열람 화면
├── alembic/               # DB 마이그레이션
├── scripts/               # 데이터 적재·검증 스크립트
├── tests/
└── docker-compose.yml
```

## 실행

```bash
docker compose up -d --build
```

- 서버: http://localhost:8002 (상태 확인: `/health`, MCP 엔드포인트: `/mcp`)
- DB: localhost:5440

> `docker-compose.yml`의 `server.volumes`에는 개발자 PC의 Claude Code 대화 폴더 경로가 들어 있습니다. 이 볼륨은 `/ingest-ledger` 기능에만 쓰이니, 자기 경로로 바꾸거나 해당 볼륨을 지우고 실행하세요.

MCP 클라이언트에는 `http://localhost:8002/mcp`를 서버 주소로 등록하면 됩니다.

## 현재 한계

- OAuth 인증은 코드는 있지만 MCP 쪽에서는 아직 꺼져 있습니다 (`mcp_adapter.py`의 TODO). 공개 서버로 띄울 때는 인증을 먼저 켜야 합니다.
- 테스트는 정규화·필터 등 일부 모듈만 있습니다.
