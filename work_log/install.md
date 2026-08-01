# 🦾 lelab_twins 설치 및 실행 가이드 (초보자용)

이 문서는 `lelab_Twins`(양팔 SO-101/OMX-AI 지원이 추가된 LeLab)를 **처음부터** 설치하고 실행하는 방법을 순서대로 설명합니다. 이 저장소 전용 가상환경 이름은 `lelab_twins`이며, 이 환경은 Ubuntu(ROS2 설치된 머신) 기준으로 작성되었습니다.

---

## 📋 사전 준비물 (Prerequisites)

- **OS**: Ubuntu (또는 다른 Linux 배포판)
- **Python**: 3.12 이상
- **git**: 소스 코드/lerobot 설치용
- 실제 로봇으로 테스트하려면: SO-101 또는 OMX-AI 리더/팔로워 암, USB 케이블, (양팔이면) 4대의 암 + 카메라

터미널에서 파이썬 버전 확인:
```bash
python3 --version
```
3.12 이상이면 OK.

---

## ⚙️ 1단계: 프로젝트 폴더로 이동

```bash
cd ~/lelab_Twins
```

---

## 🐍 2단계: 가상환경(venv) 만들기

이 프로젝트 전용 가상환경 `lelab_twins`를 폴더 안에 만듭니다. (다른 프로젝트의 가상환경과 이름이 겹치지 않도록 새로 만드는 것입니다.)

```bash
python3 -m venv lelab_twins
```

가상환경 활성화:
```bash
source lelab_twins/bin/activate
```
활성화되면 터미널 프롬프트 앞에 `(lelab_twins)`가 붙습니다.

> 이후 모든 명령어는 **이 가상환경이 활성화된 상태**에서 실행해야 합니다. 새 터미널을 열 때마다 `source lelab_twins/bin/activate`를 다시 실행해주세요.

---

## 📥 3단계: 패키지 설치

```bash
pip install --upgrade pip
pip install -e ".[test]"
```

- `lerobot`(torch 포함)을 GitHub에서 직접 받기 때문에 **용량이 크고(수 GB) 시간이 꽤 걸립니다** (네트워크 상황에 따라 수 분~수십 분).
- 설치가 끝나면 `lelab`, `pytest`, `ruff` 등 명령어를 바로 쓸 수 있습니다.

### ⚠️ ROS2가 설치된 머신에서 주의할 점

이 머신처럼 ROS2(예: jazzy)가 설치되어 있으면, 쉘 프로필이 `PYTHONPATH`에 ROS2 경로를 자동으로 넣어둡니다. 이게 `lelab_twins` 가상환경과 섞이면 import 에러가 날 수 있습니다. **`lelab`이나 `pytest`를 실행할 때는 `PYTHONPATH`를 비우고 실행하세요:**

```bash
env -u PYTHONPATH lelab --dev
env -u PYTHONPATH pytest -q
```

---

## ✅ 4단계: 설치 확인 (선택)

```bash
env -u PYTHONPATH pytest -q
```
마지막 줄에 `passed`가 뜨면 정상 설치된 것입니다.

```bash
uvx ruff check lelab tests
```
(인터넷에서 `ruff`를 한 번 받아오며, 코드 스타일 검사만 합니다.)

---

## 🚀 5단계: 실행하기

### A. 개발 모드 (권장 — 코드 수정 시 자동 반영)
```bash
env -u PYTHONPATH lelab --dev
```
- 백엔드(FastAPI)는 `http://localhost:8000`, 프런트엔드(Vite)는 `http://localhost:8080`에서 실행되고 브라우저가 자동으로 열립니다.
- 코드를 수정하면 자동으로 반영됩니다(hot-reload).
- 브라우저를 자동으로 열지 않으려면: `env -u PYTHONPATH lelab --dev --no-open`

### B. 일반 모드 (빌드된 프런트엔드 사용)
```bash
env -u PYTHONPATH lelab
```
포트 `8000` 하나로 백엔드+프런트엔드가 함께 실행됩니다.

### 종료하기
터미널에서 `Ctrl+C`를 누르면 됩니다.

---

## 🖥️ 6단계: 처음 화면에서 할 일 (양팔 로봇 기준)

1. 브라우저에서 `http://localhost:8080` (또는 `:8000`) 접속
2. **로봇 만들기**: 로봇 선택 드롭다운을 눌러 이름을 입력
   - **"Bimanual (2 arms)" 체크박스**를 켜면 왼쪽/오른쪽 팔의 모델(SO-101 / OMX-AI)을 각각 선택할 수 있습니다.
   - 이미 만들어둔 단일팔 로봇을 나중에 양팔로 바꾸고 싶다면 → Calibration 페이지의 Configuration 카드에서 같은 토글을 켜면 됩니다.
3. **캘리브레이션**: 로봇 타일의 톱니바퀴(⚙) 아이콘 클릭 → Calibration 페이지로 이동
   - 양팔 로봇이면 "Arm" 선택(Left/Right) + "Device Type" 선택(Teleoperator/Robot) 조합으로 **총 4단계**를 순서대로 진행합니다: Left leader → Left follower → Right leader → Right follower
   - 각 단계마다: 포트 확인/감지("Find" 버튼) → **"Start Calibration"** 클릭 → 팔을 실제로 움직여 관절 전체 범위 이동 → **"Save Calibration"** 클릭
   - 4단계 모두 체크(✓)되면 로봇 타일 상태가 "Ready"로 바뀌고 Teleoperation 버튼이 활성화됩니다.
4. **텔레오퍼레이션**: "Teleoperation" 버튼 클릭 → 리더 팔을 움직이면 팔로워 팔이 따라 움직이는지 확인 (양팔이면 화면에 좌/우 3D 뷰어가 동시에 뜹니다)
5. **레코딩(데이터셋 수집)**: 메인 화면에서 데이터셋 이름/작업 설명/에피소드 수 등을 입력하고 카메라를 연결한 뒤 녹화 시작

---

## 🛠️ 자주 만나는 문제

| 증상 | 원인 / 해결 |
|---|---|
| `lelab: command not found` | 가상환경이 활성화 안 됨 → `source lelab_twins/bin/activate` 먼저 실행 |
| `pytest`에서 `ModuleNotFoundError: lark` 등 이상한 에러 | ROS2의 `PYTHONPATH`가 섞임 → `env -u PYTHONPATH` 붙여서 실행 |
| 카메라 여러 대 연결했는데 일부만 "사용 가능"으로 보임 | 브라우저가 이미 다른 카메라를 미리보기로 열어둔 상태일 수 있음 → "Attached cameras" 토글을 껐다 켜거나, 새로고침(⟳) 버튼으로 재스캔 |
| "Failed to write 'Homing_Offset' ... torque" 같은 에러로 텔레오퍼레이션/캘리브레이션 시작이 안 됨 | 팔의 전원을 껐다 켜서(power-cycle) 모터 토크 상태를 리셋해보세요 |
| "Failed to sync read ... no status packet" (간헐적) | 케이블 접촉 불량/전원 문제일 가능성이 높음 → USB 케이블 재체결, 허브 대신 본체 포트에 직결 |
| 포트(`/dev/ttyACM*`)가 자꾸 바뀜 | 케이블을 뽑았다 꽂으면 리눅스가 새 번호를 줄 수 있음 → Calibration 페이지의 "Find" 버튼으로 다시 감지 |

더 자세한 개발 배경/버그 수정 이력은 같은 폴더의 `2026-07-26_bimanual_support.md`를 참고하세요.
