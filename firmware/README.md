# firmware/ — 자체 FC 펌웨어 (STM32F405, bare-metal)

HAL과 SPL 없이, CMSIS 헤더(레지스터 정의)만 쓰고 레지스터를 직접 다룬다.

이 문서는 **뼈대를 직접 작성하는 순서**다. 단계마다 "무엇을 쓰는지 / 무엇을 보고 쓰는지 / 어떻게 확인하는지"를 적었다. 한 단계가 끝날 때마다 빌드가 되는지 확인하고 다음으로 넘어간다.

## 원칙

- HAL, SPL, CubeMX 생성 코드는 쓰지 않는다. 외부 코드는 `lib/cmsis/` 헤더까지만.
- 레지스터는 숫자 대신 CMSIS 이름을 쓴다. `RCC->CR |= RCC_CR_HSEON;` (`0x10000` 아님)
- 값의 근거가 되는 매뉴얼 장이나 Betaflight 코드 위치를 주석에 남긴다. 나중에 논문에 쓸 때 필요하다.

---

## 0. 준비

### 툴체인 (이미 설치돼 있음, 2026-09 확인)

STM32CubeCLT 1.22.0이 설치돼 있고 PATH에 잡혀 있다. 추가 설치는 필요 없다.

| 도구 | 경로 | 버전 |
|------|------|------|
| arm-none-eabi-gcc | `/opt/ST/STM32CubeCLT_1.22.0/GNU-tools-for-STM32/bin/` | 14.3.1 |
| make | `/opt/ST/STM32CubeCLT_1.22.0/Make/bin/make` 또는 `/usr/bin/make` | |
| STM32_Programmer_CLI | `/opt/ST/STM32CubeCLT_1.22.0/STM32CubeProgrammer/bin/` | 굽기용. 아직 쓰지 않음 |
| F405 SVD | `/opt/ST/STM32CubeCLT_1.22.0/STMicroelectronics_CMSIS_SVD/STM32F405.svd` | 디버거용 |

```bash
arm-none-eabi-gcc --version
```

다른 맥에 새로 설치한다면: `brew install --cask gcc-arm-embedded` (Arm 공식 배포판). 주의 - cask가 아닌 formula `arm-none-eabi-gcc`에는 newlib이 없어서 `--specs=nano.specs` 링크가 실패한다.

논문 재현성을 위해 컴파일러 버전을 기록해 둔다. 대조군 Betaflight 4.2.0은 GCC 9.2.1로 빌드됐다 (`betaflight/make/tools.mk:17-19`).

### CMSIS 헤더 (`lib/cmsis/`, 이미 복사돼 있음)

`~/STM32Cube/Repository/STM32Cube_FW_F4_V1.28.3/Drivers/CMSIS/`에서 필요한 파일만 가져온 것이다. 직접 작성할 대상이 아니므로 지우지 않는다.

- `core/` : CMSIS-Core(M) 5.6 — `core_cm4.h`, `cmsis_gcc.h` 등
- `device/` : ST STM32F4xx device 2.6.11 — `stm32f4xx.h`, `stm32f405xx.h`, `system_stm32f4xx.h`
- 라이선스: Apache-2.0 (`LICENSE-*.txt`)

`stm32f4xx.h`는 `USE_HAL_DRIVER`를 정의하지 않으면 HAL을 포함하지 않는다. 그 매크로를 정의하지 않는 것이 HAL을 배제하는 방법이다.

### 참고할 파일 (이 맥에 있음)

```
~/STM32Cube/Repository/STM32Cube_FW_F4_V1.28.3/
├── Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/gcc/startup_stm32f405xx.s   ← 스타트업 원본(어셈블리)
└── Projects/*/Templates/STM32CubeIDE/*_FLASH.ld                                   ← 링커 스크립트 형식
```

F405 전용 `.ld`는 이 패키지에 없다. 아무 보드 것이나 열어 구조를 보고, 메모리 크기와 주소만 F405에 맞게 바꾼다.

대조군 코드도 참고가 된다. 다만 조건부 컴파일이 많아 읽기 번거로우므로, ST 템플릿으로 구조를 먼저 잡은 뒤 "실제 FC에서는 이렇게 하는구나"를 확인하는 용도로 본다.

```
betaflight/src/main/startup/startup_stm32f40xx.s      스타트업
betaflight/src/main/startup/system_stm32f4xx.c:690    SetSysClock()
betaflight/src/link/stm32_flash_f405.ld               링커 스크립트
```

### 문서 (레지스터 값의 최종 근거)

| 문서 | 내용 |
|------|------|
| **RM0090** | F405/407 레퍼런스 매뉴얼. RCC, FLASH, GPIO, SPI, TIM, DMA. **F405의 RCC는 7장이다. 6장은 F42x/43x용이니 주의** |
| **DS8626** | F405 데이터시트. 핀 배치, 전기적 특성 |
| **PM0214** | Cortex-M4 프로그래밍 매뉴얼. NVIC, SysTick, FPU, DWT, 예외 |
| **ES0182** | F405/407 에라타 |

### 보드 정보

핀 배치는 `baseline/VGRC-VGOODRCF4.config`가 기준이다 (보드에 실제로 적용된 설정). 자세한 분석은 [docs/betaflight-analysis.md](../docs/betaflight-analysis.md).

| 항목 | 값 |
|------|-----|
| MCU | STM32F405RG, HSE 8MHz (`set system_hse_mhz = 8`) |
| LED | PC14 (LED 1), PC15 (LED 2), active-low |
| 비퍼 | PC13, inverted |
| 자이로 | MPU6000, SPI3 (SCK PB3 / MISO PB4 / MOSI PB5, AF6), CS PA15, INT PC3 |
| 자이로 방향 | `gyro_1_sensor_align = CW270` |
| 모터 1~4 | PA8 (TIM1_CH1), PA9 (TIM1_CH2), PA10 (TIM1_CH3), PC8 (TIM8_CH3) |
| 모터 프로토콜 | DSHOT600 |
| 수신기 | CRSF, UART5 (TX PC12 / RX PD2) |

핀 관련 함정:
- **PA15, PB3, PB4**는 리셋 후 JTAG 기능(AF0)이다. SPI3로 쓰려면 MODER/AFR을 다시 설정해야 한다.
- **PC13~15**는 백업 도메인 핀이다. 출력 속도 2MHz 이하, 싱크 3mA 이하. PC14는 LSE 핀이기도 해서 LSE가 켜져 있으면 GPIO로 못 쓴다.
- **PA13, PA14**(SWD)는 이 보드에서 PINIO(VTX 전원, 카메라 전환)로 쓰인다. SWD 디버깅은 기대하기 어렵다.

---

## 1단계 — 링커 스크립트 (`ld/stm32f405rg.ld`)

코드와 변수를 메모리 어디에 놓을지 정한다.

**쓸 내용**

- `MEMORY`: FLASH 0x08000000 1024K, RAM 0x20000000 128K, CCM 0x10000000 64K
- `_estack` = RAM 끝 주소
- 섹션: `.isr_vector`(FLASH 맨 앞) → `.text` → `.rodata` → `.data`(RAM에 놓되 초기값은 FLASH에) → `.bss`(RAM, NOLOAD)
- `.data`용 심볼 `_sidata`(FLASH의 초기값 위치) `_sdata` `_edata`, `.bss`용 `_sbss` `_ebss`

**이해하고 넘어갈 것**

- `>RAM AT >FLASH`의 의미. 실행 주소(VMA)와 저장 주소(LMA)가 왜 다른가
- `.bss`는 왜 파일에 저장되지 않는가
- CCM을 스택이나 일반 변수에 쓸 수 있지만 **DMA 버퍼로는 쓸 수 없는** 이유 (CCM은 D-bus 전용이라 DMA가 접근하지 못한다. 나중에 DShot 출력에서 바로 문제가 된다)

**확인**: 이 단계만으로는 빌드가 안 된다. 3단계 후에 `build/fc.map`에서 주소를 확인한다.

---

## 2단계 — 스타트업 (`src/startup_stm32f405.c`)

리셋 직후부터 `main()`까지.

**쓸 내용**

1. 벡터 테이블: `[0]`= 초기 스택 포인터(`_estack`), `[1]`= `Reset_Handler`, 이후 예외/인터럽트 핸들러
2. `Reset_Handler`: FPU 활성화 → `.data` 복사 → `.bss` 0으로 채움 → `main()` 호출
3. `Default_Handler`: 무한 루프. 모든 핸들러를 weak alias로 여기에 연결

**권하는 방법**

IRQ 이름을 82개 손으로 옮겨 적지 말고, `lib/cmsis/device/stm32f405xx.h`의 `IRQn_Type` 나열을 보고 배열 인덱스를 `[IRQ번호 + 16]`으로 지정한다. 이렇게 하면 순서가 어긋날 수 없고, F405에 없는 인터럽트 자리는 자동으로 0이 된다.

```c
#define IRQ(n) ((n) + 16)
const vector_t g_vectors[] = {
    [0] = (vector_t)&_estack,
    [1] = Reset_Handler,
    [IRQ(SysTick_IRQn)] = SysTick_Handler,
    [IRQ(EXTI3_IRQn)]   = EXTI3_IRQHandler,
    ...
};
```

**함정**

- **FPU를 가장 먼저 켠다.** `-mfloat-abi=hard`로 빌드하면 컴파일러가 아무 데서나 VFP 명령을 쓸 수 있다. FPU가 꺼진 상태에서 실행하면 UsageFault다.
- **복사 루프가 `memcpy` 호출로 바뀌지 않게 한다.** GCC가 루프를 라이브러리 호출로 치환할 수 있다. `Reset_Handler`에 `__attribute__((optimize("no-tree-loop-distribute-patterns")))`를 붙인다.
- `-nostartfiles`로 링크하면 `__libc_init_array`를 쓸 수 없다(`_init` 없음). C++ 전역 생성자가 필요하면 `.init_array`를 직접 순회한다.

**확인**: 빌드 후 아래로 검증한다. 벡터 테이블 크기는 98 × 4 = **392바이트(0x188)**여야 한다.

```bash
arm-none-eabi-size -A build/fc.elf | grep isr_vector
xxd -e -l 16 build/fc.bin
```

첫 워드가 `20020000`(스택 포인터), 둘째 워드가 `Reset_Handler` 주소 + 1이면 정상이다. **+1(Thumb 비트)이 없으면 리셋 직후 HardFault가 난다.**

---

## 3단계 — Makefile

**필수 플래그**

| 항목 | 값 | 이유 |
|------|-----|------|
| 아키텍처 | `-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard` | Cortex-M4F. 단정밀도 FPU |
| 칩 선택 | `-DSTM32F405xx` | `stm32f4xx.h`가 어느 칩 헤더를 포함할지 결정 |
| 인클루드 | `-Isrc -Ilib/cmsis/core -Ilib/cmsis/device` | |
| 경고 | `-Wall -Wextra -Wdouble-promotion` | `-Wdouble-promotion`은 실수로 double 연산이 들어가는 것을 잡아 준다. F405의 FPU는 단정밀도만 지원해서 double은 소프트웨어로 처리되어 매우 느리다 |
| 섹션 분리 | `-ffunction-sections -fdata-sections` + `-Wl,--gc-sections` | 안 쓰는 코드 제거 |
| 링크 | `-T ld/stm32f405rg.ld -nostartfiles --specs=nano.specs --specs=nosys.specs` | |
| 맵 파일 | `-Wl,-Map=build/fc.map,--cref -Wl,--print-memory-usage` | 주소 확인용 |

hex/bin 생성은 `arm-none-eabi-objcopy -O ihex` / `-O binary`.

**굽기 타깃(`flash`)은 만들지 않는다.** 굽는 작업은 실행 전에 확인을 받기로 했다(CLAUDE.md).

**확인**: `make`가 통과하고 `--print-memory-usage`가 FLASH/RAM 사용량을 출력하면 된다.

---

## 4단계 — 클럭 (`src/clock.c`)

8MHz HSE → PLL → 168MHz. **여기가 이 뼈대에서 가장 중요한 부분이다.**

**PLL 계산** (RM0090 7장 RCC_PLLCFGR)

```
f_VCO_in  = f_HSE / PLLM   : 1~2MHz (2MHz 권장)   8 / 4 = 2MHz
f_VCO_out = f_VCO_in * PLLN : 100~432MHz          2 * 168 = 336MHz
f_SYSCLK  = f_VCO_out / PLLP : ≤168MHz            336 / 2 = 168MHz
f_USB     = f_VCO_out / PLLQ : 정확히 48MHz       336 / 7 = 48MHz
```

→ PLLM=4, PLLN=168, PLLP=2, PLLQ=7. Betaflight도 같은 값이다.

**순서 (지켜야 한다)**

1. SYSCLK를 HSI로 두고 PLL을 끈다 (PLLCFGR은 PLL이 꺼져 있을 때만 쓸 수 있다)
2. HSE 켜고 HSERDY 대기 (타임아웃 필요 — 크리스탈이 죽으면 무한 대기에 빠진다)
3. PWR 클럭 켜고 `PWR->CR |= PWR_CR_VOS` (레귤레이터 스케일 1)
4. **클럭을 올리기 전에** `FLASH->ACR`에 5 wait state + 프리페치 + I/D 캐시 설정. 쓴 뒤 다시 읽어 확인
5. AHB /1, APB1 /4, APB2 /2
6. PLLCFGR 설정 → PLLON → PLLRDY 대기
7. SW=PLL로 전환하고 SWS가 PLL이 될 때까지 대기

**이해하고 넘어갈 것**

- 4번을 3번보다 먼저 하면 왜 안 되는지, 반대로 순서를 어기면 어떻게 되는지
- APB1은 42MHz, APB2는 84MHz인데 **타이머 클럭은 각각 84MHz, 168MHz**가 되는 이유 (APB 분주가 1이 아니면 타이머 클럭은 2배). DShot 프리스케일러 계산에서 바로 쓰인다
- SPI3는 APB1(42MHz), SPI1은 APB2(84MHz)에 붙어 있다는 점

**권하는 것**: HSE 기동에 실패하면 HSI로 대체하고, 그 사실을 반환값으로 알려 주도록 만든다. 첫 보드 작업에서 "크리스탈 문제인가 내 코드 문제인가"를 구분할 수 있다.

---

## 5단계 — SysTick + LED (`src/main.c`)

**쓸 내용**

- SysTick: `LOAD = 168000 - 1`, CLKSOURCE=프로세서 클럭, TICKINT, ENABLE. 핸들러에서 ms 카운터 증가
- `delay_ms()`: 카운터 차이를 비교. 뺄셈으로 비교해야 오버플로에 안전하다
- LED(PC14): GPIOC 클럭 → MODER 출력, OTYPER PP, OSPEEDR low, 출력은 BSRR로
- active-low이므로 켤 때 LOW

**확인 (보드에 구운 뒤)**

- 1Hz로 깜빡이면 168MHz가 정상. 스톱워치로 10회에 10초인지 본다
- 클럭이 안 잡혀 HSI 16MHz로 돌고 있으면 10.5배 느려진다 — 눈으로 구분된다
- 전혀 깜빡이지 않으면 HardFault 등으로 `Default_Handler`에 빠진 것

---

## 검증에 쓰는 명령

```bash
make                                    # 빌드
arm-none-eabi-size -A build/fc.elf      # 섹션별 크기
arm-none-eabi-nm -n build/fc.elf        # 심볼이 놓인 주소 (주소순)
arm-none-eabi-objdump -d -S build/fc.elf > build/fc.lst   # 역어셈블
xxd -e -l 64 build/fc.bin               # 벡터 테이블 앞부분
```

`build/fc.map`에서 각 섹션의 시작 주소와 크기, 어떤 오브젝트가 무엇을 가져왔는지 확인할 수 있다.

---

## 망가뜨려 보기 (이해를 확인하는 방법)

뼈대가 동작한 뒤, 아래를 하나씩 해 보면 각 부분이 무슨 일을 하는지 확실해진다. 되돌릴 수 있는 것들이다.

1. `Reset_Handler`에서 FPU 활성화를 빼고 float 연산을 넣는다 → UsageFault
2. 플래시 wait state를 0으로 두고 168MHz로 올린다 → 동작 이상
3. `.bss` 0 채우기를 뺀다 → 초기화 안 한 전역 변수가 쓰레기 값을 갖는다
4. 벡터 테이블 `[1]`에서 Thumb 비트를 없앤다 → 리셋 직후 HardFault
5. `_estack`을 CCM 끝(0x10010000)으로 옮긴다 → 동작한다 (Betaflight가 실제로 쓰는 방식)
6. SysTick 대신 TIM2로 LED를 깜빡인다 → 타이머는 제어 루프에서 쓰게 된다

---

## 다음 단계

뼈대가 보드에서 확인되면 CLAUDE.md의 순서를 따른다: **센서 읽기 + 로그** → 자세 추정 → 모터 출력(프로펠러 제거) → 축별 제어 루프(기체 고정) → 호버링.

바로 다음은 SPI3 + MPU6000이다. 레지스터 설정 순서와 값은 [docs/betaflight-analysis.md](../docs/betaflight-analysis.md) 4.2~4.3절에 정리돼 있다. 그 전에 USB 시리얼이나 UART로 값을 출력할 수단이 필요하다(이 보드는 블랙박스 저장장치가 없다).

---

## 보드에 굽기 (아직 하지 않음)

굽는 명령은 실행 전에 확인을 받는다 (CLAUDE.md).

굽기 전에 `baseline/`에 있어야 하는 것:

- [x] `diff_all.txt` — 설정 백업
- [x] `VGRC-VGOODRCF4.config` — 보드 핀/타이머/DMA 설정
- [ ] `betaflight_4.2.0_STM32F405.hex` — 복원용 펌웨어 (이 보드는 유니파이드 STM32F405 빌드를 쓴다. VGOODRCF4 전용 hex는 없다)
- [ ] `dump_hardware.txt` — 보드에 실제 적용된 resource/timer/dma
- [ ] `status.txt`, `tasks.txt` — 성능 비교 기준값

자체 펌웨어는 0x08000000부터 쓰므로 Betaflight와 설정 섹터(0x08004000)를 모두 덮어쓴다. 되돌리려면 hex를 다시 굽고 `diff all`을 붙여 넣어야 한다.

SWD 핀이 PINIO로 쓰이므로 굽기는 USB DFU(부트 버튼을 누른 채 USB 연결)로 하게 될 가능성이 높다. ROM 부트로더는 지워지지 않으므로, 잘못 구워도 DFU로 다시 들어갈 수 있다.
