# Dooray RAG MCP

Dooray(NHN 프로젝트 관리) Task/Wiki 문서를 검색하는 RAG 시스템. `dooray-rag` MCP 서버로 연동.

## 도구

| 도구 | 용도 |
|------|------|
| `search` | 하이브리드 검색 (임베딩+BM25+RRF). 문서 제목, ID, URL, 미리보기 반환 |
| `get_document` | `search` 결과의 `doc_id`로 문서 전문 조회 |
| `get_status` | 크롤링/인덱싱 현황 (프로젝트별 상태, 문서 수) |
| `crawl` | 크롤링 실행 (mode: "tasks"/"wikis"/"all", force: bool) |
| `index` | 인덱싱 실행 (force: bool) |

## 검색 워크플로우

1. `search`로 관련 문서를 찾는다
2. 미리보기만으로 부족하면 `get_document`로 전문을 가져온다
3. 전문 내용을 바탕으로 사용자 질문에 답변한다

## 사용 시점

- 사용자가 Dooray 문서, Task, Wiki 내용을 물어볼 때
- NHN Cloud IaaS Compute 팀의 업무 지식, 이슈, 운영 가이드를 찾을 때
- "두레이에서 찾아줘", "관련 문서 있어?" 등의 요청

## 주의사항

- `search` 결과의 URL은 Dooray 웹 링크이므로 그대로 사용자에게 제공한다
- `crawl`과 `index`는 실행 시간이 길 수 있으므로 (수분) 사용자에게 미리 알린다
- 데이터가 오래된 것 같으면 `get_status`로 마지막 크롤링 시각을 확인한다
- 도구 호출이 실패하면 다른 방법으로 우회하지 말고 즉시 사용자에게 실패 사실과 에러 내용을 알린다
