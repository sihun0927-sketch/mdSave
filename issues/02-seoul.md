# [이관] 서울 — OfficialDeliciousMap develop → governdeliciousmap

## 배경

- 데이터 주체는 governdeliciousmap(old)이다. OfficialDeliciousMap develop(new)은 읽기 전용으로 참조하며 수정하지 않는다.
- 이관 원칙: old에 없는 행·상호·수동 보정만 추가한다. old 행은 삭제·수정하지 않는다.
- new의 규칙(classify 식당 판정, 반복 게시 합침, expense 금액 보정)은 가져오지 않는다. 추가 전에 old 자체 규칙(`parsers/place_name.py`의 `drop_personal`·`normalize_place`·`split_places`)을 new 행에 적용한다.
- 작업 장소는 governdeliciousmap이다. 스크립트 `migrate_official.py --city seoul`이 여기 위치하고 `parsers/place_name.py`를 import한다.
- 결론: **서울은 이관할 데이터가 없다.** 스크립트는 0단계 precheck에서 중단한다.

## new develop 현황 (서울)

| 파일 | new develop | 비고 |
|---|---|---|
| `data/seoul/orgs/*/fetch.json` | 있음 (25개 기관) | 원본 수집 목록만. 강북구(`seoul-gangbuk`)는 registry에만 있고 fetch.json 없음 |
| `data/seoul/fetch.json` | 없음 | 도시 단위 fetch 없음 |
| `data/seoul/headermap.json` | 없음 | |
| `data/seoul/records.csv` | 없음 | parse 단계 미실행 |
| `data/seoul/geocode.json` | 없음 | geocode 단계 미실행 |
| `data/seoul/category-lookup-v1.jsonl` | 없음 | |
| `data/manual/seoul/` | 없음 | 수동 보정 없음 |

old 서울 데이터는 그대로 둔다.

| old 파일 | 서울 현황 | 이관 후 |
|---|---|---|
| `data/processed/records.csv` | 서울 179,255행 (2026H1 145,951행) | 변동 없음 |
| `data/processed/geocode_cache.json` | `\|서울` 키 38,061건 | 변동 없음 |
| `data/manual/truncated_places_seoul.csv` | 파일 없음 | 생성 안 함 |

## 기관 slug 변환표 (미래 이관 대비)

old `cities.py`의 `sources` 키 ↔ new `registry/seoul.py`의 org slug. new는 구 이름을 풀어 쓰고 old는 일부 축약형을 쓴다.

| new slug | old source | 화면 표기 | new fetch.json |
|---|---|---|---|
| `seoul-city` | `seoul_city` | 서울시청 | 있음 |
| `seoul-jongno` | `seoul_jongno` | 종로구청 | 있음 |
| `seoul-jung` | `seoul_junggu` | 중구청 | 있음 |
| `seoul-yongsan` | `seoul_yongsan` | 용산구청 | 있음 |
| `seoul-seongdong` | `seoul_seongdong` | 성동구청 | 있음 |
| `seoul-gwangjin` | `seoul_gwangjin` | 광진구청 | 있음 |
| `seoul-dongdaemun` | `seoul_ddm` | 동대문구청 | 있음 |
| `seoul-jungnang` | `seoul_jungnang` | 중랑구청 | 있음 |
| `seoul-seongbuk` | `seoul_seongbuk` | 성북구청 | 있음 |
| `seoul-gangbuk` | `seoul_gangbuk` | 강북구청 | **없음** |
| `seoul-dobong` | `seoul_dobong` | 도봉구청 | 있음 |
| `seoul-nowon` | `seoul_nowon` | 노원구청 | 있음 |
| `seoul-eunpyeong` | `seoul_eunpyeong` | 은평구청 | 있음 |
| `seoul-seodaemun` | `seoul_sdm` | 서대문구청 | 있음 |
| `seoul-mapo` | `seoul_mapo` | 마포구청 | 있음 |
| `seoul-yangcheon` | `seoul_yangcheon` | 양천구청 | 있음 |
| `seoul-gangseo` | `seoul_gangseo` | 강서구청 | 있음 |
| `seoul-guro` | `seoul_guro` | 구로구청 | 있음 |
| `seoul-geumcheon` | `seoul_geumcheon` | 금천구청 | 있음 |
| `seoul-yeongdeungpo` | `seoul_ydp` | 영등포구청 | 있음 |
| `seoul-dongjak` | `seoul_dongjak` | 동작구청 | 있음 |
| `seoul-gwanak` | `seoul_gwanak` | 관악구청 | 있음 |
| `seoul-seocho` | `seoul_seocho` | 서초구청 | 있음 |
| `seoul-gangnam` | `seoul_gangnam` | 강남구청 | 있음 |
| `seoul-songpa` | `seoul_songpa` | 송파구청 | 있음 |
| `seoul-gangdong` | `seoul_gangdong` | 강동구청 | 있음 |

단순 `-`→`_` 치환으로 안 되는 4개: `seoul-jung→seoul_junggu`, `seoul-dongdaemun→seoul_ddm`, `seoul-seodaemun→seoul_sdm`, `seoul-yeongdeungpo→seoul_ydp`.

## 스크립트 공통 구조

`migrate_official.py --city {city}` 하나로 7개 도시를 돌린다. 도시별 차이는 값만 다르다.

| 단계 | 함수 | 입력 (new develop) | 출력 (old) | 규칙 |
|---|---|---|---|---|
| 0 | `precheck(city)` | `data/{city}/records.csv`, `geocode*.json`, `fetch.json` | 없음 | 하나라도 없으면 중단 |
| 1 | `load_records(city)` | `records.csv` 12열 | 메모리 | `amount_krw` 빈 행·`spent_on` 일 없는 행은 제외하고 건수만 보고 (expense 보정은 new 규칙이라 미적용) |
| 2 | `map_source(org)` | `organization` | `source` | 도시별 변환표 |
| 3 | `map_file(source_hash)` | `fetch.json` `sources[].path` | `file` | basename |
| 4 | `apply_old_rules(rows)` | 1~3 결과 | 메모리 | old의 `drop_personal`(개인·경조사·현금성 제거) → `normalize_place`(빈 값이면 제거) → `place_raw`=원본 |
| 5 | `append_records(city)` | 4 결과 | `data/processed/records.csv` | 키 `(date, source, place_raw 정규화, amount)`가 old에 없는 행만 끝에 추가. old `place`로도 한 번 더 대조. old 행은 삭제·수정 없음 |
| 6 | `append_geocode(city)` | `geocode*.json` 분할 전부 + `category-lookup-v1.jsonl` | `geocode_cache.json` | 키 `{merchant}\|{region}`이 old에 없고 주소가 `region_prefixes`+`districts` 안일 때만 추가. `place_url=""` |
| 7 | `append_manual(city)` | `data/manual/{city}/restore·classify·geocode.jsonl` | `truncated_places_{city}.csv` | `잘린상호`가 old에 없을 때만 추가 |
| 8 | `report(city)` | 5~7 | stdout | 추가·제외·건너뜀 건수 |

## 서울 실행 결과 (예상)

| 단계 | 서울 값 | 결과 |
|---|---|---|
| 0 | `data/seoul/records.csv` 없음, `geocode.json` 없음, `fetch.json`(도시 단위) 없음 | **중단**. 출력: `seoul: parse 단계 없음, 이관 없음` |
| 1~8 | 실행 안 함 | old 서울 179,255행·캐시 38,061키·보정표(없음) 그대로 |

## 검증 체크리스트

- [ ] `python migrate_official.py --city seoul` 실행 시 0단계에서 중단하고 `seoul: parse 단계 없음, 이관 없음`을 출력한다.
- [ ] 실행 전후 `data/processed/records.csv` 해시가 같다.
- [ ] 실행 전후 `data/processed/geocode_cache.json` 해시가 같다.
- [ ] `data/manual/truncated_places_seoul.csv`가 생성되지 않는다.
- [ ] 종료 코드가 0이 아닌 값(중단)이거나, 0이더라도 쓰기 작업이 하나도 없다.

## 재개 조건

new develop에 `data/seoul/records.csv`와 `data/seoul/geocode.json`이 생기면 광주 이슈와 같은 절차(1~8단계)로 진행한다. 그때 위 변환표를 그대로 쓰고, 강북구 fetch.json 유무를 다시 확인한다.

## 참조 파일 (OfficialDeliciousMap develop, 읽기 전용)

저장소: https://github.com/snowjaewon/OfficialDeliciousMap (브랜치 `develop`)

| 경로 | 링크 |
|---|---|
| `data/seoul/orgs/ (25개 기관의 fetch.json, precheck 중단 판정용)` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/data/seoul/orgs/ |
| `src/deliciousmap/registry/seoul.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/registry/seoul.py |
| `src/deliciousmap/storage.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/storage.py |
| `src/deliciousmap/contracts.py` | https://github.com/snowjaewon/OfficialDeliciousMap/blob/develop/src/deliciousmap/contracts.py |
