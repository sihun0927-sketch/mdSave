지도 마커 관련 파일, 두 분류로 정리:

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

