# [이관] 부산 — OfficialDeliciousMap develop → governdeliciousmap

## 배경

- 데이터 주체는 governdeliciousmap(old)이다. OfficialDeliciousMap develop(new)에서 **old에 없는 행·상호만 추가**한다.
- new의 규칙(classify 식당 판정, 반복 게시 합침, expense 금액 보정)은 이관하지 않는다. old 규칙만 유지한다.
- 추가 전에 old의 `parsers/place_name.py` 규칙(`drop_personal` · `normalize_place` · `split_places`)을 new 행에 적용한다.
- 작업 장소는 governdeliciousmap이다. 스크립트도 여기에 두고 `parsers/place_name.py`를 import해서 쓴다.
- new(`OfficialDeliciousMap` develop, `data/busan/`)는 읽기 전용으로만 참조하며 수정하지 않는다.

## 대상 파일 (old, 구조 변경 없음)

| old 파일 (유지) | 저장 형식 | 이관 방식 | new 출처 (읽기 전용) |
|---|---|---|---|
| `data/processed/records.csv` | CSV 8열 | old에 없는 행만 끝에 추가 | `data/busan/records.csv`, `data/busan/fetch.json` |
| `data/processed/geocode_cache.json` | JSON | old에 없는 키만 추가 | `data/busan/geocode.json` + 분할 조각, `data/busan/category-lookup-v1.jsonl` |
| `data/manual/truncated_places_busan.csv` | CSV | **대상 아님** | `data/manual/busan/` 없음 (old에도 파일 없음) |

`dist/data/busan.*.json`, `closure_report_busan.json`, `match_keys_busan.json`은 old 빌드가 재생성하므로 이관하지 않는다.

## 기관 slug 변환표 (new `organization` → old `source`)

| new slug | old source | old 표기 | new `records.csv` |
|---|---|---|---|
| `busan-city` | `busan_city` | 부산시청 | 있음 (4,382행) |
| `busan-jung` | `busan_junggu` | 중구청 | 있음 (711행) |
| `busan-seo` | `busan_seogu` | 서구청 | 있음 (464행) |
| `busan-dong` | `busan_donggu` | 동구청 | 있음 (401행) |
| `busan-busanjin` | `busan_busanjin` | 부산진구청 | 있음 (751행) |
| `busan-dongnae` | `busan_dongnae` | 동래구청 | 있음 (465행) |
| `busan-nam` | `busan_namgu` | 남구청 | 있음 (534행) |
| `busan-buk` | `busan_bukgu` | 북구청 | 있음 (357행) |
| `busan-haeundae` | `busan_haeundae` | 해운대구청 | 있음 (676행) |
| `busan-geumjeong` | `busan_geumjeong` | 금정구청 | 있음 (430행) |
| `busan-gangseo` | `busan_gangseo` | 강서구청 | 있음 (761행) |
| `busan-yeonje` | `busan_yeonje` | 연제구청 | 있음 (421행) |
| `busan-suyeong` | `busan_suyeong` | 수영구청 | 있음 (833행) |
| `busan-sasang` | `busan_sasang` | 사상구청 | 있음 (216행) |
| `busan-yeongdo` | `busan_yeongdo`, `busan_yeongdo_head` | 영도구청 | **없음** → old 그대로 |
| `busan-saha` | `busan_saha` | 사하구청 | **없음** → old 그대로 |
| `busan-gijang` | `busan_gijang` | 기장군청 | **없음** → old 그대로 |

규칙: `busan-jung/seo/dong/nam/buk`는 `_junggu/_seogu/_donggu/_namgu/_bukgu`로, 나머지는 `-`→`_`.

## 스크립트 공통 구조 (`migrate_official.py --city busan`)

| 단계 | 함수 | 입력 (new develop) | 출력 (old) | 규칙 |
|---|---|---|---|---|
| 0 | `precheck(city)` | `data/busan/records.csv`, `geocode*.json`, `fetch.json` | 없음 | 하나라도 없으면 중단 |
| 1 | `load_records(city)` | `records.csv` 12열 | 메모리 | `amount_krw` 빈 행·`spent_on` 일 없는 행은 **제외하고 건수만 보고** (expense 보정은 new 규칙이라 미적용) |
| 2 | `map_source(org)` | `organization` | `source` | 위 변환표 |
| 3 | `map_file(source_hash)` | `fetch.json` `sources[].path` | `file` | basename |
| 4 | `apply_old_rules(rows)` | 1~3 결과 | 메모리 | old의 `drop_personal`(개인·경조사·현금성 제거) → `normalize_place`(빈 값이면 제거) → `place_raw`=원본 |
| 5 | `append_records(city)` | 4 결과 | `data/processed/records.csv` | 키 `(date, source, place_raw 정규화, amount)`가 old에 **없는 행만 끝에 추가**. old 행은 삭제·수정 없음 |
| 6 | `append_geocode(city)` | `geocode*.json` 분할 전부 + `category-lookup-v1.jsonl` | `geocode_cache.json` | 키 `{merchant}\|부산`이 old에 없고 주소가 `region_prefixes`+`districts` 안일 때만 추가. `place_url=""` |
| 7 | `append_manual(city)` | `data/manual/busan/` | `truncated_places_busan.csv` | 부산은 manual 없음 → 건너뜀 |
| 8 | `report(city)` | 5~7 | stdout | 추가·제외·건너뜀 건수 |

## 부산 도시별 값 · 검증 값

| 단계 | 도시별 값 | 검증 값 (사용자 확인) |
|---|---|---|
| 1 | new 11,402행. 제외: 금액 빈 0 · 일 없음 0 | 대상 11,402 |
| 2 | 위 변환표 14개 매핑 | 매핑 실패 0. 영도·사하·기장은 old 그대로 |
| 3 | `fetch.json` sources 4,049개. 예 `busan/busan-city/expenses-mayor/21945-1.xlsx` | `file` 못 찾은 행 0 |
| 4~5 | old에 없는 행 4,134 (규칙 적용 전, 키 `일자+기관+금액`). 시청 1,361 · 강서 627 · 부산진 419 · 금정 275 · 해운대 247 | 추가 ≤ 4,134. `개인(성명 비공개)` 등은 `drop_personal`에서 빠짐. old 부산 11,781행은 그대로 |
| 6 | new 좌표 성공 4,318건 → 상호 1,935개. old 캐시 `\|부산` 4,512키(좌표 있는 상호 3,813). 공통 1,367 건너뜀. old에 없음 568, 그중 관할 주소 554 | 추가 ≤ 554. old 4,512키 그대로. `confirmation` 0건이라 `naver_name`·`address`는 `lookup.candidates[]` 좌표 일치 후보에서 |
| 7 | `data/manual/busan/` 없음 | 보정표 생성 안 함 |

## 필드 매핑

### `records.csv`

| old 열 | new 요소 | 변환 |
|---|---|---|
| `place` | `merchant` | old `normalize_place` 적용 후 값 |
| `place_raw` | `merchant` | 원본 그대로 |
| `amount` | `amount_krw` | 빈 값이면 행 제외 (부산 0건) |
| `date` | `spent_on` | `YYYY-MM-DD` 그대로 (부산 전부 10자) |
| `purpose` | `purpose` | 그대로 |
| `dept` | `department` | 그대로 |
| `source` | `organization` | 위 변환표 |
| `file` | `source_hash` → `fetch.json` `sources[].path` | basename |

### `geocode_cache.json`

| old 키/값 | new 요소 | 변환 |
|---|---|---|
| 키 `{상호}\|부산` | `geocode*.json` `results[].merchant` (status=success) | old에 없는 키만 |
| `naver_name` | `lookup.candidates[]` 중 `latitude/longitude`가 결과와 같은 후보의 `merchant` | confirmation 0건 |
| `lat`, `lng` | `latitude`, `longitude` | 주소가 부산 16개 구·군 밖이면 제외 |
| `address` | 좌표 일치 후보의 `address` | 그대로 |
| `category` | `category-lookup-v1.jsonl` (1,708줄) `value.categories[].category` | 후보 `source.source_id`로 조인 |
| `place_url` | 없음 | 빈 문자열 |

## 검증 체크리스트

- [ ] 0단계: `data/busan/records.csv`, `geocode.json`, `fetch.json` 존재 확인
- [ ] 1단계: 제외 건수 0 (금액 빈 0, 일 없음 0), 대상 11,402
- [ ] 2단계: slug 매핑 실패 0. 영도·사하·기장 old 행 변동 없음
- [ ] 3단계: `file` 못 찾은 행 0
- [ ] 4단계: `drop_personal`·`normalize_place` 적용 후 제거 건수 보고
- [ ] 5단계: `records.csv` 추가 ≤ 4,134. 기존 265,612행 삭제·수정 0
- [ ] 6단계: `geocode_cache.json` 추가 ≤ 554. 기존 `|부산` 4,512키 변동 없음. 추가된 주소 전부 부산 관할
- [ ] 7단계: 보정표 생성·수정 없음
- [ ] 8단계: 보고 건수가 위 검증 값과 일치
- [ ] `python map_builder.py` 부산 빌드 성공, 마커 수·레코드 수 변화 기록

## 참조 파일 (OfficialDeliciousMap develop, 읽기 전용)

저장소: https://github.com/snowjaewon/OfficialDeliciousMap (브랜치 `develop`)

| 경로 | 링크 |
|---|---|
| `data/busan/records.csv` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/busan/records.csv |
| `data/busan/fetch.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/busan/fetch.json |
| `data/busan/geocode.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/busan/geocode.json |
| `data/busan/category-lookup-v1.jsonl` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/busan/category-lookup-v1.jsonl |
| `src/deliciousmap/registry/busan.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/registry/busan.py |
| `src/deliciousmap/storage.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/storage.py |
| `src/deliciousmap/contracts.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/contracts.py |
