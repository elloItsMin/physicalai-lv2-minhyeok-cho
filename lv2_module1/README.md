# Physical AI Lv.2 — Module 1

조민혁의 Physical AI Lv.2 Module 1 과제 저장소입니다.

Raspberry Pi와 OpenCR을 이용하여 DYNAMIXEL XM430-W350을 제어하고, 위치 피드백과 P 제어 특성을 확인했습니다.

## 수행 내용

- **문제 1:** 상대 목표각 `+30 deg` 입력 및 실제 모터 응답 확인
- **문제 2:** 위치 오차와 피드백 구조 해석
- **문제 3:** `Kp = 1.0`과 `Kp = 0.5`의 응답 비교
- **문제 4:** ROS 2, micro-ROS, OpenCR, DYNAMIXEL 간 제어·통신 구조 해석

## 환경

| 항목 | 내용 |
|---|---|
| Host | Raspberry Pi |
| OS | Ubuntu Server 22.04.5 LTS |
| Controller | OpenCR 1.0 |
| Motor | DYNAMIXEL XM430-W350 |
| Protocol | DYNAMIXEL Protocol 2.0 |

## 파일 구조

```text
lv2_module1/
├── report.md
└── results/
    ├── environment.txt
    ├── upload.log
    ├── run_A.log
    └── run_B.log
```

상세한 실험 과정과 결과는 [`lv2_module1/report.md`](lv2_module1/report.md)에 정리했습니다.
