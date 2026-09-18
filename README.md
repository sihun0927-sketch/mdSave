# Govern <old>

| 분류                            | 파일                                           | 역할                                                                                                                                                            |
| ------------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **① 출력 (마커를 그리는 쪽)**   | `map_builder.py`                               | 핵심. `aggregate()`가 records → 마커 목록 생성, `build_city_data()`가 `data/{key}.{hash}.json` 기록, `build_map_shell()`이 `map.html`(JS 마커 렌더링 포함) 생성 |
|                                 | `dist/map.html`                                | 실제 브라우저에서 `?city=` 읽어 JSON fetch 후 네이버 지도에 마커 찍는 화면                                                                                      |
|                                 | `dist/data/{city}.{hash}.json` ×7              | 도시별 마커·메타 (busan·daegu·daejeon·gwangju·incheon·seoul·ulsan)                                                                                              |
|                                 | `dist/data/{city}.ledger.{hash}.json` ×7       | 마커 클릭 시 보이는 장부(지출 내역)                                                                                                                             |
|                                 | `landing_builder.py` → `dist/index.html`       | 첫 페이지 도시 카드. JSON meta의 식당 수 표시                                                                                                                   |
|                                 | `pwa_builder.py`                               | `data/processed/` 산출물을 `dist/`로 복사 + PWA 껍데기                                                                                                          |
| **② 데이터 (마커를 이루는 쪽)** | `data/processed/records.csv`                   | 전 도시 지출 장부 원본 (scrapers → parsers 결과)                                                                                                                |
|                                 | `data/processed/geocode_cache.json`            | 상호 → 좌표 캐시 (`geocoder.py`가 네이버 지역검색으로 채움)                                                                                                     |
|                                 | `data/manual/truncated_places_{city}.csv` ×3   | 사람이 상호를 보정한 수동 오버라이드 (`export_truncated.py`가 뽑고 `geocoder.load_overrides()`가 읽음)                                                          |
|                                 | `licence_geocode.py`                           | 네이버에 없는 가게를 인허가 좌표(EPSG:5174)로 보강                                                                                                              |
|                                 | `data/processed/match_keys_{city}.json` ×7     | 마커의 주소 키(구·도로명) — 인허가 대조용                                                                                                                       |
|                                 | `data/processed/closure_report_{city}.json` ×7 | 폐업 판정 결과. 마커 표시 여부에 반영                                                                                                                           |
|                                 | `cities.py`                                    | 도시 정의(키·중심좌표·기관명)                                                                                                                                   |

흐름 한 줄: `scrapers/` → `parsers/` → `records.csv` → `geocoder.py`(+cache, 수동 CSV, 인허가) → `map_builder.py` → `dist/data/*.json` + `dist/map.html`.



# Official <new>

지도 탭 마커 관련 파일, 두 분류로 정리:

| 분류                            | 파일                                                                                     | 역할                                                                                                                                                                             |
| ------------------------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **① 출력 (마커를 그리는 쪽)**   | `src/deliciousmap/site.py`                                                               | 핵심. `_marker_file()`이 후보 → `PublishedMarker` 목록 생성, `write_city_data()`가 `dist/{city}/markers.json`·`records.json` 기록, 도시 `index.html`(data-markers-url 속성) 생성 |
|                                 | `src/deliciousmap/site_assets/app.js`                                                    | 브라우저에서 `data-markers-url` fetch 후 네이버 지도에 마커 표시. `markerIcon()`이 방문 구간 색·폐업·선택 상태를 그림, `filterMarkers()`·`rankMarkers()`로 필터·목록             |
|                                 | `src/deliciousmap/site_assets/styles.css`                                                | `.map-marker`, `.band-*`, `.is-closed` 마커 스타일                                                                                                                               |
|                                 | `dist/{city}/markers.json` ×7                                                            | 도시별 마커(business_id·상호·좌표·방문 횟수·폐업·업종·합계 금액·기관)                                                                                                            |
|                                 | `dist/{city}/records.json` ×7                                                            | 마커 클릭·장부 탭에서 보이는 전체 레코드(마커가 못 된 것도 사유와 함께)                                                                                                          |
|                                 | `src/deliciousmap/pipeline.py`                                                           | `marker_candidates()`가 식당 판정 + 지오코딩 성공 레코드를 `business_id`로 묶어 마커 후보 생성. `build` 단계가 site.py 호출                                                      |
|                                 | `src/deliciousmap/contracts.py`                                                          | `MarkerCandidate`·`PublishedMarker`·`MarkerFile` 스키마                                                                                                                          |
|                                 | `src/deliciousmap/publish.py`                                                            | `dist/` 공개 파일 검사·manifest 봉인                                                                                                                                             |
| **② 데이터 (마커를 이루는 쪽)** | `data/{city}/records.csv`                                                                | 파싱된 레코드 전체(장부 원천). 열 순서는 `storage.RECORD_FIELDS`                                                                                                                 |
|                                 | `data/{city}/parse.json`                                                                 | 레코드 메타(원본 목록·기간 제외·재게시 합침)                                                                                                                                     |
|                                 | `data/{city}/classify.json`                                                              | 비식당 판별 결과. `restaurant`인 레코드만 마커 후보                                                                                                                              |
|                                 | `data/{city}/geocode.json`                                                               | 레코드별 좌표·`business_id`·상호 확정. 마커 좌표와 묶음의 직접 원천                                                                                                              |
|                                 | `data/{city}/geocode-lookup-v1.jsonl`, `geocode-history-v2.jsonl`                        | 네이버 지역검색 조회 캐시와 판정 이력(추가만 함)                                                                                                                                 |
|                                 | `data/{city}/closure.json`                                                               | 인허가 대조 폐업 판정. 마커 `closed` 플래그                                                                                                                                      |
|                                 | `data/{city}/category-lookup-v1.jsonl`                                                   | 업종 조회 캐시. 마커 `category`·`category_group`                                                                                                                                 |
|                                 | `data/{city}/build.json`                                                                 | build 단계 메타(입력 해시·marker_count·record_count)                                                                                                                             |
|                                 | `data/manual/{city}/geocode.jsonl`, `merchants.jsonl`, `classify.jsonl`, `restore.jsonl` | 사람 보정(업소 확인·상호 가르기·식당 판정·상호 복원). 자동 판정보다 우선                                                                                                         |
|                                 | `src/deliciousmap/registry/{city}.py`                                                    | 도시 정의(slug·중심좌표·주소 접두·기관·게시판)                                                                                                                                   |
|                                 | `src/deliciousmap/naver.py`, `licenses.py`, `local.py`                                   | 지오코딩 제공자(네이버 지역검색·인허가 좌표·로컬)                                                                                                                                |
|                                 | `src/deliciousmap/identity.py`                                                           | `business_id` 생성, 좌표 출처·주소 결정(`coordinate_owner`)                                                                                                                      |

흐름 한 줄: `scrapers/` → `extract.py`/`headermap.py` → `records.csv` → `classify.py` → `naver.py`/`licenses.py`(+캐시, `data/manual/`) → `geocode.json` → `closure.json` → `pipeline.marker_candidates()` → `site.py` → `dist/{city}/markers.json` → `app.js`.

기관 단위 실행은 같은 구조가 `data/{city}/orgs/{org}/` 아래에 반복된다.

