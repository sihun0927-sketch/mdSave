# [이관] 인천 — OfficialDeliciousMap develop → governdeliciousmap

## 배경

- 데이터 주체는 governdeliciousmap(old)이다. OfficialDeliciousMap develop(new)은 읽기 전용 참조이며 수정하지 않는다.
- new에서 old에 없는 행·상호·보정만 추가한다. new 규칙(classify·반복 게시 합침·expense)은 적용하지 않는다.
- 추가 전에 old 자체 규칙(`parsers/place_name.py`의 `drop_personal`·`normalize_place`·`split_places`)을 new 행에 적용한다.
- 작업 장소는 governdeliciousmap이다. 스크립트 `migrate_official.py --city incheon`이 여기 있고, 쓰기 대상도 여기의 `data/processed/records.csv`·`data/processed/geocode_cache.json`·`data/manual/truncated_places_incheon.csv`다.
- 인천은 new에 parse 이후 단계 산출물이 없어 이번 이관에서 **추가되는 데이터가 없다**. 이 이슈는 그 사실을 확인하고 old 파일이 변하지 않았음을 검증하는 이슈다.

## new develop 인천 현황

| 구분 | 파일 | 상태 |
|---|---|---|
| 있음 | `data/incheon/fetch.json` | 원본 5,682개, missing 4 |
| 있음 | `data/incheon/headermap.json` | 헤더 판정 |
| 있음 | `data/incheon/orgs/{org}/fetch.json`, `headermap.json` | 12개 기관 |
| 없음 | `data/incheon/records.csv` | parse 단계 미실행 |
| 없음 | `data/incheon/parse.json`, `classify.json`, `geocode*.json`, `closure.json`, `build.json` | 미실행 |
| 없음 | `data/incheon/category-lookup-v1.jsonl`, `geocode-lookup-v1.jsonl`, `geocode-history-v2.jsonl` | 미실행 |
| 없음 | `data/manual/incheon/` | 수동 보정 없음 |

## 기관 slug 변환표 (old `source` ↔ new `organization`)

12개 전부 `_` ↔ `-` 차이뿐이다.

| old `source` | new `organization` | 기관 |
|---|---|---|
| `incheon_city` | `incheon-city` | 인천시청 |
| `incheon_jemulpo` | `incheon-jemulpo` | 제물포구청 |
| `incheon_yeongjong` | `incheon-yeongjong` | 영종구청 |
| `incheon_michuhol` | `incheon-michuhol` | 미추홀구청 |
| `incheon_yeonsu` | `incheon-yeonsu` | 연수구청 |
| `incheon_namdong` | `incheon-namdong` | 남동구청 |
| `incheon_bupyeong` | `incheon-bupyeong` | 부평구청 |
| `incheon_gyeyang` | `incheon-gyeyang` | 계양구청 |
| `incheon_seohae` | `incheon-seohae` | 서해구청 |
| `incheon_geomdan` | `incheon-geomdan` | 검단구청 |
| `incheon_ganghwa` | `incheon-ganghwa` | 강화군청 |
| `incheon_ongjin` | `incheon-ongjin` | 옹진군청 |

## 스크립트 공통 구조 (모든 도시 동일)

| 단계 | 함수 | 입력 (new develop) | 출력 (old) | 규칙 |
|---|---|---|---|---|
| 0 | `precheck(city)` | `data/{city}/records.csv`, `geocode*.json`, `fetch.json` | 없음 | 하나라도 없으면 중단 |
| 1 | `load_records(city)` | `records.csv` 12열 | 메모리 | `amount_krw` 빈 행·`spent_on` 일 없는 행은 제외하고 건수만 보고 (expense 보정은 new 규칙이라 미적용) |
| 2 | `map_source(org)` | `organization` | `source` | 도시별 변환표 |
| 3 | `map_file(source_hash)` | `fetch.json` `sources[].path` | `file` | basename |
| 4 | `apply_old_rules(rows)` | 1~3 결과 | 메모리 | old의 `drop_personal`(개인·경조사·현금성 제거) → `normalize_place`(빈 값이면 제거) → `place_raw`=원본 |
| 5 | `append_records(city)` | 4 결과 | `data/processed/records.csv` | 키 `(date, source, place_raw 정규화, amount)`가 old에 없는 행만 끝에 추가. old 행은 삭제·수정 없음 |
| 6 | `append_geocode(city)` | `geocode*.json` 분할 전부 + `category-lookup-v1.jsonl` | `geocode_cache.json` | 키 `{merchant}\|{region}`이 old에 없고 주소가 `region_prefixes`+`districts` 안일 때만 추가. `place_url=""` |
| 7 | `append_manual(city)` | `data/manual/{city}/restore·classify·geocode.jsonl` | `truncated_places_{city}.csv` | `잘린상호`가 old에 없을 때만 추가 |
| 8 | `report(city)` | 5~7 | stdout | 추가·제외·건너뜀 건수 |

## 인천 실행 결과 (예상)

| 단계 | 도시별 값 | 검증 값 |
|---|---|---|
| 0 | `data/incheon/records.csv`·`geocode*.json` 없음. `fetch.json`(원본 5,682)·`headermap.json`만 있음 | **중단**. 출력: `incheon: parse 단계 없음, 이관 없음` |
| 1~8 | 실행 안 함 | old 인천 16,700행·캐시 `\|인천` 5,339키 그대로 |

## 검증 체크리스트

- [ ] `python migrate_official.py --city incheon` 실행 시 0단계에서 중단되고 stdout에 `incheon: parse 단계 없음, 이관 없음`이 출력된다.
- [ ] 종료 코드가 0이 아니거나, 중단 사유가 명시된다 (records.csv·geocode.json 부재).
- [ ] 실행 전후 `data/processed/records.csv`의 해시가 같다. 인천 행(`source`가 `incheon_*`) 16,700행 유지.
- [ ] 실행 전후 `data/processed/geocode_cache.json`의 해시가 같다. `|인천` 키 5,339개 유지.
- [ ] `data/manual/truncated_places_incheon.csv`가 새로 생성되지 않는다 (old에도 원래 없음).
- [ ] new 저장소(OfficialDeliciousMap)에 아무 파일도 쓰지 않는다.

## 재개 조건

new develop에 `data/incheon/records.csv`와 `data/incheon/geocode.json`(분할 포함)이 생기면 이 이슈를 다시 열고 광주 이슈의 절차(1~8단계)를 그대로 적용한다. 그때 확인할 값:

- new records.csv 행수, 금액 빈 행·일 없는 날짜 행 수
- 기관 12개 중 new에 실제 레코드가 있는 기관
- old 캐시 `|인천` 5,339키와 겹치지 않는 new 성공 상호 수, 그중 관할 주소(`region_prefixes` + 11개 구·군) 안인 수
- `data/manual/incheon/` 생성 여부

## 참조 파일 (OfficialDeliciousMap develop, 읽기 전용)

저장소: https://github.com/snowjaewon/OfficialDeliciousMap (브랜치 `develop`)

| 경로 | 링크 |
|---|---|
| `data/incheon/fetch.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/incheon/fetch.json |
| `data/incheon/headermap.json` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/incheon/headermap.json |
| `src/deliciousmap/registry/incheon.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/registry/incheon.py |
| `src/deliciousmap/storage.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/storage.py |
| `src/deliciousmap/contracts.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/contracts.py |
