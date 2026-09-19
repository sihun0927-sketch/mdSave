# [이관] 대전 — OfficialDeliciousMap develop → governdeliciousmap

## 배경

- 데이터 주체는 governdeliciousmap(old)이다. OfficialDeliciousMap develop(new)에서 **old에 없는 행·상호만 추가**한다.
- new의 규칙(classify 식당 판정, 반복 게시 합침, expense 금액 보정)은 적용하지 않는다. old 규칙을 그대로 유지한다.
- 추가 전에 old 자체 규칙인 `parsers/place_name.py`의 `drop_personal`·`normalize_place`·`split_places`를 new 행에 적용한다.
- 작업 장소는 governdeliciousmap이다. 스크립트가 여기 있고 여기 파일을 쓴다. new의 `data/daejeon/`은 읽기 전용으로만 참조한다.
- 기간은 양쪽 모두 2026년 상반기(2026-01-01 ~ 2026-06-30)이다.

## 대상 파일

| old 파일 (유지) | 저장 형식 | 필드 변경 | 이관 내용 |
|---|---|---|---|
| `data/processed/records.csv` | CSV 유지 | 8열 그대로 | old에 없는 행만 끝에 추가 |
| `data/processed/geocode_cache.json` | JSON 유지 | 값 6키 그대로 | old에 없는 `{상호}\|대전` 키만 추가 |
| `data/manual/truncated_places_daejeon.csv` | CSV 유지 | 11열 그대로 | new에 `data/manual/daejeon/`이 없어 37행 그대로 |

## 기관 slug 변환표

| new `organization` | old `source` | 비고 |
|---|---|---|
| `daejeon-dong` | `daejeon_donggu` | |
| `daejeon-jung` | `daejeon_junggu` | |
| `daejeon-seo` | `daejeon_seogu` | |
| `daejeon-yuseong` | `daejeon_yuseong` | `-`→`_` |
| `daejeon-daedeok` | `daejeon_daedeok` | `-`→`_` |
| (없음) | `daejeon_city` | new csv에 시청 행 없음. old `daejeon_city` 행 그대로 |

## 스크립트 공통 구조 (`migrate_official.py --city daejeon`)

| 단계 | 함수 | 입력 (new develop) | 출력 (old) | 규칙 |
|---|---|---|---|---|
| 0 | `precheck(city)` | `data/daejeon/records.csv`, `geocode*.json`, `fetch.json` | 없음 | 하나라도 없으면 중단 |
| 1 | `load_records(city)` | `records.csv` 12열 | 메모리 | `amount_krw` 빈 행·`spent_on` 일 없는 행은 제외하고 건수만 보고 (expense 보정은 new 규칙이라 미적용) |
| 2 | `map_source(org)` | `organization` | `source` | 위 변환표 |
| 3 | `map_file(source_hash)` | `fetch.json` `sources[].path` | `file` | basename |
| 4 | `apply_old_rules(rows)` | 1~3 결과 | 메모리 | old의 `drop_personal`(개인·경조사·현금성 제거) → `normalize_place`(빈 값이면 제거) → `place_raw`=원본 |
| 5 | `append_records(city)` | 4 결과 | `data/processed/records.csv` | 키 `(date, source, place_raw 정규화, amount)`가 old에 없는 행만 끝에 추가. old `place`(정규화 후)로도 한 번 더 대조. old 행은 삭제·수정 없음 |
| 6 | `append_geocode(city)` | `geocode*.json` 분할 전부 + `category-lookup-v1.jsonl` | `geocode_cache.json` | 키 `{merchant}\|대전`이 old에 없고 주소가 `region_prefixes`+`districts` 안일 때만 추가. `place_url=""` |
| 7 | `append_manual(city)` | `data/manual/daejeon/*.jsonl` | `truncated_places_daejeon.csv` | new manual 없음 → 실행 안 함 |
| 8 | `report(city)` | 5~7 | stdout | 추가·제외·건너뜀 건수 |

## 대전 도시별 값 · 검증 값

| 단계 | 도시별 값 | 검증 값 (사용자 확인) |
|---|---|---|
| 1 | new 4,664행. 제외 0 (금액 빈 값 0, 일 없는 날짜 0) | 대상 4,664 |
| 2 | 5개 구청 slug 변환. new에 시청 없음 | 매핑 실패 0. old `daejeon_city` 그대로 |
| 3 | `fetch.json` sources 1,417건. 예 `daejeon/daejeon-dong/expenses-mayor/143190-….xlsx` | `file` 못 찾은 행 0 |
| 4~5 | old에 없는 행 4,000 (old 규칙 적용 전). old의 이 5개 구청 행이 965뿐이라 대부분 신규 | 추가 ≤ 4,000. old 대전 5,720행 그대로 |
| 6 | new 지오코딩 성공 2,847건 → 상호 1,555개. old 캐시 없음 1,074, 관할 주소 1,064 | 추가 ≤ 1,064. old `\|대전` 2,018키 그대로 |
| 7 | `data/manual/daejeon/` 없음 | 보정표 37행 그대로 |

기관별 old에 없는 행 (규칙 적용 전): 동구 828 · 중구 1,401 · 서구 721 · 유성구 18 · 대덕구 1,032.

## 필드 매핑

### `records.csv`

| old 항목 | new 출처 → 요소 | 변환 |
|---|---|---|
| `place` | `merchant` | old `normalize_place` 적용 |
| `place_raw` | `merchant` | 원본 그대로 |
| `amount` | `amount_krw` | 빈 값 0건 |
| `date` | `spent_on` | 전부 `YYYY-MM-DD` |
| `purpose` | `purpose` | 그대로 |
| `dept` | `department` | 그대로 |
| `source` | `organization` | 위 변환표 |
| `file` | `source_hash` → `fetch.json` `sources[].path` | basename |

### `geocode_cache.json`

| old 항목 | new 출처 → 요소 | 변환 |
|---|---|---|
| 키 `{상호}\|대전` | `geocode*.json` success `merchant` | old에 없는 키만 |
| `naver_name`, `address`, `lat`, `lng` | `lookup.candidates[]` 중 `latitude/longitude`가 결과와 같은 후보 | confirmation 0건이라 후보 매칭으로만 |
| `category` | `category-lookup-v1.jsonl` (1,341줄) `value.categories[].category` | 후보 `source.source_id`로 조인 |
| `place_url` | 없음 | 빈 문자열 |

## 검증 체크리스트

- [ ] 0단계: new `data/daejeon/records.csv`·`geocode.json`·`fetch.json` 존재 확인
- [ ] 1단계: 대상 4,664행, 제외 0건
- [ ] 2단계: slug 매핑 실패 0건, old `daejeon_city` 행 변동 없음
- [ ] 3단계: `file` 미해결 0건
- [ ] 5단계: 추가 ≤ 4,000행. 실행 전 old 대전 5,720행이 실행 후에도 그대로 남아 있음
- [ ] 5단계: 추가된 행에 `개인(성명 비공개)`·`쿠팡` 등 `drop_personal`·`normalize_place` 대상 없음
- [ ] 6단계: 추가 ≤ 1,064키. old `|대전` 2,018키 변동 없음. 추가 키의 주소가 전부 대전 5개 구
- [ ] 7단계: `truncated_places_daejeon.csv` 37행 그대로
- [ ] `python map_builder.py`로 대전 빌드 후 `dist/data/daejeon.*.json` 정상 생성

## 참조 파일 (OfficialDeliciousMap develop, 읽기 전용)

저장소: https://github.com/snowjaewon/OfficialDeliciousMap (브랜치 `develop`)

| 경로 | 링크 |
|---|---|
| `data/daejeon/records.csv` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/daejeon/records.csv |
| `data/daejeon/fetch.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/daejeon/fetch.json |
| `data/daejeon/geocode.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/daejeon/geocode.json |
| `data/daejeon/geocode.002.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/daejeon/geocode.002.json |
| `data/daejeon/category-lookup-v1.jsonl` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/daejeon/category-lookup-v1.jsonl |
| `src/deliciousmap/registry/daejeon.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/registry/daejeon.py |
| `src/deliciousmap/storage.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/storage.py |
| `src/deliciousmap/contracts.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/contracts.py |
