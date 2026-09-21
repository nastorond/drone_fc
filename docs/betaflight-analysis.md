# Betaflight 4.2.0 제어 경로 분석 (VGOODRCF4 / STM32F405 / MPU6000)

- 분석 대상: `betaflight/` (태그 4.2.0, 커밋 8f2d21460), 타깃 `VGOODRCF4`
- 작성일: 2026-09-18
- 목적: 레지스터 수준 자체 펌웨어를 작성할 때 참고할 수 있도록, 부팅부터 모터 출력까지의 경로를 최하위 하드웨어 접근 지점까지 추적한다.
- 경로 표기: 특별한 말이 없으면 `betaflight/src/main/` 기준 상대 경로. `파일:줄` 형식.

> **아직 확인하지 않은 전제.** 아래 "기본값"은 소스코드의 기본값이다. 대조군 기체가 실제로 쓰는 설정(`motor_pwm_protocol`, 필터, F 게인 등)은 `baseline/`의 `diff all` 출력으로 확인해야 한다. 이 글을 쓰는 시점에 `baseline/` 폴더는 비어 있다(디렉터리 자체가 없음).

---

## 0. 요약: 자체 펌웨어 설계에 직접 영향을 주는 사실

| # | 사실 | 근거 |
|---|------|------|
| 1 | 8kHz 자이로/PID 루프는 **인터럽트가 아니라 메인 루프 폴링**으로 돈다. `scheduler()`가 `micros()`로 125us 경과 여부를 확인해 실행한다. | `scheduler/scheduler.c:338-354` |
| 2 | MPU6000의 data-ready 핀(PC3)으로 EXTI 인터럽트를 설정하지만, 핸들러는 `dataReady` 플래그만 세운다. 4.2.0에서는 **이 플래그를 확인하는 코드가 없다.** 자이로는 매 주기 무조건 읽는다. 따라서 자이로 샘플 클럭(MPU 내부 발진기)과 MCU 루프 클럭은 동기화되지 않는다. | `drivers/accgyro/accgyro_mpu.c:109-119`, `sensors/gyro.c:381-386` |
| 3 | 자이로 SPI 읽기는 **DMA 없이** TXE/RXNE 폴링으로 7바이트를 주고받는 블로킹 전송이다. | `drivers/bus_spi_stdperiph.c:145-179` |
| 4 | 자이로 → 필터 → PID → 믹서 → DShot DMA 시작까지 **한 번의 `scheduler()` 호출 안에서 순차 실행**된다. 모터 출력만 DMA로 하드웨어가 내보낸다. | `scheduler/scheduler.c:344-350`, `fc/core.c:1272-1295` |
| 5 | CLI `status`의 `CPU:6%`는 **CPU 시간 점유율이 아니다.** 스케줄러가 한 번 돌 때 대기 중이던 비실시간 태스크 개수의 평균 × 100이다. 두 펌웨어의 CPU 사용률을 비교하려면 측정 방법을 따로 정해야 한다(7장). | `scheduler/scheduler.c:132-141`, `cli/cli.c:4775-4776` |
| 6 | 콜드 부팅 때는 **두 번 부팅한다.** 첫 부팅은 HSI 기반 PLL로 168MHz를 만든다. 설정에서 HSE=8MHz를 읽은 뒤 RTC 백업 레지스터에 기록하고 소프트 리셋하며, 두 번째 부팅부터 HSE를 쓴다. | `startup/system_stm32f4xx.c:690-735`, `fc/init.c:535`, `drivers/persistent.c:112-133` |

---

## 1. 대상 하드웨어 매핑 (`target/VGOODRCF4/target.h`, `target.c`)

| 기능 | 핀 / 자원 | 비고 |
|------|-----------|------|
| MCU | STM32F405 (`-DSTM32F40_41xxx -DSTM32F405xx`) | `make/mcu/STM32F4.mk:159` |
| HSE | 8MHz | `Makefile:101` `HSE_VALUE ?= 8000000`, 타깃이 변경하지 않음 |
| 자이로 | MPU6000, **SPI3**, CS=**PA15**, SCK=PB3, MISO=PB4, MOSI=PB5 (AF6) | `target.h:44-46,133-136`, `drivers/bus_spi_pinconfig.c:88-145` |
| 자이로 INT | **PC3** → EXTI3, `USE_MPU_DATA_READY_SIGNAL` | `target.h:36-40` |
| 자이로 정렬 | `CW180_DEG` → (x, y, z) → (-x, -y, z) | `target.h:34`, `sensors/boardalignment.c:99-103` |
| LED | LED0=PC14, LED1=PC15 (active-low) | `target.h:26-27`, `drivers/light_led.c:89-93` |
| 비퍼 | PC13, inverted | `target.h:29-31` |
| 모터 S1~S4 | PA9 TIM1_CH2, PA10 TIM1_CH4, PB0 TIM3_CH3, PB1 TIM3_CH4 | `target.c:31-35` |
| 모터 S5, S6 | PC8 TIM8_CH3, PC9 TIM8_CH4 | `target.c:36-37` |
| OSD | MAX7456, SPI1, CS=PA4 | `target.h:56-59` |
| 수신기 (기체 설정) | CRSF, UART5 (PC12/PD2) | CLAUDE.md. 타깃 기본값은 SBUS/UART6 |
| PINIO | PA13 (VTX 전원), PA14 (카메라 전환) | `target.h:122-124` → **SWD 핀이 PINIO로 쓰인다** |

주의할 핀:
- **PA15 / PB3 / PB4**는 리셋 후 JTAG 핀(JTDI / JTDO·SWO / NJTRST, AF0)이다. SPI3로 쓰려면 MODER와 AFR을 반드시 다시 설정해야 한다.
- **PC13~15**는 백업 도메인 핀이다. 출력 속도 2MHz 이하, 싱크 3mA 이하, 전류 소스 용도 금지.
- **PA13/PA14**를 Betaflight가 PINIO로 쓰므로, 이 보드에서 SWD 디버깅은 기대하기 어렵다. 굽기는 USB DFU(부트 버튼)로 하게 될 가능성이 높다.

---

## 2. 부팅부터 메인 루프까지

### 2.1 전체 순서

```
전원/리셋
 └ 벡터 테이블[0] → MSP, [1] → Reset_Handler         (FLASH 0x0800_0000)
    Reset_Handler                                     startup/startup_stm32f40xx.s:77
     ├ RCC_AHB1ENR |= CCMDATARAMEN (0x4002_3830 |= 1<<20)   :80-84
     ├ persistentObjectInit()   RTC 백업 레지스터 준비       drivers/persistent.c:112
     ├ checkForBootLoaderRequest()   DFU 진입 요청 확인
     ├ .data 복사, .bss 0 채움, FASTRAM(.fastram_bss) 0 채움
     ├ 힙/스택 영역을 0xA5A5A5A5로 채움 (스택 사용량 측정용)
     ├ CPACR(0xE000_ED88) |= 0xF<<20   FPU 활성화
     ├ SystemInit()                                  startup/system_stm32f4xx.c:508
     │   RCC 리셋 상태로, VTOR 설정. (여기서는 SetSysClock 호출이 주석 처리돼 있음 :541)
     └ main()                                        main.c:32
        ├ init()                                     fc/init.c:309
        │  ├ systemInit()                            drivers/system_stm32f4xx.c:148
        │  │  ├ SetSysClock()    ← 실제 168MHz 설정은 여기서
        │  │  ├ NVIC_PriorityGroupConfig(Group 2)
        │  │  ├ cycleCounterInit()  DWT CYCCNT 활성화   drivers/system.c:56
        │  │  └ SysTick_Config(SystemCoreClock/1000)  1ms
        │  ├ IOInitGlobal, pgResetAll, initEEPROM, readEEPROM
        │  ├ ledInit, EXTIInit                       fc/init.c:470
        │  ├ systemClockSetHSEValue(8MHz) → 필요하면 리셋   fc/init.c:535
        │  ├ timerInit                               fc/init.c:559
        │  ├ serialInit
        │  ├ mixerInit / mixerConfigureOutput        fc/init.c:583
        │  ├ motorDevInit  (DShot 타이머/DMA 설정)    fc/init.c:597
        │  ├ configureSPIAndQuadSPI → spiInit(SPIDEV_1/2/3)
        │  ├ sensorsAutodetect → gyroInit → mpuDetect → mpu6000SpiGyroInit   fc/init.c:726
        │  ├ gyroSetTargetLooptime(pid_process_denom) fc/init.c:741,747
        │  ├ gyroInitFilters                         fc/init.c:750
        │  ├ pidInit                                 fc/init.c:752
        │  ├ imuInit, failsafeInit, rxInit, ...
        │  ├ gyroStartCalibration                    fc/init.c:874
        │  ├ timerStart                              fc/init.c:909
        │  ├ mspInit, osdInit, ...
        │  ├ motorPostInit, motorEnable              fc/init.c:1037
        │  └ tasksInit()  태스크 주기 등록           fc/init.c:1044 → fc/tasks.c
        └ run(): while(true) { scheduler(); processLoopback(); }   main.c:42-49
```

### 2.2 클럭 설정 (`startup/system_stm32f4xx.c:690` `SetSysClock`)

HSE=8MHz일 때 계산 과정:
- `pll_m = hse_mhz / 2 = 4` → PLL 입력 2MHz (:729-733)
- `overclockLevels[0] = {168, 336, 2, 7}` (1MHz 입력 기준 n) → `pll_n = 336 / pll_input(2) = 168` (:438-441, :476)

| 레지스터 | 설정 | 값 / 의미 | 코드 |
|----------|------|-----------|------|
| RCC_CR | HSEON, HSERDY 대기 (타임아웃 5000us) | | :722-725 |
| RCC_APB1ENR | PWREN | | :739 |
| PWR_CR | VOS=1 (Scale 1) | 168MHz 동작 조건 | :740 |
| RCC_CFGR | HPRE=/1, PPRE2=/2, PPRE1=/4 | HCLK 168, APB2 84, APB1 42MHz | :743-750 |
| RCC_PLLCFGR | PLLM=4, PLLN=168 (<<6), PLLP 필드=0 (/2, <<16), PLLSRC=HSE, PLLQ=7 (<<24) | VCO 336MHz, SYSCLK 168, USB 48 | :775 |
| RCC_CR | PLLON, PLLRDY 대기 | | :779-784 |
| FLASH_ACR | PRFTEN, ICEN, DCEN, LATENCY_5WS | | :801 |
| RCC_CFGR | SW=PLL, SWS=PLL 대기 | | :810-814 |

파생 클럭:
- APB1 타이머(TIM2~7, 12~14) = 84MHz, APB2 타이머(TIM1, 8~11) = 168MHz (`drivers/timer_stm32f4xx.c:227-237`)
- SPI1 = APB2 84MHz, SPI2/3 = APB1 42MHz

**콜드 부팅 두 번 부팅 동작**
1. 파워온 리셋이면 `persistentObjectInit()`이 백업 레지스터를 모두 0으로 지운다 (`drivers/persistent.c:128-133`).
2. `SetSysClock()`은 백업 레지스터에서 HSE 값을 읽는데 0이다. 그래서 HSI(16MHz, M=8)로 PLL을 설정한다 (:710-719).
3. `init()`에서 설정의 `hseMhz`(=8)를 읽고 `systemClockSetHSEValue(8000000)`을 호출한다. 백업 레지스터 값과 다르므로 그 값을 기록하고 `NVIC_SystemReset()`한다 (`startup/system_stm32f4xx.c:497-506`).
4. 소프트 리셋 뒤에는 백업 레지스터가 보존되므로 HSE로 PLL을 잡는다.

자체 펌웨어는 처음부터 HSE를 쓰면 된다. 부팅 시간 측정을 비교할 때는 이 차이를 고려해야 한다.

### 2.3 코어 설정 (`drivers/system_stm32f4xx.c:148-175`, `drivers/system.c`)

| 항목 | 값 | 근거 |
|------|----|------|
| NVIC 우선순위 그룹 | Group 2 (선점 2비트, 서브 2비트) | `drivers/nvic.h:71-80` |
| SysTick | `SystemCoreClock/1000` → LOAD=167999, 1ms | `drivers/system_stm32f4xx.c:174` |
| DWT CYCCNT | DEMCR.TRCENA=1, LAR=0xC5ACCE55, CTRL.CYCCNTENA=1 | `drivers/system.c:56-82` |
| `micros()` | `ms*1000 + (usTicks*1000 - SysTick->VAL)/usTicks`, usTicks=168 | `drivers/system.c:131-145` |

`micros()`는 SysTick 카운터를 읽어 1us 해상도를 만든다. 스케줄러의 모든 시간 판단이 이 함수에 의존한다.

### 2.4 메모리 배치 (`src/link/stm32_flash_f405.ld`, `target/common_pre.h:152-167`)

| 영역 | 주소 | 용도 |
|------|------|------|
| FLASH 섹터 0 (16K) | 0x0800_0000 | 벡터, 스타트업 (10K) + 커스텀 기본값 (6K) |
| FLASH 섹터 1 (16K) | 0x0800_4000 | **설정(EEPROM 에뮬레이션)** |
| FLASH | 0x0800_8000~ | 펌웨어 본체 |
| SRAM 128K | 0x2000_0000 | 일반 데이터, 벡터 테이블 복사본(VECTAB) |
| CCM 64K | 0x1000_0000 | **스택**, `FAST_RAM`/`FAST_RAM_ZERO_INIT` 변수 |

`FAST_CODE`는 F4에서 빈 매크로라서 코드는 FLASH에서 실행된다(ART 가속기 사용). F7의 ITCM에만 의미가 있다. CCM은 DMA가 접근할 수 없으므로 DMA 버퍼는 SRAM에 있어야 한다.

---

## 3. 스케줄러: 8kHz 루프가 도는 방식

### 3.1 구조

`run()`은 인터럽트 없이 `scheduler()`를 무한 반복 호출한다(`main.c:42-49`). 선점형 RTOS가 아니고 협조형(cooperative)이다. 태스크는 끝까지 실행되며 중간에 끊기지 않는다.

`scheduler()` (`scheduler/scheduler.c:326-441`) 한 번의 호출:

```c
now = micros();
if (gyroEnabled) {
    gyroTask = TASK_GYRO;
    due = periodBasis(gyroTask) + gyroTask->desiredPeriodUs;   // 125us
    gyroTaskDelayUs = due - now;
    if (now >= due) {
        schedulerExecuteTask(TASK_GYRO)                 // 자이로 읽기
        if (gyroFilterReady())  schedulerExecuteTask(TASK_FILTER)  // 필터
        if (pidLoopReady())     schedulerExecuteTask(TASK_PID)     // PID+믹서+모터
        realtimeTaskRan = true;
    }
}
if (!gyroEnabled || realtimeTaskRan || gyroTaskDelayUs > 10us) {
    // 비실시간 태스크 중 동적 우선순위가 가장 높은 것 하나 선택
    // 예상 실행시간(이동평균 + 5us) < gyroTaskDelayUs 일 때만 실행
}
```

- 실시간 태스크 3개(GYRO/FILTER/PID)는 `TASK_PRIORITY_REALTIME`이고 동적 우선순위 계산에서 빠진다(`:361`).
- `GYRO_TASK_GUARD_INTERVAL_US = 10`: 다음 자이로 시점까지 10us 이하로 남으면 다른 태스크를 시작하지 않는다(`scheduler/scheduler.h:30`, `scheduler.c:356`).
- 비실시간 태스크는 예상 실행시간이 다음 자이로 시점까지 남은 시간보다 짧을 때만 실행한다(`:415-427`). 예상 실행시간은 `movingSumExecutionTimeUs/32 + 5us`이다(`:418`, `TASK_STATS_MOVING_SUM_COUNT=32`).
- 한 번의 `scheduler()` 호출에서 비실시간 태스크는 **최대 1개**만 실행된다.

### 3.2 동적 우선순위 (`scheduler.c:360-409`)

- 시간 기반 태스크: `taskAgeCycles = (now - lastExecutedAt) / desiredPeriod`. 1 이상이면 `dynamicPriority = 1 + staticPriority * taskAgeCycles`.
- 이벤트 기반 태스크(`checkFunc`가 있는 것, 예: RX): `checkFunc`가 true를 반환하면 대기 상태가 되고, 대기 중인 동안 나이에 비례해 우선순위가 오른다.
- 정적 우선순위: IDLE=0, LOW=1, MEDIUM=3, MEDIUM_HIGH=4, HIGH=5 (`scheduler.h:39-45`).

### 3.3 125us는 어디서 오는가

```
gyroSetSampleRate()               drivers/accgyro/gyro_sync.c:50
  MPU6000 → default 분기: gyroSampleRateHz = 8000, accSampleRateHz = 1000   :80-83
  mpuDividerDrops = 0                                                      :87
gyroSetTargetLooptime(pid_process_denom = 1)   sensors/gyro_init.c:666
  gyro.sampleLooptime = 1e6 / 8000 = 125
  gyro.targetLooptime = 1 * 1e6 / 8000 = 125
tasksInit()                       fc/tasks.c:264-271
  rescheduleTask(TASK_GYRO,   125)
  rescheduleTask(TASK_FILTER, 125)
  rescheduleTask(TASK_PID,    125)
  schedulerEnableGyro()
```

`pid_process_denom` 기본값은 F405에서 1이다(`flight/pid.c:96-104`). 그래서 `activePidLoopDenom = 1`이고, `gyroFilterReady()`와 `pidLoopReady()`는 매번 true다(`fc/core.c:1238-1263`). denom이 2 이상이면 자이로는 매번 읽고 누적하다가 필터는 denom번에 1번, PID는 그 중간 시점에 실행한다.

### 3.4 주기 기준점: `lastExecutedAtUs`와 `lastDesiredAt`

`getPeriodCalculationBasis()` (`scheduler.c:269-276`):
- 기본(`scheduler_optimize_rate = AUTO`이고 DShot 텔레메트리가 꺼져 있을 때): **`lastExecutedAtUs`** 기준. 다음 실행 시각 = 실제 실행 시각 + 125us이므로 늦게 실행되면 이후 주기가 전부 밀린다. 평균 주기가 125us보다 길어질 수 있다.
- `OPTIMIZE_RATE ON`(또는 AUTO + DShot 텔레메트리): **`lastDesiredAt`** 기준. 이상적인 격자에 맞추므로 평균 주파수가 유지된다(`schedulerExecuteTask` `:286`).
- 설정 근거: `config/config.c:163`.

대조군이 어느 쪽으로 동작하는지는 `diff all`의 `scheduler_optimize_rate`와 `dshot_bidir`로 확인해야 한다.

### 3.5 지터가 생기는 원인 (구조상 추정)

1. 폴링 방식이라, 자이로 실행 시점이 늦어지는 정도는 앞서 실행 중이던 비실시간 태스크의 남은 실행시간에 좌우된다. 예상 실행시간 가드가 있지만 예측이 틀리면(최대값이 평균보다 길 때) 늦어진다.
2. `micros()` 해상도는 1us다.
3. ISR(UART RX, USB, DShot DMA TC, SysTick)이 실시간 태스크 도중에 끼어든다.
4. MPU6000 내부 8kHz와 MCU 125us가 서로 다른 클럭이라 비트(beat)가 생긴다. 같은 샘플을 두 번 읽거나 하나를 건너뛸 수 있다.

### 3.6 CPU 사용률의 의미 (중요)

```c
// scheduler.c:132-141
averageSystemLoadPercent = 100 * totalWaitingTasks / totalWaitingTasksSamples;
```
- `totalWaitingTasks`: 비실시간 태스크 선택 블록이 돌 때마다 "실행 대기 상태인 태스크 수"를 더한 값
- `totalWaitingTasksSamples`: 그 블록이 실행된 횟수
- `TASK_SYSTEM`(10Hz)이 이 값을 갱신하고, CLI `status`의 `CPU:%d%%`와 MSP가 이 값을 보고한다(`cli/cli.c:4775`, `msp/msp.c:1012`).

따라서 **`CPU:6%`는 CPU가 일한 시간의 비율이 아니다.** 비교 실험에서는 다음 방법을 권한다.
- 대조군: CLI `tasks` 명령이 태스크별 `max/us`, `avg/us`, `avgload`를 출력한다(`cli/cli.c:4815-4846`, `USE_TASK_STATISTICS`는 기본 활성, `target/common_pre.h:201`). 실시간 태스크 3개의 avg/us 합 ÷ 125us가 제어 루프가 차지하는 CPU 시간 비율이다.
- 자체 펌웨어: DWT CYCCNT로 루프 구간별 사이클을 직접 측정한다.
- 논문에는 두 지표의 정의를 분리해서 적는다.

### 3.7 태스크 표 (기본값, `fc/tasks.c:395-470`)

| 태스크 | 주기 | 정적 우선순위 | 비고 |
|--------|------|---------------|------|
| GYRO / FILTER / PID | 125us | REALTIME | 스케줄러 앞단에서 별도 처리 |
| ACC | 1kHz | MEDIUM | |
| ATTITUDE | 100Hz | MEDIUM | IMU(자세 추정), 무장 시 등 500Hz로 변경 가능 (`fc/core.c:998`) |
| RX | 33Hz (이벤트 기반) | HIGH | `rxUpdateCheck`가 새 프레임을 감지하면 실행 |
| DISPATCH | 1kHz | HIGH | |
| SYSTEM (LOAD) | 10Hz | MEDIUM_HIGH | 위 CPU% 계산 |
| MAIN (UPDATE) | 1kHz | MEDIUM_HIGH | |
| SERIAL | 100Hz | LOW | MSP/CLI |
| OSD | 60Hz | LOW | MAX7456 (SPI1) |
| TELEMETRY | 250Hz | LOW | |
| BATTERY_* | 5~50Hz | MEDIUM | |

---

## 4. MPU6000 자이로 읽기 경로

### 4.1 호출 계층 (위에서 아래로)

```
scheduler()                                   scheduler/scheduler.c:344
 └ taskGyroSample()                           fc/core.c:1238
    └ gyroUpdate()                            sensors/gyro.c:411
       └ gyroUpdateSensor(&gyroSensor1)       sensors/gyro.c:381
          └ gyroDev.readFn = mpuGyroReadSPI   drivers/accgyro/accgyro_mpu.c:180
             │  tx = {0x43|0x80, 0xFF x6}  (GYRO_XOUT_H 부터 6바이트 읽기)
             └ spiBusTransfer()               drivers/bus_spi.c:139
                ├ IOLo(CS=PA15)   → GPIOA->BSRR = 1<<(15+16)
                ├ spiTransfer(SPI3, tx, rx, 7)   drivers/bus_spi_stdperiph.c:145
                │   for each byte:
                │     while(!(SPI3->SR & TXE));   SPI3->DR = b;
                │     while(!(SPI3->SR & RXNE));  b = SPI3->DR;
                └ IOHi(CS=PA15)   → GPIOA->BSRR = 1<<15
```

최하위 하드웨어 접근: `SPI3->SR`, `SPI3->DR`(StdPeriph `SPI_I2S_GetFlagStatus`, `SPI_I2S_SendData`, `SPI_I2S_ReceiveData`), `GPIOA->BSRR`.

- **DMA: 사용하지 않는다.** (SPI DMA는 4.3에서 도입됐다. 4.2.0의 `bus_spi.c`와 `bus_spi_stdperiph.c`에는 DMA 코드가 없다.)
- **SPI 인터럽트: 사용하지 않는다.** CR2=0.
- 바이트마다 RXNE를 기다린 뒤 다음 바이트를 쓰므로 바이트 사이에 공백이 생긴다. 21MHz에서 순수 클럭 시간은 56비트 × 47.6ns = 2.67us이고, 실제로는 수 us가 걸릴 것으로 추정한다(실측 필요).
- 타임아웃: 바이트당 폴링 1000회. 넘으면 `spiTimeoutUserCallback`으로 에러 카운트를 올린다.

### 4.2 SPI3 초기화 (`drivers/bus_spi_stdperiph.c:47-110`)

| 단계 | 레지스터 수준 동작 |
|------|--------------------|
| 클럭 | `RCC->APB1ENR |= SPI3EN`, `RCC->APB1RSTR` 토글 (리셋) |
| 핀 | PB3/PB4/PB5: MODER=AF(10), OTYPER=PP, OSPEEDR=50MHz(10), PUPDR=없음, AFRL=6 (`SPI_IO_AF_CFG`, `drivers/bus_spi.h:32`) |
| CS | PA15: MODER=출력(01), PP, 50MHz, 풀 없음, 초기값 HIGH (`accgyro_mpu.c:239-242`, `SPI_IO_CS_CFG`) |
| CR1 | MSTR=1, SSM=1, SSI=1, DFF=0(8비트), LSBFIRST=0, **CPOL=1, CPHA=1 (SPI 모드 3)**, BR은 아래 표 |
| 기타 | CRCPR=7(사용 안 함), CR2=0, SPE=1 |

- 모드 3이 되는 이유: `requiresSpiLeadingEdge(SPIDEV_3)`가 false이기 때문(SD카드/RX_SPI 없음, `fc/init.c:214-243`). 그러면 `CPOL_High/CPHA_2Edge`로 설정된다(`bus_spi_stdperiph.c:91-94`).
- CR1 값: 초기화용 속도에서 `0x0377`, 동작 속도에서 `0x0347` (MSTR|SSI 0x0104 + SSM 0x0200 + CPOL 0x0002 + CPHA 0x0001 + SPE 0x0040 + BR). 이 값은 StdPeriph 상수로 계산한 것이다.

**분주비** (`drivers/bus_spi.h:53-58`, `bus_spi_stdperiph.c:181-211`)

`spiDivisorToBRbits()`는 SPI2/3이면 요청한 분주비를 2로 나눈다. APB1이 APB2의 절반이라서 같은 SCK를 얻기 위한 보정이다. `BR = (ffs(div) - 2) << 3`.

| 용도 | 요청 divisor | SPI3 실제 분주 | BR[2:0] | SCK |
|------|--------------|----------------|---------|-----|
| `SPI_CLOCK_INITIALIZATION` (감지, 레지스터 쓰기) | 256 | 128 | 110 | 42MHz/128 = **328kHz** |
| `SPI_CLOCK_FAST` (데이터 읽기) | 4 | 2 | 000 | 42MHz/2 = **21MHz** |

MPU6000 데이터시트는 SPI 클럭을 전체 레지스터 1MHz, 센서/인터럽트 레지스터 읽기 20MHz로 규정한다. Betaflight는 읽기에 21MHz를 쓴다(규격 +5%). 자체 펌웨어에서 20MHz 이하를 지키려면 SPI3에서는 /4 = 10.5MHz가 가능한 다음 값이다.

`spiSetDivisor()`는 SPE를 끄고 BR을 바꾼 뒤 SPE를 켠다(`:206-211`).

### 4.3 MPU6000 초기화 시퀀스

감지 (`drivers/accgyro/accgyro_spi_mpu6000.c:127` `mpu6000SpiDetect`):

| 순서 | 동작 | 대기 |
|------|------|------|
| 0 | `mpuDetect()` 진입 시 전원 안정화 대기 | 35ms (`accgyro_mpu.c:278-279`) |
| 1 | PWR_MGMT_1(0x6B) ← 0x80 (H_RESET) | |
| 2 | WHO_AM_I(0x75) 읽기 == 0x68 까지 반복 (최대 6회) | 회당 150ms |
| 3 | PRODUCT_ID(0x0C) 읽기, 0x14~0x18 / 0x54~0x5A 중 하나인지 확인 | |

초기화 (`mpu6000AccAndGyroInit` `:171-214` → `mpu6000SpiGyroInit` `:101-119`). 모두 328kHz에서 수행:

| 순서 | 레지스터 | 값 | 의미 | 줄 |
|------|----------|----|------|----|
| 1 | PWR_MGMT_1 0x6B | 0x80 | 디바이스 리셋 | :176, 이후 150ms |
| 2 | SIGNAL_PATH_RESET 0x68 | 0x07 | gyro/acc/temp 신호 경로 리셋 | :179, 이후 150ms |
| 3 | PWR_MGMT_1 0x6B | 0x03 | 클럭 소스 = PLL(Z 자이로) | :183, 15us |
| 4 | USER_CTRL 0x6A | 0x10 | I2C_IF_DIS (SPI 전용) | :187, 15us |
| 5 | PWR_MGMT_2 0x6C | 0x00 | 모든 축 활성 | :190, 15us |
| 6 | SMPLRT_DIV 0x19 | 0x00 | 분주 없음 → 8kHz (DLPF_CFG=0일 때) | :195, 15us |
| 7 | GYRO_CONFIG 0x1B | 0x18 | ±2000dps | :199, 15us |
| 8 | ACCEL_CONFIG 0x1C | 0x18 | ±16g | :203, 15us |
| 9 | INT_PIN_CFG 0x37 | 0x10 | INT_RD_CLEAR(아무 레지스터나 읽으면 INT 클리어), active-high, push-pull, 50us 펄스 | :206, 15us |
| 10 | INT_ENABLE 0x38 | 0x01 | DATA_RDY_EN | :210, 15us |
| 11 | CONFIG 0x1A | 0x00 | DLPF_CFG=0 (자이로 대역폭 256Hz, 출력 8kHz) | :110 (`mpuGyroDLPF` → 0, `accgyro_mpu.c:339-362`) |
| 12 | SPI → 21MHz, 한 번 읽어 보고 X/Y가 모두 0xFF면 실패 | | :113-118 |

스케일: 16.4 LSB/(°/s) → `gyro->scale = 1/16.4` (`:239`). 가속도 `acc_1G = 2048` (±16g, `:124`).

### 4.4 인터럽트 (EXTI) 설정과 실제 사용 여부

설정 (`accgyro_mpu.c:125-144` → `drivers/exti.c:121-199`):

| 대상 | 설정 |
|------|------|
| PC3 | 입력, 플로팅 (`IOCFG_IN_FLOATING`) |
| SYSCFG_EXTICR1 | EXTI3[3:0] = 0010 (포트 C) |
| EXTI_RTSR | bit3 = 1 (상승 에지) |
| EXTI_IMR | bit3 = 1 |
| NVIC | EXTI3_IRQn, 우선순위 `NVIC_BUILD_PRIORITY(0xF,0xF)` → 0xF0 (가장 낮음) (`drivers/nvic.h:31`) |

핸들러 `mpuIntExtiHandler` (`accgyro_mpu.c:109-119`)는 `gyro->dataReady = true`만 한다. 소비하는 쪽을 찾아보면 다음과 같다.
- `gyroSyncCheckUpdate()`(`drivers/accgyro/gyro_sync.c:38`)는 정의만 있고 호출하는 곳이 없다.
- `gyroUpdateSensor()`는 `readFn()`을 조건 없이 호출한 뒤 `dataReady = false`로 지우기만 한다(`sensors/gyro.c:383-386`).

즉 4.2.0에서 EXTI는 **활성화돼 있지만 동기화에는 쓰이지 않는다.** 인터럽트는 8kHz로 계속 발생해 약간의 CPU 부하만 만든다. 자체 펌웨어에서는 두 가지 중 하나를 고를 수 있다.
- (a) Betaflight처럼 타이머 기준으로 폴링한다. 대조군과 동일한 조건이다.
- (b) EXTI를 트리거로 SPI 읽기를 시작한다. 샘플과 루프가 동기화된다. 차이가 생기면 논문에 명시한다.

### 4.5 원시값 → °/s (`sensors/gyro.c:381-430`)

```
gyroADCRaw[i] = int16(data[2i+1] << 8 | data[2i+2])
gyroADC[i]    = gyroADCRaw[i] - gyroZero[i]            (캘리브레이션 오프셋)
alignSensorViaRotation(CW180): x=-x, y=-y
gyro.gyroADC[i] = gyroADC[i] * (1/16.4)                 [°/s]
sampleSum[i]  = LPF2(gyro.gyroADC[i])                   (lowpass2가 켜져 있으면 다운샘플 필터 역할)
```
캘리브레이션: `gyroCalibrationDuration = 125` → 1.25s → `125*10000/125us = 10000` 샘플을 평균한다(`sensors/gyro.c:108,174`). 움직임 임계값 48(`:109`)을 넘으면 다시 시작한다.

---

## 5. 필터 → PID → 믹서 → 모터

### 5.1 자이로 필터 체인 (`sensors/gyro_filter_impl.c`, 기본값 `sensors/gyro.c:108-134`)

한 축에 대해 적용되는 순서:

| 순서 | 필터 | 기본값 | 실행 위치 |
|------|------|--------|-----------|
| 1 | gyro LPF2 (다운샘플) | PT1 250Hz, dT=sampleLooptime | `gyroUpdate()` (GYRO 태스크) |
| 2 | RPM 필터 | DShot 텔레메트리가 켜져 있을 때만 | `gyro_filter_impl.c:65-67` |
| 3 | 정적 노치 1, 2 | 0Hz (꺼짐) | `:72-73` |
| 4 | gyro LPF1 | **동적** PT1, 200~500Hz (스로틀에 따라 변함) | `:74` |
| 5 | 동적 노치 ×2 | FFT 기반, 150~600Hz, 폭 8%, Q=120 | `:84-86` |

- 출력: `gyro.gyroADCf[axis]` [°/s]가 PID 입력이다.
- PT1: `k = dT/(RC+dT)`, `RC = 1/(2πf)`, `y += k(x-y)` (`common/filter.c:47-68`)
- 동적 LPF 컷오프 (`sensors/gyro.c:621-643`):
  `f = max(dynThrottle(t) * 500, 200)`, `dynThrottle(t) = 1.5·t·(1 - t²/3)`.
  스로틀을 100단계로 양자화하고, 최소 5ms 간격으로 갱신한다(`flight/mixer.c:70-71,811-829`).
- 동적 노치: `FEATURE_DYNAMIC_FILTER`가 기본 활성이다(`config/feature.c:35`). 32포인트 FFT를 축당 4단계로 나눠 계산한다(`flight/gyroanalyse.c`, `gyroanalyse.h:27`).

### 5.2 PID (`flight/pid.c:1273` `pidController`)

**계수 변환** (`flight/pid.c:625-628`, 상수 `flight/pid.h:39-45`):

```
Kp = 0.032029 × P,  Ki = 0.244381 × I,  Kd = 0.000529 × D,  Kf = 0.013754 × F/100
dT = targetLooptime × 1e-6 = 125e-6 s,  pidFrequency = 8000 Hz   (pid.c:246-251)
```

대조군 profile 0 값 (CLAUDE.md)을 대입하면:

| 축 | P / I / D | Kp | Ki | Kd |
|----|-----------|----|----|----|
| roll | 32 / 50 / 19 | 1.0249 | 12.219 | 0.010051 |
| pitch | 34 / 55 / 19 | 1.0890 | 13.441 | 0.010051 |
| yaw | 46 / 51 / 0 | 1.4733 | 12.463 | 0 |

**한 축의 계산** (단순화. 모드 = acro, 크래시 회복/런치 컨트롤 제외):

```
setpoint  = getSetpointRate(axis)                 [°/s]  (fc/rc.c:107, RC 레이트 커브 적용)
gyroRate  = gyro.gyroADCf[axis]
error     = setpoint - gyroRate
(iterm_relax: 스틱이 빠르게 움직이면 I 적분에 들어가는 오차를 줄임)

P = Kp · error · tpa
I = clamp(I + (Ki·dT + agGain) · itermError, ±itermLimit(400))
      yaw 축만 dynCi(모터 포화 시 적분 감소) 적용
dGyro = LPF2_D(LPF1_D(notch_D(gyroRate)))         D 전용 필터
D = Kd · (-(dGyro - dGyro_prev) · 8000) · tpa · dMinFactor   ← 측정값 미분 (derivative on measurement)
F = Kf · (setpoint - setpoint_prev) · 8000         (acro 모드에서만)
Sum = P + I + D + F
```

- D 필터 기본값: 동적 PT1 70~170Hz + PT1 150Hz (`pid.c:195-203`)
- D_min: 기본값은 roll 23 / pitch 25지만 대조군은 0이라 꺼져 있다 → `dMinFactor = 1`
- 스로틀 0 / 비무장 상태에서는 `pidStabilisationEnabled = false`로 모든 항이 0이 된다(`pid.c:1606-1617`).

### 5.3 믹서 (`flight/mixer.c:830` `mixTable`, `:751` `applyMixToMotors`)

```
r = clamp(Sum_roll,  ±500) / 1000          (pidSumLimit=500, PID_MIXER_SCALING=1000)
p = clamp(Sum_pitch, ±500) / 1000
y = -clamp(Sum_yaw,  ±400) / 1000          (yaw_motors_reversed=false 이면 부호 반전)
mix[i] = r·M[i].roll + p·M[i].pitch + y·M[i].yaw
QuadX M (mixer.c:99-104):   throttle roll pitch yaw
   0 REAR_R                   1      -1    1    -1
   1 FRONT_R                  1      -1   -1     1
   2 REAR_L                   1       1    1     1
   3 FRONT_L                  1       1   -1    -1
mixRange = max(mix) - min(mix)
if mixRange > 1: mix /= mixRange, (airmode) throttle = 0.5
else (airmode 또는 throttle>0.5): throttle = clamp(throttle, -min(mix), 1-max(mix))
motor[i] = motorOutputMin + motorOutputRange · (mix[i] + throttle)
motor[i] = clamp(motor[i], motorRangeMin, motorRangeMax)
```
- Airmode는 기본 활성이다(`config/feature.c:35`).
- DShot 출력 범위 (`drivers/dshot.c:52-64`): `outputLow = 48 + 1999 × 0.055 = 157.9`(dshot_idle_value 550 = 5.5%), `outputHigh = 2047`. 비무장 = 0 (`DSHOT_CMD_MOTOR_STOP`).

### 5.4 모터 출력: DShot (DMA)

**프로토콜 선택.** `motor_pwm_protocol` 기본값은 `PWM_TYPE_DISABLED`(`pg/motor.c:57`)이므로 대조군은 사용자가 설정한 값을 쓴다. **`diff all`로 확인해야 한다.** 아래는 DShot600을 가정했다. DShot 텔레메트리가 꺼져 있으면 `dshot_bitbang = AUTO`는 비트뱅이 아닌 타이머+DMA 방식을 쓴다(`drivers/motor.c:324-327`). `dshot_burst` 기본값은 OFF다(`target/common_post.h:164`).

**호출 계층:**

```
taskMainPidLoop → subTaskMotorUpdate          fc/core.c:1178
 └ mixTable()  → motor[] (float, 48~2047)
 └ writeMotors() → motorWriteAll(motor)       flight/mixer.c:481, drivers/motor.c:59
    ├ for i: vTable.write(i, value) = dshotWrite → pwmWriteDshotInt(i, lrintf(value))
    │     drivers/dshot_dpwm.c:133, drivers/pwm_output_dshot_shared.c:87
    │   ├ prepareDshotPacket: packet = (value<<1 | telem) << 4 | crc4   drivers/dshot.c:106
    │   │    crc = (p ^ p>>4 ^ p>>8) & 0xF
    │   ├ loadDmaBufferDshot: buf[0..15] = bit ? 14 : 7, buf[16..17] = 0   drivers/dshot_dpwm.c:56
    │   └ DMAx_Streamy->NDTR = 18; CR.EN = 1             (:128-129)
    └ vTable.updateComplete = pwmCompleteDshotMotorUpdate   drivers/pwm_output_dshot.c:162
        for each timer (TIM1, TIM3):
          ARR = 19; CNT = 0; DIER |= CCxDE (해당 채널들)          :180-186
        → 이후 하드웨어가 CC 이벤트마다 CCR ← buf[k] 전송 (DMA)
DMA 전송 완료 IRQ: motor_DMA_IRQHandler                  :196-224
   stream EN=0, DIER.CCxDE=0, TCIF 클리어
```

**타이머 설정** (`pwmDshotMotorHardwareConfig` `drivers/pwm_output_dshot.c:227-440`):

| 항목 | 값 | 근거 |
|------|----|------|
| 심볼 클럭 | DShot600 12MHz, DShot300 6MHz | `drivers/dshot_dpwm.h:28-30` |
| PSC (TIM1, 168MHz) | DShot600: 13, DShot300: 27 | `:306` `timerClock/hz - 1` |
| PSC (TIM3, 84MHz) | DShot600: 6, DShot300: 13 | |
| ARR | 19 (`MOTOR_BITLENGTH=20`) | `dshot_dpwm.h:34` |
| CCR (비트 1 / 비트 0) | 14 / 7 (70% / 35% 듀티) | `dshot_dpwm.h:32-33` |
| 비트 시간 | DShot600: 1.667us, 프레임 16비트 26.7us + 0 두 칸 3.3us | |
| OC 모드 | PWM1, 프리로드 on, 극성 high, idle state set | `:314-325`, `:95-97` |
| 출력 핀 | AF, 50MHz, PP, 풀업 | `:296-297` |
| TIM1/TIM8 | BDTR.MOE=1 (`TIM_CtrlPWMOutputs`) | `:435` |
| DMA | M→P, word/word, MINC, normal, priority high, FIFO on (1/4), PAR=&TIMx->CCRy, NDTR=18 | `:380-403` |
| DMA 요청 | TIMx_DIER.CCxDE (채널별 CC 이벤트) | `drivers/timer.c:898` |
| NVIC | DMA TC, `NVIC_PRIO_DSHOT_DMA` = (2,1) | `drivers/nvic.h:29` |

**DMA 스트림 매핑** (`target.c` DMA opt 0, 표 `drivers/timer_def.h:450-477` F4 부분):

| 모터 | 핀 | 타이머 채널 | DMA | 스트림 | 채널 |
|------|----|-------------|-----|--------|------|
| S1 | PA9 | TIM1_CH2 | DMA2 | 6 | 0 |
| S2 | PA10 | TIM1_CH4 | DMA2 | 4 | 6 |
| S3 | PB0 | TIM3_CH3 | DMA1 | 7 | 5 |
| S4 | PB1 | TIM3_CH4 | DMA1 | 2 | 5 |
| S5 | PC8 | TIM8_CH3 | DMA2 | 2 | 0 |
| S6 | PC9 | TIM8_CH4 | DMA2 | 7 | 7 |

모터 번호와 믹서 인덱스(REAR_R, FRONT_R, ...)의 대응은 `resource`/`diff all`의 `motor` 재매핑이 있는지 확인해야 한다.

### 5.5 한 사이클의 데이터 흐름과 지연 성분

```
t0  MPU6000 내부 샘플 (8kHz, 자체 클럭)
    ↓  (0~125us: 루프와 비동기라 대기 시간이 균등분포로 추정)
t1  scheduler가 due 판단 → taskGyroSample: SPI 7바이트 폴링 읽기
t2  taskFiltering: LPF2 → notch → dyn LPF1 → dyn notch
t3  taskMainPidLoop: RC 처리 → pidController → mixTable
t4  pwmWriteDshotInt ×4 → pwmCompleteDshotMotorUpdate (DMA 시작)
t5  DShot 프레임 전송 완료 (DShot600 약 26.7us)
t6  ESC가 프레임을 받아 출력 갱신 (ESC 펌웨어 쪽, 범위 밖)
```
센서 → 모터 지연을 비교하려면, 두 펌웨어 모두에서 t1과 t4 시점에 GPIO를 토글하거나 DWT 타임스탬프를 남기는 방법이 가장 직접적이다. 대조군은 수정 금지이므로, 대조군 쪽은 외부에서 SPI CS(PA15)와 모터 신호(PA9)를 로직 분석기로 동시에 측정하는 방법을 고려한다.

---

## 6. 자체 펌웨어 작성 시 체크리스트

### 클럭/코어
- [ ] HSE 8MHz, PLL M=4 N=168 P=2 Q=7, FLASH 5WS + PRFTEN/ICEN/DCEN, VOS=1, AHB/1 APB1/4 APB2/2 (→ `firmware/src/clock.c`)
- [ ] FPU 활성화(CPACR)는 float 코드보다 먼저 (Betaflight: `startup/startup_stm32f40xx.s:144-148`)
- [ ] NVIC 우선순위 그룹을 정하고 모든 IRQ 우선순위를 표로 관리
- [ ] DWT CYCCNT 활성화 (타이밍 측정의 기준)

### 자이로
- [ ] PA15/PB3/PB4 JTAG 기본 기능 해제 (MODER/AFR 재설정)
- [ ] SPI3 모드 3, 8비트, 소프트 NSS, 초기화는 1MHz 이하(BR=/64 → 656kHz 또는 /128 → 328kHz)
- [ ] 리셋 → 150ms → 신호 경로 리셋 → 150ms → 레지스터 설정 (4.3 표)
- [ ] 읽기 클럭: 규격(20MHz) 준수 여부 결정 (SPI3에서 21MHz 또는 10.5MHz)
- [ ] 버스트 읽기: 0x43|0x80 + 6바이트. 가속도까지 한 번에 읽으려면 0x3B부터 14바이트
- [ ] EXTI(PC3) 동기화 사용 여부 결정 → 논문에 명시
- [ ] 좌표 변환 CW180: (x, y, z) → (-x, -y, z)

### 루프
- [ ] 시작은 1~2kHz (CLAUDE.md 방침). MPU6000 SMPLRT_DIV 또는 DLPF 설정으로 출력율을 맞출지, 8kHz로 읽고 평균할지 결정
- [ ] 루프 주기 기준: 하드웨어 타이머 인터럽트 또는 EXTI 권장 (Betaflight식 폴링보다 지터가 작음)

### 모터
- [ ] 대조군의 프로토콜 확인 후 동일하게 (DShot300/600)
- [ ] TIM1/TIM8은 BDTR.MOE 필요
- [ ] DMA 버퍼는 SRAM(0x2000_xxxx)에. CCM에 두면 DMA가 동작하지 않는다
- [ ] 비무장 시 0(모터 정지) 프레임을 지속적으로 송신 (ESC 무장 해제 방지)
- [ ] 프로펠러 제거 상태에서만 모터 테스트

---

## 7. 비교 실험을 위한 주의점

1. **CPU 사용률 지표가 다르다** (3.6). 대조군은 `tasks` 명령의 실행시간 통계를 쓰고, 자체 펌웨어는 DWT로 같은 정의(제어 루프 실행시간 ÷ 주기)를 계산한다.
2. **컴파일러가 다르다.** 대조군은 GCC 9.2.1(`make/tools.mk:17-19`), `-O2` 기본 + 일부 `-Ofast` + LTO(`Makefile:151-155`)로 빌드했다. 자체 펌웨어는 현재 GCC 14.3.1(STM32CubeCLT 1.22.0) `-O2`다. 실행시간 비교 시 명시한다.
3. **루프 주기 차이**: 대조군 8kHz / 자체 1~2kHz로 시작 (CLAUDE.md). 필터 컷오프와 D항 게인은 `dT`에 의존하므로 PID 값을 그대로 옮기면 안 된다. 5.2의 `Kp/Ki/Kd` 환산식과 `dT`를 함께 기록한다.
4. **설정 확인**: 이 문서의 기본값 가정(모터 프로토콜, 동적 노치/LPF, airmode, scheduler_optimize_rate, F 게인, TPA)은 `baseline/`의 `diff all`로 검증한 뒤 이 문서를 갱신한다.
5. **복원 준비**: CLAUDE.md에 따라 4.2.0 hex와 `diff all` 백업이 `baseline/`에 있어야 한다. 현재는 없다.

---

## 부록 A. 상수 요약

| 상수 | 값 | 위치 |
|------|----|------|
| `TASK_GYROPID_DESIRED_PERIOD` | 125us | `target/common_pre.h:129` |
| `GYRO_TASK_GUARD_INTERVAL_US` | 10us | `scheduler/scheduler.h:30` |
| `TASK_AVERAGE_EXECUTE_PADDING_US` | 5us | `scheduler/scheduler.c:44` |
| `TASK_STATS_MOVING_SUM_COUNT` | 32 | `scheduler/scheduler.h:33` |
| gyro sample rate (MPU6000) | 8000Hz | `drivers/accgyro/gyro_sync.c:80-83` |
| acc sample rate | 1000Hz | 같은 곳 |
| `PID_PROCESS_DENOM_DEFAULT` (F405) | 1 | `flight/pid.c:104` |
| `PTERM/ITERM/DTERM/FEEDFORWARD_SCALE` | 0.032029 / 0.244381 / 0.000529 / 0.013754 | `flight/pid.h:39-45` |
| `PIDSUM_LIMIT` / `_YAW` | 500 / 400 | `flight/pid.h:33-34` |
| `PID_MIXER_SCALING` | 1000 | `flight/pid.h:31` |
| itermLimit | 400 | `flight/pid.c:179` |
| `DSHOT_MIN/MAX_THROTTLE` | 48 / 2047 | `drivers/dshot.h:27-28` |
| `MOTOR_BIT_0 / BIT_1 / BITLENGTH` | 7 / 14 / 20 | `drivers/dshot_dpwm.h:32-34` |
| `DSHOT_DMA_BUFFER_SIZE` | 18 | `drivers/dshot_dpwm.h:58` |
| dshot idle | 550 (5.5%) | `pg/motor.c:63` |
| `SPI_CLOCK_INITIALIZATION / FAST` | 256 / 4 (APB2 기준) | `drivers/bus_spi.h:53-57` |
| `NVIC_PRIO_MPU_INT_EXTI` | (15,15) → 0xF0 | `drivers/nvic.h:31` |
| `NVIC_PRIO_DSHOT_DMA` | (2,1) | `drivers/nvic.h:29` |
| `NVIC_PRIORITY_GROUPING` | Group 2 | `drivers/nvic.h:71,77` |
| gyro 캘리브레이션 | 1.25s, 이동 임계 48 | `sensors/gyro.c:108-109` |
| dyn LPF gyro / dterm | 200~500Hz / 70~170Hz | `sensors/gyro.c:128-129`, `flight/pid.c:202-203` |
| gyro LPF2 / dterm LPF2 | PT1 250Hz / PT1 150Hz | `sensors/gyro.c:116-117`, `flight/pid.c:199-201` |
| dyn notch | 150~600Hz, width 8%, Q 120 | `sensors/gyro.c:130-133` |

## 부록 B. 파일 색인

| 주제 | 파일 |
|------|------|
| 스타트업 | `startup/startup_stm32f40xx.s`, `startup/system_stm32f4xx.c`, `drivers/system_stm32f4xx.c`, `drivers/system.c` |
| 초기화 | `main.c`, `fc/init.c` |
| 스케줄러 | `scheduler/scheduler.c/.h`, `fc/tasks.c`, `fc/core.c` |
| 자이로 | `sensors/gyro.c`, `sensors/gyro_init.c`, `sensors/gyro_filter_impl.c`, `drivers/accgyro/accgyro_mpu.c`, `drivers/accgyro/accgyro_spi_mpu6000.c`, `drivers/accgyro/gyro_sync.c` |
| SPI | `drivers/bus_spi.c/.h`, `drivers/bus_spi_stdperiph.c`, `drivers/bus_spi_pinconfig.c` |
| EXTI / NVIC | `drivers/exti.c`, `drivers/nvic.h` |
| 필터 | `common/filter.c`, `flight/gyroanalyse.c` |
| PID / 믹서 | `flight/pid.c/.h`, `flight/mixer.c`, `fc/rc.c` |
| 모터 | `drivers/motor.c`, `drivers/dshot.c`, `drivers/dshot_dpwm.c/.h`, `drivers/pwm_output_dshot.c`, `drivers/pwm_output_dshot_shared.c`, `drivers/timer_def.h`, `drivers/timer_stm32f4xx.c` |
| 링커 | `src/link/stm32_flash_f405.ld`, `src/link/stm32_flash_split.ld` |
| 타깃 | `target/VGOODRCF4/target.h`, `target.c`, `target.mk` |
