# [이관] 울산 — OfficialDeliciousMap develop → governdeliciousmap

## 배경

- 데이터 주체는 governdeliciousmap(old)이다. OfficialDeliciousMap develop(new)에서는 **old에 없는 행·상호·보정만 추가**한다.
- new의 규칙(classify 식당 판정, 반복 게시 합침, expense 금액 보정)은 적용하지 않는다. old 행은 삭제·수정하지 않는다.
- 추가 전에 old 자체 규칙인 `parsers/place_name.py`의 `drop_personal` → `normalize_place` → `split_places`를 new 행에 적용한다. 상호가 `-`인 울산시청 행은 `normalize_place`에서 빠진다.
- 작업 장소는 governdeliciousmap이다. 스크립트도 여기에 두고 `parsers/place_name.py`를 import한다. new 저장소는 `data/ulsan/`을 읽기 전용으로 참조하며 수정하지 않는다.
- 기간은 양쪽 모두 2026년 상반기(2026-01-01 ~ 2026-06-30)다.

## 대상 파일 (old, 형식·필드 변경 없음)

| old 파일 | 형식 | 현재 | new 출처 (읽기 전용) |
|---|---|---|---|
| `data/processed/records.csv` | CSV 8열 | 울산 6,288행 (전체 265,612행) | `data/ulsan/records.csv` 6,078행, `data/ulsan/fetch.json` |
| `data/processed/geocode_cache.json` | JSON | `\|울산` 1,686키 | `data/ulsan/geocode.json` (+ `.002`, `.003` 분할 조각), `data/ulsan/category-lookup-v1.jsonl` 1,240줄 |
| `data/manual/truncated_places_ulsan.csv` | CSV 11열 | 61행 | `data/manual/ulsan/geocode.jsonl` 1줄 (`merchants.jsonl` 100줄은 대상 아님) |

## 기관 slug 변환표

| new `organization` | old `source` | new 행수 |
|---|---|---|
| `ulsan-city` | `ulsan_city` | 3,205 |
| `ulsan-junggu` | `ulsan_junggu` | 906 |
| `ulsan-namgu` | `ulsan_namgu` | 767 |
| `ulsan-donggu` | `ulsan_donggu` | 540 |
| `ulsan-bukgu` | `ulsan_bukgu` | 549 |
| `ulsan-ulju` | `ulsan_ulju` | 111 |

6개 전부 `-`→`_` 치환이다. old `cities.py`의 울산 `sources` 6개와 1:1 대응한다.

## 스크립트 공통 구조 (`migrate_official.py --city ulsan`)

| 단계 | 함수 | 입력 (new develop) | 출력 (old) | 규칙 |
|---|---|---|---|---|
| 0 | `precheck(city)` | `data/ulsan/records.csv`, `geocode*.json`, `fetch.json` | 없음 | 하나라도 없으면 중단 |
| 1 | `load_records(city)` | `records.csv` 12열 | 메모리 | `amount_krw` 빈 행·`spent_on` 일 없는 행은 **제외하고 건수만 보고** (expense 보정은 new 규칙이라 미적용) |
| 2 | `map_source(org)` | `organization` | `source` | 위 변환표 |
| 3 | `map_file(source_hash)` | `fetch.json` `sources[].path` | `file` | basename |
| 4 | `apply_old_rules(rows)` | 1~3 결과 | 메모리 | old의 `drop_personal`(개인·경조사·현금성 제거) → `normalize_place`(빈 값이면 제거) → `place_raw`=원본 |
| 5 | `append_records(city)` | 4 결과 | `data/processed/records.csv` | 키 `(date, source, place_raw 정규화, amount)`가 old에 **없는 행만 끝에 추가**. old `place`(정규화 후)로도 한 번 더 대조. old 행은 삭제·수정 없음 |
| 6 | `append_geocode(city)` | `geocode*.json` 분할 전부 + `category-lookup-v1.jsonl` | `geocode_cache.json` | 키 `{merchant}\|울산`이 old에 없고 주소가 `region_prefixes`(울산광역시·울산) + `districts`(중구·남구·동구·북구·울주군) 안일 때만 추가. `place_url=""` |
| 7 | `append_manual(city)` | `data/manual/ulsan/geocode.jsonl` | `truncated_places_ulsan.csv` | `잘린상호`가 old에 없을 때만 추가 |
| 8 | `report(city)` | 5~7 | stdout | 추가·제외·건너뜀 건수 |

## 울산 도시별 값 · 검증 값

| 단계 | 도시별 값 | 검증 값 (사용자 확인) |
|---|---|---|
| 1 | 6,078행. 제외: 금액 빈 256 → 대상 5,822. 일 없는 날짜 0 | 제외 256건 보고 |
| 2 | 6개 전부 `-`→`_` | 매핑 실패 0 |
| 4~5 | old에 없는 행 2,339 (규칙 적용 전, 키 `(date, source, 상호, amount)` 기준). 기관별: 중구 734·남구 608·동구 509·시청 337·북구 130·울주 21. 상호 `-`인 시청 행은 `normalize_place`에서 빠짐 | 추가 ≤ 2,339. old 울산 6,288행은 그대로 |
| 6 | new 좌표 성공 2,702건 → 상호 1,048개. old 캐시 없음 425, 그중 관할 주소 422 | 추가 ≤ 422. old `\|울산` 1,686키 그대로 |
| 7 | `geocode.jsonl` 1건 → `잘린상호`·`주소힌트`. `merchants.jsonl` 100건은 old에 대응 개념 없어 대상 아님 | 보정표 61 → 62행 |

## 필드 매핑

### `records.csv`

| old 열 | new 요소 | 변환 |
|---|---|---|
| `place` | `merchant` | old `normalize_place()` 적용 후 값. 빈 값이면 행 제외 |
| `place_raw` | `merchant` | 원본 그대로 |
| `amount` | `amount_krw` | 빈 값이면 행 제외 (256건) |
| `date` | `spent_on` | 전부 `YYYY-MM-DD` |
| `purpose` | `purpose` | 그대로 |
| `dept` | `department` | 그대로 |
| `source` | `organization` | 변환표 |
| `file` | `source_hash` → `fetch.json` `sources[].path` | basename (예 `182571-1.pdf`) |

### `geocode_cache.json` (키 `{상호}|울산`)

| old 키 | new 요소 | 변환 |
|---|---|---|
| `naver_name` | `confirmed_merchant`, 없으면 `lookup.candidates[]` 중 좌표 일치 후보의 `merchant` | 울산은 `confirmation` 0건이라 후보 매칭으로만 |
| `lat`, `lng` | `latitude`, `longitude` | 주소가 관할 밖이면 제외 |
| `address` | 좌표 일치 후보의 `address` | 그대로 |
| `category` | `category-lookup-v1.jsonl` `value.categories[].category` | 후보 `source.source_id`로 조인 |
| `place_url` | 없음 | 빈 문자열 |

### `truncated_places_ulsan.csv`

| old 열 | new 요소 | 변환 |
|---|---|---|
| `잘린상호` | `geocode.jsonl` `scope.merchant` | 그대로 |
| `주소힌트` | `geocode.jsonl` `address` | 그대로 |
| `정확한상호명`, `식당아님` | `restore.jsonl`, `classify.jsonl` 없음 | 비움 |

## 검증 체크리스트

- [ ] 0단계: `data/ulsan/records.csv`, `geocode.json`(+분할), `fetch.json` 존재 확인
- [ ] 1단계: 제외 256건(금액 빈 값) 보고, 일 없는 날짜 0건
- [ ] 2단계: 기관 매핑 실패 0건
- [ ] 3단계: `file` 미해결 0건 (해시 → path 전부 조회)
- [ ] 5단계: `records.csv` 추가 행 ≤ 2,339, old 기존 265,612행 변경 없음, 추가 행에 `개인`·`-` 상호 없음
- [ ] 6단계: `geocode_cache.json` 추가 키 ≤ 422, 기존 1,686키 변경 없음, 추가 주소 전부 울산 관할
- [ ] 7단계: `truncated_places_ulsan.csv` 61 → 62행
- [ ] `python map_builder.py`(울산) 실행 후 마커 수 ≥ 기존 1,233

## 사용자 결정 필요

- 금액이 빈 256행(new에서 `expense_amount_krw`에만 금액이 있는 행)을 **제외**하는 것으로 설계했다. new의 expense 규칙을 가져오지 않기 위해서다. 채워서 넣을지, 제외할지 결정 필요.
