# [이관] 광주 — OfficialDeliciousMap develop → governdeliciousmap

## 배경

- 데이터 주체는 governdeliciousmap(old)이다. OfficialDeliciousMap develop(new)에서는 **old에 없는 행·상호·보정만 추가**한다. old 행은 삭제·수정하지 않는다.
- new의 규칙(classify 식당 판정, 반복 게시 합침, expense 금액 보정)은 이관하지 않는다.
- 추가 전에 old 자체 규칙인 `parsers/place_name.py`의 `drop_personal`·`normalize_place`·`split_places`를 new 행에 적용한다.
- 작업 장소는 governdeliciousmap이다. OfficialDeliciousMap은 읽기 전용으로만 참조한다.
- 기간은 양쪽 모두 2026년 상반기(2026-01-01 ~ 2026-06-30) 기준이다.

## 대상 파일

| old 파일 (유지) | 형식 | new 출처 (읽기 전용) |
|---|---|---|
| `data/processed/records.csv` | CSV 8열 | `data/gwangju/records.csv`, `data/gwangju/fetch.json` |
| `data/processed/geocode_cache.json` | JSON | `data/gwangju/geocode.json` + `geocode.002.json` ~ `geocode.004.json`, `data/gwangju/category-lookup-v1.jsonl` |
| `data/manual/truncated_places_gwangju.csv` | CSV 11열 | `data/manual/gwangju/restore.jsonl`, `classify.jsonl`, `geocode.jsonl` |

이관하지 않는 old 파일: `dist/data/*.json`(빌드가 재생성), `closure_report_gwangju.json`(new는 전부 `unknown`), `match_keys_gwangju.json`(빌드가 재생성).

## 기관 slug 변환표

| new `organization` | old `source` |
|---|---|
| `gwangju-city` | `gwangju_city` |
| `gwangju-buk` | `bukgu` |
| `gwangju-dong` | `donggu` |
| `gwangju-nam` | `namgu` |
| `gwangju-seo` | `seogu` |
| `gwangju-gwangsan` | `gwangsan` |

old의 council 소스 5개(`donggu_council`, `bukgu_council`, `gwangju_council`, `donggu_member`, `seogu_member`)는 new에 없으므로 건드리지 않는다.

## 스크립트 공통 구조 (`migrate_official.py --city gwangju`)

| 단계 | 함수 | 입력 (new develop) | 출력 (old) | 규칙 |
|---|---|---|---|---|
| 0 | `precheck(city)` | `data/{city}/records.csv`, `geocode*.json`, `fetch.json` | 없음 | 하나라도 없으면 중단 |
| 1 | `load_records(city)` | `records.csv` 12열 | 메모리 | `amount_krw` 빈 행·`spent_on` 일 없는 행은 **제외하고 건수만 보고** (expense 보정은 new 규칙이라 미적용) |
| 2 | `map_source(org)` | `organization` | `source` | 위 변환표 |
| 3 | `map_file(source_hash)` | `fetch.json` `sources[].path` | `file` | basename |
| 4 | `apply_old_rules(rows)` | 1~3 결과 | 메모리 | old의 `drop_personal`(개인·경조사·현금성 제거) → `normalize_place`(빈 값이면 제거) → `place_raw`=원본 |
| 5 | `append_records(city)` | 4 결과 | `data/processed/records.csv` | 키 `(date, source, place_raw 정규화, amount)`가 old에 **없는 행만 끝에 추가**. old `place`(정규화 후)로도 한 번 더 대조. old 행은 삭제·수정 없음 |
| 6 | `append_geocode(city)` | `geocode*.json` 분할 전부 + `category-lookup-v1.jsonl` | `geocode_cache.json` | 키 `{merchant}\|광주`가 old에 없고 주소가 `region_prefixes`+`districts` 안일 때만 추가. `place_url=""` |
| 7 | `append_manual(city)` | `data/manual/gwangju/restore·classify·geocode.jsonl` | `truncated_places_gwangju.csv` | `잘린상호`가 old에 없을 때만 추가 |
| 8 | `report(city)` | 5~7 | stdout | 추가·제외·건너뜀 건수 |

## 광주 도시별 값 · 검증 값

| 단계 | 도시별 값 | 검증 값 (사용자 확인) |
|---|---|---|
| 1 | 10,335행. 제외: 금액 빈 1,020 · 일 없음 2 → 대상 9,313 | 제외 1,022건 보고 |
| 2 | `gwangju-city→gwangju_city`, `-buk→bukgu`, `-dong→donggu`, `-nam→namgu`, `-seo→seogu`, `-gwangsan→gwangsan` | 매핑 실패 0 |
| 4~5 | old에 없는 행 1,661 (규칙 적용 전). `개인(성명 비공개)`·`집무실` 등이 `drop_personal`에서 빠짐 | 추가 ≤ 1,661. old 광주 33,477행은 그대로 |
| 6 | new 상호 2,381 중 old 캐시 없음 519 | 관할 주소만 추가 (여수·나주·담양 제외). old `\|광주` 3,600키 그대로 |
| 7 | `restore` 6 → 정확한상호명, `classify` non_restaurant → 식당아님, `geocode` 33 → 주소힌트 | 보정표 111행 + ≤ 49 |

참고 수치 (판단 근거):

| 항목 | old | new |
|---|---|---|
| 광주 2026H1 레코드 | 9,073 (council 제외) | 10,335 |
| 기관별 old에 없는 행 (규칙 적용 전) | — | 시청 646 · 북구 353 · 남구 393 · 광산 130 · 서구 70 · 동구 69 |
| 좌표 있는 상호 | 3,004 | 2,381 (성공 6,480건) |
| 공통 상호 좌표 1km 이상 어긋남 | 435 / 1,862 (new가 여수·나주·담양 등 외곽 인허가 채택) | |

공통 상호는 old 좌표를 유지한다. new 좌표로 덮어쓰지 않는다.

## 필드 매핑

### `data/processed/records.csv`

| old 열 | new 요소 | 변환 |
|---|---|---|
| `place` | `merchant` | old `normalize_place` 적용 후 값 |
| `place_raw` | `merchant` | 원본 그대로 |
| `amount` | `amount_krw` | 그대로. 빈 값은 행 제외 |
| `date` | `spent_on` | `YYYY-MM-DD`만. 일 없는 표기는 행 제외 |
| `purpose` | `purpose` | 그대로 |
| `dept` | `department` | 그대로 |
| `source` | `organization` | slug 변환표 |
| `file` | `source_hash` → `fetch.json` `sources[].path` | basename. 못 찾으면 해시 앞 16자 |

### `data/processed/geocode_cache.json`

| old 키 / 값 | new 요소 | 변환 |
|---|---|---|
| 키 `{상호}\|광주` | `results[].merchant` (status=success) | old에 없는 상호만 |
| `naver_name` | `confirmed_merchant`, 없으면 `lookup.candidates[].merchant` | 좌표 일치 후보 |
| `lat`, `lng` | `latitude`, `longitude` | 주소가 광주 5개 구 밖이면 제외 |
| `address` | `confirmation.address`, 없으면 `lookup.candidates[].address` | 좌표 일치 후보 |
| `category` | `category-lookup-v1.jsonl` `value.categories[].category` | 후보 `source.source_id`로 조인 |
| `place_url` | 없음 | 빈 문자열 |

### `data/manual/truncated_places_gwangju.csv`

| old 열 | new 요소 | 건수 |
|---|---|---|
| `잘린상호`, `정확한상호명` | `restore.jsonl` → `scope.merchant`, `restored_merchant` | 6 |
| `잘린상호`, `식당아님` | `classify.jsonl` → `merchant`, `status=non_restaurant` → `Y` | 10건 중 non_restaurant만 |
| `잘린상호`, `주소힌트` | `geocode.jsonl` → `scope.merchant`, `address` | 33 |

`merchants.jsonl`(448)·`repeats.jsonl`(2)·`sources.jsonl`(15)은 old에 대응 개념이 없어 이관하지 않는다.

## 검증 체크리스트

- [ ] 0단계: new `data/gwangju/records.csv`, `geocode.json`~`geocode.004.json`, `fetch.json` 존재 확인
- [ ] 1단계: 제외 건수 = 1,022 (금액 빈 1,020 + 일 없음 2)
- [ ] 2단계: slug 매핑 실패 0건
- [ ] 3단계: `file` 미해결 건수 보고 (해시 앞 16자로 대체된 행 수)
- [ ] 5단계: `records.csv` 추가 행 ≤ 1,661
- [ ] 5단계: old 광주 6개 소스 기존 행 33,477 불변, council 5개 소스 2,651행 불변
- [ ] 5단계: old 전체 행 수 = 265,612 + 추가 행 수
- [ ] 6단계: `geocode_cache.json` 추가 키 ≤ 519, 전부 `|광주` 접미
- [ ] 6단계: 추가 좌표의 주소가 전부 `region_prefixes` 접두 + 5개 구(동구·서구·남구·북구·광산구) 포함
- [ ] 6단계: 기존 `|광주` 3,600키 값 불변
- [ ] 7단계: 보정표 111행 + ≤ 49행, `잘린상호` 중복 0
- [ ] 이관 후 `python main.py`(또는 map_builder) 빌드가 광주에서 오류 없이 끝남

## 사용자 결정 필요

1. **금액 빈 1,020행 제외 여부.** new는 반복 게시 합침(expense)으로 금액을 `expense_amount_krw`에만 남겼다. old 규칙에는 없는 처리라 기본은 제외. 채워서 넣으려면 중복 집계 위험을 감수해야 한다.
2. **일 없는 날짜 2행 제외 여부.** `spent_on`이 `2026.03`, `2026.05`인 행. 기본은 제외. `-01`로 보정해 넣을지 결정 필요.

## 참조 파일 (OfficialDeliciousMap develop, 읽기 전용)

저장소: https://github.com/snowjaewon/OfficialDeliciousMap (브랜치 `develop`)

| 경로 | 링크 |
|---|---|
| `data/gwangju/records.csv` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/gwangju/records.csv |
| `data/gwangju/fetch.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/gwangju/fetch.json |
| `data/gwangju/geocode.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/gwangju/geocode.json |
| `data/gwangju/geocode.002.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/gwangju/geocode.002.json |
| `data/gwangju/geocode.003.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/gwangju/geocode.003.json |
| `data/gwangju/geocode.004.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/gwangju/geocode.004.json |
| `data/gwangju/category-lookup-v1.jsonl` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/gwangju/category-lookup-v1.jsonl |
| `data/manual/gwangju/restore.jsonl` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/manual/gwangju/restore.jsonl |
| `data/manual/gwangju/classify.jsonl` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/manual/gwangju/classify.jsonl |
| `data/manual/gwangju/geocode.jsonl` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/manual/gwangju/geocode.jsonl |
| `src/deliciousmap/registry/gwangju.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/registry/gwangju.py |
| `src/deliciousmap/storage.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/storage.py |
| `src/deliciousmap/contracts.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/contracts.py |
