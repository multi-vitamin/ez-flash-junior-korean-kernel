# ez-flash-junior-korean-kernel
ezflash junior의 최신 버전(https://www.ezflash.cn/zip/ezjunior-fw5-0918.zip) 커널에 한국어 지원을 추가.

CODEX GPT 5.5를 이용해서 기능을 구현함.

## 변경 요약

최신 `ezgb.dat`의 원래 브라우저 렌더러 흐름을 최대한 유지하면서, 파일명/폴더명에 CP949 한글을 표시하고 한글 LFN 경로로 폴더 진입, ROM 실행, `.sav` 생성/로드가 가능하도록 패치.

주요 변경 사항.

- CP949 한글 2350자 매핑 테이블 추가
- Galmuri 7 기반 8x8 한글 비트맵 폰트 추가
- 파일명 렌더러에 CP949 lead/trail 처리 추가
- 포커스 상태의 파란 배경이 한글 뒤에도 채워지도록 한글 8x8 셀 전체를 그리게 수정
- 포커스 파일명 플로우가 한글 2바이트 중간에서 시작하지 않도록 표시 글자 단위 스크롤 helper 추가
- LFN 경로 변환에서 CP949 2바이트 이름을 유지하도록 `ff_convert` wrapper 및 경로 parser 보강

## 주요 변경 영역

| 영역 | 오프셋 | 길이 | 목적 |
| --- | --- | ---: | --- |
| ROM0 한글 draw helper | `0x00200-0x003CB` | `0x1CC` | Galmuri 8x8 한글 glyph를 최신 ASCII 렌더러와 같은 셀/배경 방식으로 VRAM에 기록 |
| ROM0 bank1 stack stub | `0x03D8C-0x03DB2` | `0x27` | bank1 파일명 출력 호출을 Galmuri 렌더러로 우회, WRAM scratch 대신 stack으로 복귀 상태 보존 |
| ROM0 focus stack stub | `0x03DD0-0x03DF6` | `0x27` | 포커스/플로우 파일명 출력 호출을 Galmuri 렌더러로 우회 |
| 포커스 CP949 표시 길이 helper | `0x03E00-0x03E22` | `0x23` | CP949 한글 2바이트를 화면 1칸으로 계산 |
| 포커스 CP949 advance helper | `0x03E40-0x03E60` | `0x21` | 표시 글자 N칸 이동을 실제 CP949 바이트 포인터로 변환 |
| OEM->Unicode hook | `0x01E37-0x01E7D` | `0x47` | `ff_convert` dir=1 CP949 처리로 우회 |
| Unicode->OEM hook | `0x01E7E-0x01ECF` | `0x52` | `ff_convert` dir=0 CP949 처리로 우회 |
| OEM->Unicode wrapper | `0x03E80-0x03ECF` | `0x50` | CP949 -> Unicode 매핑 table 조회 |
| Unicode->OEM wrapper | `0x03F00-0x03F5B` | `0x5C` | Unicode -> CP949 매핑 table 조회 |
| Bank1 파일명 렌더러 | `0x075F3-0x0771B` | `0x129` | 최신 파일명 렌더 흐름에 CP949 한글 출력 추가 |
| Galmuri font banks | `0x30000-0x3496F` | `0x4970` | CP949 한글 2350자 x 8 bytes |
| CP949 table part0 | `0x08380-0x0BFFF` | `0x3C80` | Unicode -> CP949 table |
| CP949 table part1 | `0x11CD6-0x1305F` | `0x138A` | Unicode -> CP949 table 나머지 |
| Reverse CP949 table | `0x85484-0x866DF` | `0x125C` | CP949 -> Unicode reverse table |

## 주요 hook

| 오프셋 | 패치 후 바이트 | 의미 |
| --- | --- | --- |
| `0x0DD8` | `CD D0 3D` | 포커스 파일명 출력 -> ROM0 focus stack stub |
| `0x0C55` | `CD 00 3E` | 포커스 파일명 길이 계산 -> CP949 표시 길이 helper |
| `0x0D50` | `CD 40 3E` | 플로우 시작 포인터 계산 -> CP949 표시 글자 advance helper |

일반 bank1 파일명 출력 호출은 `0x3D8C` stub을 통해 Galmuri 렌더러로 우회한다.

## LFN/경로 관련 보강

다음 bank의 path parser에 CP949 2바이트 보존 helper를 추가.

- bank `0x03`: helper `0x0FEAF`
- bank `0x05`: helper `0x1767F`
- bank `0x06`: helper `0x1BE4C`
- bank `0x07`: helper `0x1FF0F`
- bank `0x09`: helper `0x27CB7`

dir=0 remap helper는 다음 위치에 적용.

- `0x0EF47`
- `0x15EAA`
- `0x1AD74`
- `0x1F268`
- `0x2715A`

dir=0 string write helper는 다음 위치에 적용.

- `0x15958`
- `0x1ED16`

## 사용법
1. https://www.ezflash.cn/zip/ezjunior-fw5-0918.zip 파일을 받아서 ezflash junior를 gb 파일 실행 후 업데이트. (!!!주의!!! 느린 SD카드 사용, 전원 부족시 문제 생길 가능성 있음)
2. ezgb.dat 파일을 받아서 SD카드 파일 교체.
