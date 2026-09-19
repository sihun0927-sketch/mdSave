# [이관] 대구 — OfficialDeliciousMap develop → governdeliciousmap

## 배경

- 데이터 주체는 governdeliciousmap(old)이다. OfficialDeliciousMap develop(new)에서 **old에 없는 행·상호만 추가**하고, old 행은 삭제·수정하지 않는다.
- new의 규칙(classify 식당 판정, 반복 게시 합침, expense 금액 보정)은 적용하지 않는다.
- 추가 전에 old 자체 규칙인 `parsers/place_name.py`의 `drop_personal` → `normalize_place` → `split_places`를 new 행에 적용한다.
- 작업 장소는 governdeliciousmap이다. 스크립트가 여기 있고 `parsers/place_name.py`를 import한다. new의 `data/daegu/`는 읽기 전용으로 참조한다.
- 기간은 2026-01-01 ~ 2026-06-30(new의 reporting_period)이다.

## 대상 파일

| 분류 | old 파일 (유지) | new 출처 (읽기 전용) | 작업 |
|---|---|---|---|
| ② 데이터 | `data/processed/records.csv` | `data/daegu/records.csv`, `data/daegu/fetch.json` | old에 없는 행 끝에 추가 |
| ② 데이터 | `data/processed/geocode_cache.json` | `data/daegu/geocode.json` + `geocode.002.json`, `data/daegu/category-lookup-v1.jsonl` | old에 없는 `{상호}\|대구` 키만 추가 |
| ② 데이터 | `data/manual/truncated_places_daegu.csv` | `data/manual/daegu/` 없음 | 작업 없음 (old도 파일 없음) |

`dist/data/daegu.*.json`, `closure_report_daegu.json`, `match_keys_daegu.json`은 old 빌드가 재생성하므로 손대지 않는다.

## 기관 slug 변환표

| new `organization` | old `source` | new `records.csv` 행 | 비고 |
|---|---|---|---|
| `daegu-city` | `daegu_city` | 4,040 | |
| `daegu-dong` | `daegu_donggu` | 1,360 | |
| `daegu-seo` | `daegu_seogu` | 1,475 | |
| `daegu-nam` | `daegu_namgu` | 987 | |
| `daegu-suseong` | `daegu_suseong` | 1,438 | |
| `daegu-dalseong` | `daegu_dalseong` | 1,556 | |
| `daegu-gunwi` | `daegu_gunwi` | 479 | |
| `daegu-jung` | (없음) | 0 | new는 fetch만, old는 봇 차단으로 미수집. 양쪽 모두 없음 |
| `daegu-buk` | `daegu_bukgu` | 0 | new csv에 없음 → old 행 그대로 |
| `daegu-dalseo` | `daegu_dalseo` | 0 | new csv에 없음 → old 행 그대로 |

규칙: `-dong→_donggu`, `-seo→_seogu`, `-nam→_namgu`, 나머지는 `-`→`_`.

## 스크립트 공통 구조 (`migrate_official.py --city daegu`)

| 단계 | 함수 | 입력 (new develop) | 출력 (old) | 규칙 |
|---|---|---|---|---|
| 0 | `precheck(city)` | `data/daegu/records.csv`, `geocode*.json`, `fetch.json` | 없음 | 하나라도 없으면 중단 |
| 1 | `load_records(city)` | `records.csv` 12열 | 메모리 | `amount_krw` 빈 행·`spent_on` 일 없는 행은 **제외하고 건수만 보고** (expense 보정은 new 규칙이라 미적용) |
| 2 | `map_source(org)` | `organization` | `source` | 위 변환표 |
| 3 | `map_file(source_hash)` | `fetch.json` `sources[].path` | `file` | basename |
| 4 | `apply_old_rules(rows)` | 1~3 결과 | 메모리 | old의 `drop_personal`(개인·경조사·현금성 제거) → `normalize_place`(빈 값이면 제거) → `place_raw`=원본 |
| 5 | `append_records(city)` | 4 결과 | `data/processed/records.csv` | 키 `(date, source, place_raw 정규화, amount)`가 old에 **없는 행만 끝에 추가**. old `place`(정규화 후)로도 한 번 더 대조. old 행은 삭제·수정 없음 |
| 6 | `append_geocode(city)` | `geocode*.json` 분할 전부 + `category-lookup-v1.jsonl` | `geocode_cache.json` | 키 `{merchant}\|대구`가 old에 없고 주소가 `region_prefixes`(`대구광역시`, `대구`) + `districts` 안일 때만 추가. `place_url=""` |
| 7 | `append_manual(city)` | `data/manual/daegu/*.jsonl` | `truncated_places_daegu.csv` | 대구는 new manual 없음 → 건너뜀 |
| 8 | `report(city)` | 5~7 | stdout | 추가·제외·건너뜀 건수 |

## 대구 도시별 값·검증 값

| 단계 | 도시별 값 | 검증 값 (사용자 확인) |
|---|---|---|
| 1 | 11,335행. 제외 0 (금액 빈 값 0, 일 없는 날짜 0) | 대상 11,335 |
| 2 | 7개 기관 매핑. new에 중구·북구·달서구 없음 | 매핑 실패 0. old `daegu_bukgu`·`daegu_dalseo` 행 그대로 |
| 3 | `fetch.json` sources 2,074개. 예 `daegu/daegu-city/expenses/822983-1.xlsx` | `file` 미해결 0 |
| 4~5 | old에 없는 행 5,162 (old 규칙 적용 전). 수성 1,381 · 시청 1,333 · 남구 952 · 서구 804 · 달성 357 · 동구 181 · 군위 154. `개인(성명 비공개)` 등은 `drop_personal`에서 빠짐 | 추가 ≤ 5,162. old 대구 9,740행은 그대로 |
| 6 | new 좌표 성공 5,395건 → 상호 2,538개. old 캐시 `\|대구` 4,008키(좌표 있는 상호 3,325). 공통 1,401 건너뜀. old에 없음 1,137, 그중 관할 주소 1,127 | 추가 ≤ 1,127. old `\|대구` 4,008키 그대로 |
| 7 | `data/manual/daegu/` 없음 | 보정표 생성 안 함 |

참고: old 2026H1 대구 마커 2,887 · 레코드 9,212 / new 마커 2,134 · 레코드 11,335. new 마커가 적은 것은 classify·지오코딩 실패 때문이며 이관과 무관하다.

## 필드 매핑

### `data/processed/records.csv`

| old 열 | new 출처 → 요소 | 변환 |
|---|---|---|
| `place` | `merchant` | old `normalize_place` 적용 후 값 |
| `place_raw` | `merchant` | 원본 그대로 |
| `amount` | `amount_krw` | 빈 값 행은 제외 (대구 0건) |
| `date` | `spent_on` | `YYYY-MM-DD` 그대로 (대구 전부 10자) |
| `purpose` | `purpose` | 그대로 |
| `dept` | `department` | 그대로 |
| `source` | `organization` | 위 변환표 |
| `file` | `source_hash` → `fetch.json` `sources[].path` | basename |

### `data/processed/geocode_cache.json`

| old 키/값 | new 출처 → 요소 | 변환 |
|---|---|---|
| 키 `{상호}\|대구` | `geocode*.json` `results[].merchant` (status=success) | old에 없는 키만 |
| `naver_name` | `lookup.candidates[]` 중 `latitude`/`longitude`가 결과와 같은 후보의 `merchant` | 대구는 `confirmation` 0건이라 후보 매칭으로만 |
| `lat`, `lng` | `latitude`, `longitude` | 주소가 대구 8개 구·군 밖이면 제외 |
| `address` | 같은 후보의 `address` | |
| `category` | `category-lookup-v1.jsonl`(2,106줄) `value.categories[].category` | 후보 `source.source_id`로 조인 |
| `place_url` | 없음 | 빈 문자열 |

## 검증 체크리스트

- [ ] 0단계: new `data/daegu/records.csv`, `geocode.json`, `geocode.002.json`, `fetch.json` 존재 확인
- [ ] 1단계: 대상 11,335행, 제외 0건
- [ ] 2단계: slug 매핑 실패 0건. 7개 기관만 등장
- [ ] 3단계: `file` 미해결 0건
- [ ] 5단계: 추가 행 ≤ 5,162. old 대구 9,740행 모두 남아 있음 (`source` 9개 폴더명 기준 행수 변화 없음)
- [ ] 5단계: 추가된 행에 `개인(성명 비공개)`·빈 `place` 없음
- [ ] 6단계: 추가 키 ≤ 1,127. 기존 `\|대구` 4,008키 값 변화 없음
- [ ] 6단계: 추가된 주소가 전부 `대구광역시` 또는 `대구`로 시작하고 8개 구·군 중 하나를 포함
- [ ] 6단계: `NaN` 없음 (`json.loads` 후 `math.isfinite` 검사)
- [ ] 7단계: `truncated_places_daegu.csv` 생성되지 않음
- [ ] 이관 후 `python main.py` 빌드로 `dist/data/daegu.*.json` 재생성, 마커 수가 2,887 이상인지 확인
