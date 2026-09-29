# Module 1 Report

## 문제 1. 목표 입력과 응답 확인

### 1. 수행 목적

본 문제의 목적은 Raspberry Pi의 Ubuntu Server 환경에서 SSH로 원격 접속하여 OpenCR 펌웨어 업로드, DYNAMIXEL 제어 명령 송신, 측정값 수신 및 로그 저장을 모두 Raspberry Pi에서 수행하고, 한 개의 작은 상대 목표각에 대한 실제 응답을 확인하는 것이다.

증거 파일:

```text
lv2_module1/
├── report.md
└── results/
    ├── environment.txt
    ├── upload.log
    └── run_A.log
```

---

### 2. 수행 환경

| 항목 | 값 |
|---|---|
| Hostname | `pa44` |
| OS | Ubuntu 22.04.5 LTS |
| Architecture | `aarch64` |
| User | `pa44` |
| Serial access group | `dialout` |
| Controller | OpenCR 1.0 |
| Motor | DYNAMIXEL XM430-W350 |
| Motor ID | 12 |
| Baud rate | 1 Mbps |
| Protocol | DYNAMIXEL Protocol 2.0 |
| Example | `opencr_position_p` |

`environment.txt`에서 다음을 확인하였다.

```text
=== Hostname ===
pa44

=== OS ===
PRETTY_NAME="Ubuntu 22.04.5 LTS"

=== Architecture ===
aarch64

=== User / groups ===
pa44
pa44 adm dialout cdrom floppy sudo audio dip video plugdev netdev lxd
```

OpenCR은 Raspberry Pi에 USB로 직접 연결하였다.

```text
/dev/serial/by-id/usb-ROBOTIS_OpenCR_Virtual_ComPort_in_FS_Mode_FFFFFFFEFFFF-if00
```

해당 persistent path는 현재 `/dev/ttyACM0`을 가리켰고, 장치 정보는 다음과 같았다.

```text
DEVNAME=/dev/ttyACM0
ID_VENDOR=ROBOTIS
ID_MODEL=OpenCR_Virtual_ComPort_in_FS_Mode
```

따라서 Raspberry Pi가 OpenCR을 정상적으로 인식하고 있으며, `dialout` 권한으로 시리얼 포트 접근이 가능함을 확인하였다.

증거:

```text
results/environment.txt
```

---

### 3. OpenCR 펌웨어 업로드

Raspberry Pi에서 다음 펌웨어를 OpenCR에 업로드하였다.

```text
/home/pa44/pa-opencr-build/output/opencr_position_p.ino.bin
```

업로드 로그의 핵심 부분은 다음과 같다.

```text
opencr_ld ver 1.0.4
file name : /home/pa44/pa-opencr-build/output/opencr_position_p.ino.bin
file size : 102 KB
Open port OK

Board Name : OpenCR R1.0
Board Ver  : 0x17020800
Board Rev  : 0x00000000

flash_erase : 0 : 1.012000 sec
flash_write : 0 : 1.186000 sec
CRC OK 9B55F6 9B55F6 0.004000 sec
[OK] Download
jump_to_fw
jump finished
```

이 로그를 통해 OpenCR 인식, flash erase, flash write, CRC 검증, 다운로드 성공, 펌웨어 실행을 모두 확인하였다.

증거:

```text
results/upload.log
```

---

### 4. 시리얼 연결 및 실행 준비

OpenCR 시리얼 모니터는 Raspberry Pi에서 다음과 같이 연결하였다.

```bash
python3 -m serial.tools.miniterm \
  "$PORT" 115200 --eol LF -e
```

연결 설정:

```text
115200 baud, 8 data bits, no parity, 1 stop bit
```

실행 전 `?` 입력에 대해 다음 메시지가 출력되었다.

```text
Invalid setting. Finite Kp, positive speed or max, angle -90..90.
Use k/v/a or s <Kp> <speed_deg_s|max> <angle_deg>.
```

이는 펌웨어가 `FAULT` 상태가 아니라 정상 command parser 상태에 있음을 보여 준다.

---

### 5. 실행 A 설정

실행 명령:

```text
s 1 10 30
```

펌웨어 확인 메시지:

```text
START: current position = 0 deg.

P SET Kp=1.0000, Ki=0.0000, Kd=0.0000,
speed_limit_deg_s=10.000, angle_deg=30.000

Run timeout [s]: 60.0
```

실행 조건:

| 항목 | 값 | 단위 |
|---|---:|---|
| `Kp` | 1.0 | 1/s |
| `Ki` | 0.0 | - |
| `Kd` | 0.0 | - |
| Relative target angle | +30 | deg |
| Velocity limit | 10 | deg/s |
| Run timeout | 60 | s |

이번 실행은 `Ki = 0`, `Kd = 0`인 순수 P 제어이다.

---

### 6. 로그 변수와 단위

| 변수 | 의미 | 단위 |
|---|---|---|
| `target_deg` | 목표 상대 위치 | deg |
| `position_deg` | 측정된 상대 위치 | deg |
| `error_deg` | `target_deg - position_deg` | deg |
| `p_deg_s` | P 제어기 계산 출력 | deg/s |
| `i_deg_s` | I 항 출력 | deg/s |
| `d_deg_s` | D 항 출력 | deg/s |
| `pid_deg_s` | P+I+D 전체 출력 | deg/s |
| `speed_deg_s` | DYNAMIXEL에서 측정된 실제 속도 | deg/s |
| `u_deg_s` | 실제 DYNAMIXEL에 보낸 속도 명령 | deg/s |
| `v_limit_deg_s` | 설정 속도 상한 | deg/s |
| `dt_ms` | 제어 루프 간격 | ms |
| `kp` | 비례 게인 | 1/s |
| `t_s` | 실행 시간 | s |

핵심 구분:

```text
target_deg   = 목표값
position_deg = 측정값
u_deg_s      = 실제 actuator command
speed_deg_s  = 측정된 모터 속도
p_deg_s      = P 제어기가 계산한 출력
```

---

### 7. 목표 변경 전

`t = 0.1 s`부터 `1.9 s`까지 목표가 0 deg였으므로 오차와 출력이 모두 0에 가까웠다.

예:

```text
target_deg:0.000
position_deg:0.000
error_deg:0.000
p_deg_s:0.000
speed_deg_s:0.000
u_deg_s:0.000
```

---

### 8. 목표 변경 직후

`t = 2.000 s`에서 목표가 `0 deg`에서 `+30 deg`로 변경되었다.

```text
target_deg:30.000
position_deg:0.000
error_deg:30.000
p_deg_s:30.000
speed_deg_s:0.000
u_deg_s:9.618
v_limit_deg_s:10.000
dt_ms:10.000
kp:1.0000
t_s:2.000
```

오차는

\[
e = 30 - 0 = +30^\circ$
\]

이고 `Kp = 1.0 1/s`이므로 P 제어기 계산 출력은

\[
u_P = K_p e = 30^\circ/s$
\]

이다.

그러나 속도 상한은 `10 deg/s`이므로 실제 DYNAMIXEL에 전달된 명령은 약 `9.618 deg/s`로 제한되었다.

즉,

```text
P controller output = 30.000 deg/s
actual actuator command = 9.618 deg/s
```

이다.

---

### 9. 목표 변경 이후 실행 기록

| Time [s] | Target [deg] | Position [deg] | Error [deg] | P output [deg/s] | `u` [deg/s] | Measured speed [deg/s] |
|---:|---:|---:|---:|---:|---:|---:|
| 2.0 | 30.000 | 0.000 | 30.000 | 30.000 | 9.618 | 0.000 |
| 2.1 | 30.000 | 0.176 | 29.824 | 29.824 | 9.618 | 1.374 |
| 2.2 | 30.000 | 1.055 | 28.945 | 28.945 | 9.618 | 8.244 |
| 2.3 | 30.000 | 2.109 | 27.891 | 27.891 | 9.618 | 9.618 |
| 2.4 | 30.000 | 3.076 | 26.924 | 26.924 | 9.618 | 9.618 |
| 2.5 | 30.000 | 4.043 | 25.957 | 25.957 | 9.618 | 8.244 |
| 2.6 | 30.000 | 5.010 | 24.990 | 24.990 | 9.618 | 9.618 |
| 2.7 | 30.000 | 5.977 | 24.023 | 24.023 | 9.618 | 9.618 |
| 2.8 | 30.000 | 6.768 | 23.232 | 23.232 | 9.618 | 8.244 |
| 2.9 | 30.000 | 7.734 | 22.266 | 22.266 | 9.618 | 8.244 |
| 3.0 | 30.000 | 8.701 | 21.299 | 21.299 | 9.618 | 9.618 |
| 3.1 | 30.000 | 9.668 | 20.332 | 20.332 | 9.618 | 8.244 |
| 3.2 | 30.000 | 10.635 | 19.365 | 19.365 | 9.618 | 9.618 |
| 3.3 | 30.000 | 11.689 | 18.311 | 18.311 | 9.618 | 9.618 |
| 3.4 | 30.000 | 12.656 | 17.344 | 17.344 | 9.618 | 8.244 |
| 3.5 | 30.000 | 13.623 | 16.377 | 16.377 | 9.618 | 9.618 |
| 3.6 | 30.000 | 14.590 | 15.410 | 15.410 | 9.618 | 8.244 |
| 3.7 | 30.000 | 15.557 | 14.443 | 14.443 | 9.618 | 8.244 |

목표 변경 이후 5행 이상의 측정 기록을 충분히 확보하였다.

---

### 10. 실제 움직임 설명

목표 입력 이후 측정 위치는

```text
0.000 → 0.176 → 1.055 → 2.109 → ... → 15.557 deg
```

로 증가하였다.

즉, `+30 deg` 목표와 양의 위치 오차에 의해 모터가 양의 방향으로 실제 이동하였다.

`t = 3.7 s`에서도

```text
error_deg = 14.443 deg
p_deg_s = 14.443 deg/s
u_deg_s = 9.618 deg/s
```

이므로 P 제어기 계산 출력은 여전히 속도 상한보다 컸고 실제 actuator command는 계속 제한된 상태였다.

---

### 11. 정지 확인

사용자가 실행 중 다음 명령을 입력하였다.

```text
x
```

펌웨어는 다음과 같이 응답하였다.

```text
STOP: user
```

따라서 SSH 종료에 의존하지 않고 펌웨어의 정지 명령으로 실행을 명시적으로 종료하였다.

이번 Run A는 `t = 3.7 s`, `position = 15.557 deg` 부근에서 수동 정지했으므로, 이 실행에서 모터가 `30 deg`에 도달했다고 해석하지 않는다.

---

### 12. 문제 1 요약

상대 목표각 `+30 deg`를 입력하자 위치 오차가 양수가 되었고 P 제어기는 양의 속도 명령을 생성하였다. 측정 위치는 `0 deg`에서 `15.557 deg`까지 목표 방향으로 증가하였으며, 실험은 `x` 명령으로 수동 정지하여 `STOP: user`를 확인하였다.

---

## 문제 2. 오차와 피드백 해석

### 1. 위치 오차 정의

\[
e = 	heta_{target} - 	heta_{current}
\]

- \(	heta_{target}\): 목표 상대 위치
- \(	heta_{current}\): 측정된 현재 상대 위치
- \(e\): 위치 오차

모두 `deg` 단위를 사용한다.

---

### 2. 세 시점의 오차 계산

#### 시점 A: 목표 변경 직후

`t = 2.000 s`

```text
target = 30.000 deg
current = 0.000 deg
```

\[
e = 30.000 - 0.000 = +30.000^\circ
\]

#### 시점 B: 이동 중간

`t = 2.800 s`

```text
target = 30.000 deg
current = 6.768 deg
```

\[
e = 30.000 - 6.768 = +23.232^\circ
\]

#### 시점 C: 수동 정지 직전

`t = 3.700 s`

```text
target = 30.000 deg
current = 15.557 deg
```

\[
e = 30.000 - 15.557 = +14.443^\circ
\]

| 시점 | Time [s] | Target [deg] | Current [deg] | Error [deg] |
|---|---:|---:|---:|---:|
| A | 2.0 | 30.000 | 0.000 | +30.000 |
| B | 2.8 | 30.000 | 6.768 | +23.232 |
| C | 3.7 | 30.000 | 15.557 | +14.443 |

오차는

```text
+30.000 → +23.232 → +14.443 deg
```

로 감소하였다.

---

### 3. P 제어와 오차의 관계

P 제어 관계는

\[
u_P = K_p e
\]

이다.

이번 실행에서는

\[
K_p = 1.0\;s^{-1}
\]

이므로 수치적으로 `error_deg`와 `p_deg_s`가 동일하게 나타난다.

예:

```text
t = 2.0 s
error_deg = 30.000 deg
p_deg_s = 30.000 deg/s

t = 2.8 s
error_deg = 23.232 deg
p_deg_s = 23.232 deg/s

t = 3.7 s
error_deg = 14.443 deg
p_deg_s = 14.443 deg/s
```

단, 단위는 서로 다르다.

```text
error_deg : deg
p_deg_s   : deg/s
```

---

### 4. `p_deg_s`와 `u_deg_s`

`p_deg_s`는 P 제어기가 계산한 요구 속도이고, `u_deg_s`는 속도 제한 및 actuator command 변환 이후 실제 DYNAMIXEL에 전달한 속도 명령이다.

목표 변경 직후:

```text
p_deg_s = 30.000 deg/s
v_limit_deg_s = 10.000 deg/s
u_deg_s = 9.618 deg/s
```

따라서 제어 흐름은 다음과 같이 볼 수 있다.

```text
position error
      ↓
     Kp
      ↓
 p_deg_s
      ↓
velocity limit / actuator conversion
      ↓
 u_deg_s
      ↓
DYNAMIXEL
```

---

### 5. `u_deg_s`와 `speed_deg_s`

```text
u_deg_s
= OpenCR이 DYNAMIXEL에 보낸 속도 명령

speed_deg_s
= DYNAMIXEL에서 측정되어 OpenCR로 돌아온 실제 속도
```

목표 변경 직후:

```text
t = 2.0 s
u_deg_s = 9.618 deg/s
speed_deg_s = 0.000 deg/s
```

`t = 2.2 s`:

```text
u_deg_s = 9.618 deg/s
speed_deg_s = 8.244 deg/s
```

`t = 2.3 s`:

```text
u_deg_s = 9.618 deg/s
speed_deg_s = 9.618 deg/s
```

따라서 `u_deg_s`는 actuator input이고, `speed_deg_s`는 plant의 측정 응답이다.

---

### 6. 위치 피드백 경로

XM430-W350은 내부 위치 측정값을 DYNAMIXEL control table의 `Present Position` 값으로 제공한다. OpenCR은 DYNAMIXEL 통신을 통해 이 값을 읽고 제어기의 피드백으로 사용한다.

```text
XM430-W350 internal position measurement
                ↓
         Present Position
                ↓
      DYNAMIXEL control table
                ↓
      DYNAMIXEL Protocol 2.0
                ↓
         OpenCR DXL interface
                ↓
        Dynamixel2Arduino
                ↓
         measured position
                ↓
target position - measured position
                ↓
              error
                ↓
               Kp
                ↓
        velocity command
                ↓
           DYNAMIXEL
                ↓
         physical motion
                └──────────── feedback
```

즉, OpenCR은 명령만 보내는 것이 아니라 DYNAMIXEL이 측정한 실제 위치를 반복적으로 읽어 목표값과 비교한다.

---

### 7. 폐루프 응답 해석

목표가 `+30 deg`로 바뀐 직후:

```text
target = 30 deg
position = 0 deg
error = +30 deg
```

이므로 큰 양의 오차가 발생하고, 양의 P 출력과 양의 속도 명령이 발생한다.

이후 위치가 증가하면서 오차는 감소한다.

```text
position ↑
error ↓
P output ↓
```

이번 Run A에서는 오차가 `14.443 deg` 남은 상태에서 사용자가 `x` 명령으로 수동 정지했으므로 완전 수렴 구간은 기록하지 않았다.

---

### 8. Target = +30 deg, Current = +35 deg

가정:

```text
target = +30 deg
current = +35 deg
```

오차는

\[
e = 30 - 35 = -5^\circ
\]

이다.

`Kp > 0`이므로

\[
u_P = K_p e
\]

에서 제어 출력도 음수가 된다.

```text
negative error
      ↓
negative velocity command
      ↓
motor moves in the negative direction
      ↓
position decreases toward +30 deg
```

따라서 모터는 `+35 deg`에서 `+30 deg` 방향으로 되돌아가도록 명령된다.

---

### 9. 문제 2 요약

실행 A에서 측정 위치가 `0.000 deg → 6.768 deg → 15.557 deg`로 증가함에 따라 오차는 `+30.000 deg → +23.232 deg → +14.443 deg`로 감소하였다. P 제어에서는 오차의 부호가 속도 명령의 방향을 결정하므로 현재 위치가 목표보다 작으면 양의 방향으로, 현재 위치가 목표보다 크면 음의 방향으로 보정한다.

---

## 제출 증거 요약

### 환경 확인

```text
results/environment.txt
```

확인 내용:

- Raspberry Pi hostname
- Ubuntu 22.04.5 LTS
- `aarch64`
- `dialout`
- OpenCR serial path
- ROBOTIS/OpenCR 장치 정보

### 업로드 확인

```text
results/upload.log
```

핵심 출력:

```text
Board Name : OpenCR R1.0
flash_erase : 0
flash_write : 0
CRC OK
[OK] Download
jump finished
```

### 실행 A

```text
results/run_A.log
```

핵심 출력:

```text
s 1 10 30
START: current position = 0 deg.
P SET Kp=1.0000, Ki=0.0000, Kd=0.0000,
speed_limit_deg_s=10.000, angle_deg=30.000
```

목표 변경 이후 5행 이상의 측정값이 기록되었고, 마지막에는 다음 정지 확인이 포함된다.

```text
STOP: user
```

---

## 결론

Raspberry Pi의 Ubuntu Server 22.04 ARM64 환경에서 SSH 기반 원격 수행 환경을 구성하고, Raspberry Pi에서 OpenCR 펌웨어를 업로드한 뒤 DYNAMIXEL XM430-W350에 P 제어 명령을 적용하였다. `+30 deg` 상대 목표에 대해 실제 측정 위치가 양의 방향으로 증가하고 위치 오차가 감소하는 것을 확인하였으며, 목표값, 측정값, P 제어 출력, 실제 actuator command, 측정 속도를 단위와 함께 구분하였다.

또한 동일한 Run A 기록을 이용하여 위치 오차와 피드백 구조를 해석하였다. 현재 위치가 목표보다 작을 때는 양의 오차와 양의 속도 명령이 발생하고, 현재 위치가 목표보다 큰 경우에는 음의 오차와 음의 속도 명령이 발생하여 목표 방향으로 보정됨을 확인하였다.

## 문제 3. P 게인 변경에 따른 응답 비교

### 1. 비교 조건

문제 3에서는 OpenCR 위치 제어기의 `Kp`만 변경하고, 목표각과 속도 상한을 동일하게 유지하여 실행 A와 B를 비교하였다.

```text
Run A: s 1.0 10 30
Run B: s 0.5 10 30
```

| 항목 | 실행 A | 실행 B |
|---|---:|---:|
| `Kp` | 1.0 1/s | 0.5 1/s |
| `Ki` | 0.0 | 0.0 |
| `Kd` | 0.0 | 0.0 |
| Relative target | +30 deg | +30 deg |
| Velocity limit | 10 deg/s | 10 deg/s |
| Target step time | 약 2.0 s | 약 2.0 s |

실행 B는 다음 설정으로 시작되었다.

```text
P SET Kp=0.5000, Ki=0.0000, Kd=0.0000,
speed_limit_deg_s=10.000, angle_deg=30.000
```

### 2. 같은 경과 시간에서의 위치 비교

| Time [s] | A: `Kp=1.0` position [deg] | B: `Kp=0.5` position [deg] | B-A [deg] | Target exceeded? |
|---:|---:|---:|---:|---|
| 2.5 | 4.043 | 3.955 | -0.088 | Neither |
| 3.0 | 8.701 | 8.877 | +0.176 | Neither |
| 3.5 | 13.623 | 13.535 | -0.088 | Neither |
| 3.7 | 15.557 | 15.205 | -0.352 | Neither |

두 실행 모두 기록된 구간에서 목표 `+30 deg`를 초과하지 않았다. 일부 시점에서는 실행 B의 위치가 아주 약간 앞서기도 했지만 차이가 작으므로, 초기 구간의 작은 위치 차이만으로 어느 설정이 항상 더 빠르다고 단정하기는 어렵다.

### 3. 초기 응답이 비슷한 이유

P 제어기의 기본 관계는 다음과 같다.

\[
u_P = K_p e
\]

실행 A의 목표 변경 직후:

```text
error_deg = 30.000
Kp = 1.0
p_deg_s = 30.000
```

실행 B의 목표 변경 직후:

```text
error_deg = 30.000
Kp = 0.5
p_deg_s = 15.000
```

계산된 P 출력은 다르지만 둘 다 속도 상한 `10 deg/s`보다 크기 때문에 실제 DYNAMIXEL 명령은 초기에는 약

```text
u_deg_s = 9.618 deg/s
```

로 제한된다. 따라서 초기에는 `Kp`가 달라도 실제 actuator command가 비슷하여 위치 응답 차이가 작게 나타난다.

### 4. 목표에 가까워지면서 나타나는 차이

실행 B의 `t=3.4 s`:

```text
position_deg = 12.656
error_deg = 17.344
p_deg_s = 8.672
u_deg_s = 8.244
```

실행 A의 `t=3.4 s`:

```text
position_deg = 12.656
error_deg = 17.344
p_deg_s = 17.344
u_deg_s = 9.618
```

`t=3.7 s`에서도 차이가 나타난다.

#### Run A, `Kp=1.0`

```text
position_deg = 15.557
error_deg = 14.443
p_deg_s = 14.443
u_deg_s = 9.618
```

#### Run B, `Kp=0.5`

```text
position_deg = 15.205
error_deg = 14.795
p_deg_s = 7.397
u_deg_s = 6.870
```

따라서 낮은 `Kp`에서는 같은 정도의 오차에서도 더 작은 P 출력이 계산되며, 목표에 접근할수록 실제 속도 명령이 더 일찍 감소하였다.

### 5. Run B 후반부 기록

| Time [s] | Position [deg] | Error [deg] | P output [deg/s] | `u` [deg/s] |
|---:|---:|---:|---:|---:|
| 3.4 | 12.656 | 17.344 | 8.672 | 8.244 |
| 3.5 | 13.535 | 16.465 | 8.232 | 8.244 |
| 3.6 | 14.326 | 15.674 | 7.837 | 8.244 |
| 3.7 | 15.205 | 14.795 | 7.397 | 6.870 |
| 3.8 | 15.820 | 14.180 | 7.090 | 6.870 |
| 3.9 | 16.523 | 13.477 | 6.738 | 6.870 |
| 4.0 | 17.227 | 12.773 | 6.387 | 6.870 |
| 4.1 | 17.842 | 12.158 | 6.079 | 5.496 |
| 4.2 | 18.457 | 11.543 | 5.771 | 5.496 |

`Kp=0.5`에서는 오차가 줄어들면서 P 출력과 실제 속도 명령이 점차 감소하였다.

### 6. 목표 초과 여부

이번 두 실행의 기록 범위에서는 어느 실행도 목표 `+30 deg`를 초과하지 않았다.

> No target overshoot was observed during the recorded interval for either run.

이는 어떤 조건에서도 overshoot가 없다는 의미가 아니라, 이번에 기록된 구간에서 관찰되지 않았다는 의미이다.

### 7. 변경한 게인의 위치

이번에 변경한 `Kp`는 DYNAMIXEL 내부의 Position P Gain이 아니라 OpenCR 펌웨어에서 수행되는 외부 위치 제어기의 P 게인이다.

```text
target position
      ↓
position error
      ↓
OpenCR P controller (Kp)
      ↓
velocity command
      ↓
DYNAMIXEL
      ↓
measured position
      └──────── feedback
```

OpenCR이 위치 오차를 이용하여 속도 명령을 계산하고, DYNAMIXEL은 그 속도 명령에 따라 동작한다.

### 8. I항과 D항의 역할

- **I term:** 시간에 따라 누적된 오차를 이용하여 지속적인 정상상태 오차를 줄이는 역할을 한다.
- **D term:** 오차의 변화율에 반응하여 빠른 변화와 진동 또는 overshoot를 억제하는 damping 역할을 한다.

이번 비교에서는 `Ki=0`, `Kd=0`으로 유지하였으며 I/D 튜닝은 수행하지 않았다.

### 9. 문제 3 결론

`Kp`를 `1.0`에서 `0.5`로 낮추면 동일한 위치 오차에서 계산되는 P 출력이 감소한다. 그러나 초기에는 두 실행 모두 속도 상한에 걸려 실제 속도 명령이 약 `9.618 deg/s`로 제한되므로 응답 차이가 작았다.

목표에 가까워지면서 `Kp=0.5` 실행은 속도 제한에서 더 일찍 벗어나 실제 속도 명령을 더 빠르게 낮췄다. `t=3.7 s`에서 실행 A는 `u=9.618 deg/s`, 실행 B는 `u=6.870 deg/s`였으며, 이는 P 게인이 목표 접근 구간의 감속 특성에 영향을 준다는 것을 보여 준다.

---

## 문제 4. 제어와 통신의 역할 해석

> 이 문항은 과제에서 제공된 가상 기록과 설명용 인터페이스를 해석한 것이다. 실제 micro-ROS 연동을 새로 구현하거나 통신 단절 실험을 수행했다는 의미가 아니다.

### 1. 단위

문제 4의 설명용 인터페이스에서 `/motor/target`과 `/motor/state`는 모두 위치각이며 단위는 `deg`이다.

```text
/motor/target = target position [deg]
/motor/state  = measured position [deg]
```

이는 기존 8강의 RPM 기반 속도 인터페이스와 구분된다.

### 2. 전체 구조

```text
                  TARGET DIRECTION

PC ROS2 node
publishes /motor/target [deg]
        |
        v
micro-ROS Agent
        |
        v
OpenCR micro-ROS client
        |
        v
+-----------------------------+
| OpenCR position controller  |
| control period = 10 ms      |
+-----------------------------+
        |
        | motor command
        v
DYNAMIXEL
        |
        | measured position
        v
OpenCR
        |
        | publishes /motor/state [deg]
        | every 100 ms
        v
micro-ROS Agent
        |
        v
PC ROS2 node
subscribes /motor/state

                  STATE DIRECTION
```

목표 전달:

```text
PC ROS2 node → micro-ROS Agent → OpenCR → DYNAMIXEL
```

측정값 반환:

```text
DYNAMIXEL → OpenCR → micro-ROS Agent → PC ROS2 node
```

### 3. 구성요소별 역할

| Component | Role |
|---|---|
| PC ROS2 node | 목표 위치 발행, 상태 수신 |
| micro-ROS Agent | PC ROS2/DDS와 OpenCR micro-ROS client 사이의 통신 연결 |
| OpenCR | 목표 수신, 위치 피드백 읽기, 제어 계산, 상태 발행 |
| DYNAMIXEL | actuator command에 따라 구동하고 실제 위치 측정값을 OpenCR에 반환 |

micro-ROS Agent는 통신을 연결하고, 빠른 모터 제어 계산 자체는 OpenCR에서 수행한다.

### 4. 송수신 주체

`/motor/target`은 PC ROS2 node가 발행하고 OpenCR이 목표 위치로 사용한다.

```text
PC ROS2 node
   |
   | /motor/target = 30.0 deg
   v
micro-ROS Agent
   |
   v
OpenCR
```

`/motor/state`는 OpenCR이 DYNAMIXEL에서 읽은 위치를 발행하고 PC ROS2 node가 수신한다.

```text
DYNAMIXEL
   |
   | measured position
   v
OpenCR
   |
   | /motor/state
   v
micro-ROS Agent
   |
   v
PC ROS2 node
```

요약:

```text
/motor/target : PC → OpenCR
/motor/state  : OpenCR → PC
```

### 5. 가상 기록 해석

| Time [s] | Event |
|---:|---|
| 0.0 | PC publishes `/motor/target = 30.0 deg` |
| 0.1 | PC receives `/motor/state = 2.0 deg` |
| 0.5 | PC receives `/motor/state = 12.0 deg` |
| 1.0 | PC receives `/motor/state = 22.0 deg` |
| 2.0 | PC receives `/motor/state = 29.0 deg` |

목표는 `30.0 deg`이고 측정 위치는

```text
2.0 → 12.0 → 22.0 → 29.0 deg
```

로 증가한다. 따라서 실제 위치가 목표에 점차 가까워지는 응답을 나타낸다.

`t=2.0 s`에서의 위치 오차:

\[
e = 30.0 - 29.0 = 1.0^\circ
\]

마지막 상태가 `29.0 deg`이므로 정확히 `30 deg`에 도달했다고 해석하지 않는다.

### 6. 제어 계산 주기와 상태 발행 주기

```text
Control calculation period = 10 ms
State publication period   = 100 ms
```

상태를 한 번 발행하는 동안 수행되는 제어 계산 횟수는

\[
100\text{ ms} / 10\text{ ms} = 10
\]

이다.

| Function | Period | Frequency |
|---|---:|---:|
| Control calculation | 10 ms | 100 Hz |
| State publication | 100 ms | 10 Hz |

따라서 OpenCR은 상태 메시지 한 번을 발행하는 동안 제어 계산을 10번 수행한다.

### 7. 제어와 통신 주기의 분리

OpenCR은 매 제어 계산마다 새로운 PC 메시지를 기다릴 필요가 없다. 한 번 목표를 받은 뒤에는 다음 과정을 로컬에서 반복할 수 있다.

```text
read measured position
        ↓
calculate error
        ↓
calculate controller output
        ↓
send actuator command
        ↓
repeat every 10 ms
```

PC에는 상태를 100 ms마다 보고하므로, 통신 주기보다 빠르고 일정한 local control loop를 유지할 수 있다.

### 8. 통신 단절 시 stale command 정책

통신 단절 뒤 마지막 목표 명령을 무기한 실행하면 의도하지 않은 움직임이 지속될 수 있다. 따라서 command watchdog 또는 timeout 정책을 적용하는 것이 적절하다.

```text
receive valid /motor/target
        ↓
store target and timestamp
        ↓
new target received before timeout?
      /       \
    yes        no
     |          |
 update       mark command stale
 target         |
                v
          zero velocity /
          defined safe state
```

예를 들어 `500 ms`를 timeout으로 설정할 수 있다. 마지막 유효 목표 수신 후 500 ms 동안 새 명령이 없다면 이전 명령을 stale command로 간주하고 더 이상 그대로 사용하지 않는다.

`500 ms`는 문제에서 지정한 값이 아니라 본 답안에서 제안한 예시값이다.

실제 시스템에서는 zero velocity 또는 기구의 특성에 맞는 safe state로 전환해야 한다. 단순 torque-off가 중력 하중을 받는 관절에서는 오히려 위험할 수 있으므로 safe state는 기구 특성에 맞게 정의해야 한다.

### 9. 문제 4 결론

PC ROS2 node는 `/motor/target`을 통해 목표 위치를 OpenCR 방향으로 전달하고, OpenCR은 DYNAMIXEL에서 읽은 실제 위치를 `/motor/state`로 PC에 반환한다. micro-ROS Agent는 통신 연결을 담당하고, 위치 제어 계산은 OpenCR에서 10 ms마다 수행된다.

상태 발행 주기가 100 ms이므로 한 번의 상태 발행 사이에 총 10회의 제어 계산이 수행된다. 통신이 단절되면 오래된 목표 명령을 무기한 유지하지 않고 timeout 이후 stale command로 처리하여 zero velocity 또는 시스템에 적합한 safe state로 전환하는 정책이 적절하다.
