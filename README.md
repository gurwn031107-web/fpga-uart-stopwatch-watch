# FPGA 스톱워치 / 시계와 UART 통신

Basys-3 FPGA에서 버튼으로 동작하는 스톱워치와 시계를 만들고, PC와 통신하기 위한 UART 송수신기를 직접 설계한 Verilog 프로젝트입니다.
UART는 받은 글자를 그대로 돌려보내는 루프백으로 검증한 뒤, 송수신 사이에 FIFO를 넣어 확장했습니다.
여기서 만든 스톱워치, 시계, UART는 이후 [FPGA UART 센서 통합 시스템](https://github.com/gurwn031107-web/fpga-uart-sensor-integrated-system) 미니 프로젝트에서 UART 명령 제어와 센서까지 합쳐 완성했습니다.

- 기간: 2026.07.31 ~ 2026.08.11
- 보드: Basys-3 (Artix-7, 100MHz)
- 도구: Vivado, Verilog

## 구성

| 폴더 | 내용 |
|---|---|
| [`01_stopwatch_watch`](01_stopwatch_watch) | 스톱워치 + 시계. 제어 유닛(FSM)과 데이터패스를 나누고 버튼 4개, 스위치 2개로 조작 |
| [`02_uart_loopback`](02_uart_loopback) | UART RX / TX 설계 (9600bps, 16배 오버샘플링), 받은 글자를 그대로 다시 보내는 루프백 |
| [`03_uart_fifo_loopback`](03_uart_fifo_loopback) | RX → FIFO → FIFO → TX 구조로 바꿔서 연속으로 들어오는 글자도 놓치지 않게 함 |

## 01. 스톱워치 / 시계

| 입력 | 스톱워치 (SW1 OFF) | 시계 (SW1 ON) |
|---|---|---|
| 왼쪽 버튼 | 시작 / 정지 | 시 증가 |
| 오른쪽 버튼 | 초기화 | 초 증가 |
| 위 버튼 | 표시 전환 | 분 증가 |
| 아래 버튼 | 동작 중 기록 저장, 정지 중 저장한 기록 보기 | 감소 방향으로 전환 |
| SW0 | 표시 전환 (초.1/100초 / 시.분) | 같음 |

- 스톱워치 제어 유닛: `STOP`, `RUN`, `CLEAR`, `MODE`, `SAVE`, `RETURN` 상태
- 시계 제어 유닛: 조정할 자리(`SELECT_SEC` / `MIN` / `HOUR`)를 고르고 증가 / 감소 펄스 출력
- 버튼은 디바운스 회로를 거쳐 1클럭 펄스로 바꿔서 사용
- FND는 4자리를 빠르게 번갈아 켜는 다이나믹 구동, 1/100초 단위 tick은 100Hz tick generator로 만듦

## 02. UART 송수신

- `baud_tick_x16`: 100MHz를 나눠 9600bps의 16배 tick 생성
- `uart_rx`: 시작 비트 가운데(8번째 tick)에서 한 번 확인하고, 이후 16 tick마다 데이터 비트를 읽음 (`IDLE → START → DATA → STOP`)
- `uart_tx`: 같은 tick으로 시작 비트, 8비트 데이터, 정지 비트를 순서대로 출력
- `uart_loop_back`: RX가 한 글자를 받으면(`rx_done`) 그대로 TX로 보내서 PC 터미널에서 동작 확인

## 03. UART + FIFO

- 포인터 기반 FIFO(레지스터 파일 + 쓰기 / 읽기 포인터, full / empty 판단)를 RX 쪽과 TX 쪽에 하나씩 둠
- RX FIFO가 비어 있지 않으면 TX FIFO로 옮기고, TX가 바쁘지 않을 때 TX FIFO에서 꺼내 전송
- TX가 보내는 동안 다음 글자가 들어와도 FIFO에 쌓여서 순서대로 나감

## 실행

Vivado에서 폴더별로 새 프로젝트를 만들고, `.v` 파일과 `Basys-3-Master.xdc`를 추가한 뒤 `top_stopwatch_watch` / `uart_loop_back` / `uart_fifo_loopback`을 Top으로 지정합니다.
`tb_*.v`로 Behavioral Simulation을 돌리거나, 비트스트림을 만들어 보드에서 확인합니다. UART는 PC 터미널을 9600bps로 열면 됩니다.
