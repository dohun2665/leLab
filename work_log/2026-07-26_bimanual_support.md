# 양팔(Bimanual) SO-101/OMX-AI 지원 작업 로그

- **기간**: 2026-07-26 ~ 2026-07-27
- **목표**: `lelab_Twins`(lelab 코드 복사본)를 리더 1대+팔로워 1대(단일팔) 구조에서, 좌/우 각각 리더+팔로워를 가진 **양팔(bimanual)** 구조로 확장. 각 팔은 SO-101 또는 OMX-AI를 독립적으로 선택 가능.
- **범위**: 백엔드(FastAPI) + 프런트엔드(React) 전체, 실제 SO-101/OMX-AI 하드웨어로 캘리브레이션 → 텔레오퍼레이션 → 레코딩까지 end-to-end 검증.

---

## 1. 아키텍처 결정

기존 코드(`teleoperate.py`, `record.py` 등)는 "리더 1개 + 팔로워 1개"를 전제로 작성되어 있었음. 두 팔을 지원하기 위해, `lerobot`이 양팔을 네이티브로 지원하는지 여부와 무관하게 동작하도록 **lelab 자체에 합성(composite) 래퍼**를 만드는 방식을 택함:

- 좌/우 각각 독립적인 단일팔 인스턴스(`SO101Follower`/`OmxFollower`/`SO101Leader`/`OmxLeader`)를 만들고, 이를 감싸는 `BimanualRobot`/`BimanualTeleoperator` 클래스가 관측/액션/카메라 키에 `left_`/`right_` 접두어를 붙여 병합.
- 이 덕분에 `teleoperate.py`의 텔레옵 루프, `record.py`의 레코딩 로직을 거의 그대로 재사용 가능.
- 로봇 레코드 스키마는 "왼쪽 = 기존 필드 재해석" 방식으로 확장 — `mode`("single"/"bimanual") + 우측 팔 5개 필드(`right_leader_port` 등)만 추가하여 **기존 단일팔 로봇 100% 하위호환** 유지.

---

## 2. 백엔드 변경 파일

| 파일 | 변경 내용 |
|---|---|
| `lelab/utils/bimanual.py` (신규) | `BimanualRobotConfig`/`BimanualRobot`/`BimanualTeleoperatorConfig`/`BimanualTeleoperator`. `left_`/`right_` 접두어로 관측·액션·카메라 병합/분배. `.bus` shim으로 `write_calibration()`/`disable_torque()` 위임. **`lerobot`의 `Robot`/`Teleoperator` 추상 클래스를 실제로 상속** (초기엔 duck-typing만 했다가, `record_loop`의 `isinstance` 검사 때문에 나중에 상속으로 변경 — §5.4 참고). |
| `lelab/utils/devices.py` | `make_bimanual_device_config()`/`make_bimanual_device()` 추가 — 기존 `make_device_config`/`make_device`를 좌우 두 번 호출해 재사용. |
| `lelab/utils/config.py` | 로봇 레코드에 `mode` + `right_*` 5필드 추가. `RobotSide` 리터럴에 `right_leader`/`right_follower` 추가(+대응 포트/설정 파일 상수). `is_robot_record_clean()`이 bimanual 모드일 때 양쪽 4개 캘리브레이션 파일 존재까지 확인하도록 확장. |
| `lelab/calibrate.py` | `CalibrationRequest`에 `side: "left"\|"right"` 필드 추가 — 완료 후 로봇 레코드 write-back 시 `right_*` 필드로 저장할지 결정하는 데만 사용(캘리브레이션 스텝 로직 자체는 무변경). |
| `lelab/teleoperate.py` | 양팔용 `_start_bimanual_teleoperation()` 추가(4개 장치 연결, 실패 시 어느 팔인지 명시하는 에러, 4개 버스 순차 연결). `get_joint_positions_from_bimanual_robot()` 추가 — 좌우 URDF 조인트 이름에 접두어. |
| `lelab/record.py` | `RecordingRequest`에 `right_*` 필드 추가. `_split_cameras_by_side()` — 카메라 이름의 `left_`/`right_` 접두어로 좌우 팔로워에 분배(접두어 없으면 왼쪽). `record_with_web_events()`에서 `BimanualRobotConfig`/`BimanualTeleoperatorConfig`일 때 `make_robot_from_config` 대신 `BimanualRobot.from_config()` 직접 호출하는 분기 추가. |
| `lelab/server.py` | `RobotSideLiteral`을 4값(`leader`/`follower`/`right_leader`/`right_follower`)으로 확장. |
| `tests/` | `test_bimanual.py`(신규), `test_utils_config.py`/`test_calibrate.py`/`test_teleoperate.py`/`test_record.py`에 양팔 케이스 추가. 최종 **pytest 215개 통과**. |

정책 추론(rollout.py)은 이번 범위에서 제외 — `lerobot_rollout` CLI가 양팔을 어떻게 지원하는지 별도 확인이 필요해 후속 작업으로 명시.

---

## 3. 프런트엔드 변경 파일

| 파일 | 변경 내용 |
|---|---|
| `hooks/useRobots.ts` | `RobotRecord`에 `mode?`, `right_*` 5필드 추가. `createRobot()`이 `{bimanual, rightRobotType}` 옵션을 받도록 확장. |
| `components/landing/RobotSelector.tsx` | 로봇 생성 팝오버에 "Bimanual (2 arms)" 체크박스 추가 → 켜면 "Left arm:"/"Right arm:" 모델(SO-101/OMX-AI) 각각 선택. |
| `components/landing/RobotConfigManager.tsx` | `handleTeleop()`이 `robot.mode === "bimanual"`일 때 `right_*` 필드도 `/move-arm` 요청에 포함. |
| `pages/Calibration.tsx` | 이진(`teleop`/`robot`) 상태에 `side`("left"/"right") 축 추가 → **4단계 체크리스트**(Left leader/follower, Right leader/follower)로 확장. 기존 로봇도 나중에 양팔로 전환할 수 있도록 **Configuration 카드에 "Bimanual (2 arms)" 토글 추가**(대화 중 사용자 피드백으로 추가됨 — 처음엔 생성 시점에만 있었음). |
| `contexts/UrdfContext.tsx`, `components/UrdfViewer.tsx`, `hooks/useRealTimeJoints.ts` | `UrdfViewer`가 `robotType`/`jointPrefix` prop을 받아 전역 컨텍스트 대신 명시적으로 좌/우 모델을 그릴 수 있도록 확장. `useRealTimeJoints`가 `left_`/`right_` 접두어로 조인트를 필터링. |
| `components/control/VisualizerPanel.tsx` | `bimanualViewers` prop 추가 — 좌우 `UrdfViewer` 2개를 나란히 렌더링. |
| `pages/Teleoperation.tsx` | 선택된 로봇이 bimanual이면 좌우 뷰어를 동시에 표시하도록 연결. |
| `pages/Landing.tsx` | `handleStartRecording()`에서 `robot.mode === "bimanual"`이면 `right_*` 필드를 레코딩 요청에 포함. HF 미로그인 시 `local/` 네임스페이스 접두어 추가(§5.3 버그 수정). |
| `components/recording/CameraConfiguration.tsx` | 카메라 새로고침 버튼이 스캔 직전 미리보기를 잠깐 멈췄다 재개하도록 수정(§5.1 버그 수정). |
| `components/ui/PortDetectionModal.tsx`/`PortDetectionButton.tsx` | `robotType` prop을 4값(`leader`/`follower`/`right_leader`/`right_follower`) 유니언으로 확장. |

---

## 4. 초기 검증 (하드웨어 테스트 이전)

- `pytest`: **215 passed**
- `ruff check`/`ruff format`(수정 범위): 통과
- 프런트엔드 `tsc --noEmit`: 통과 (기존 무관 에러 1개 제외)
- 헤드리스 브라우저(Playwright)로 UI 스모크 테스트: 로봇 생성 시 Bimanual 토글, 4단계 캘리브레이션 체크리스트 렌더링 확인, 콘솔 에러 0건

---

## 5. 실제 하드웨어 테스트 중 발견/수정한 버그

`lelab --dev`로 실제 SO-101/OMX-AI 하드웨어(리더 2대 + 팔로워 2대, 카메라 3대)를 연결해 테스트하며 발견된 문제들. **발견 순서대로** 기록.

### 5.1 카메라: 동일 모델 3대 중 1~2대만 인식됨
- **증상**: 동일 모델 USB 카메라 3대 연결 시 2대만 "사용 가능"으로 표시, 3번째를 추가할 수 없음.
- **원인 조사 과정**:
  1. 처음엔 하드웨어 USB 대역폭 문제로 추정 → `lsusb`/`fuser`로 확인해보니 실제로는 **Chrome의 Video Capture Service 프로세스가 이미 추가된 카메라 2대의 장치를 붙잡고 있어서** 백엔드(cv2)가 스캔 시 열지 못하는 것으로 확인.
  2. 크롬 프로세스를 강제 종료해 임시로 해결했으나, 재발함 — 근본 원인은 **"Attached cameras" 미리보기가 켜져 있으면 항상 2대를 점유**하는 구조적 문제.
- **수정**: `CameraConfiguration.tsx`의 새로고침 버튼이 스캔 직전 모든 미리보기 스트림을 0.4초간 멈췄다가 스캔 후 재개하도록 변경(`rescanCameras()`). 이렇게 하면 스캔하는 순간만은 백엔드가 카메라 3대를 모두 볼 수 있음.
- **검증**: 실제 하드웨어로 3대 모두 인식 확인.

### 5.2 로깅: 예외 트레이스백이 로그에서 사라짐
- **증상**: 레코딩 실패 시 에러 메시지("not enough values to unpack...")만 보이고 정확한 위치(파일:줄)를 알 수 없음.
- **원인**: `lerobot`의 `init_logging()`이 루트 로거 포맷터를 커스텀 함수로 교체하는데, 이 함수가 `record.getMessage()`만 사용하고 `exc_info`(트레이스백)를 전혀 렌더링하지 않음 — `logger.exception()`을 호출해도 트레이스백이 통째로 사라짐.
- **수정**: `record.py`의 레코딩 워커 예외 처리부에서 `traceback.format_exc()`를 명시적으로 메시지 문자열에 포함해 로그로 남기도록 변경.

### 5.3 데이터셋 이름: HF 로그인 안 하면 레코딩이 즉시 실패
- **증상**: "Recording Session Failed — not enough values to unpack (expected 2, got 1)".
- **원인**: (5.2의 로깅 수정 덕분에 트레이스백 확보) `lerobot.common.control_utils.sanity_check_dataset_name()`이 `repo_id.split("/")`를 무조건 2개로 언패킹하는데, Hugging Face 로그인이 안 되어 있으면 `dataset_repo_id`에 `"네임스페이스/"`가 없어 슬래시가 0개라 실패.
- **수정**: `Landing.tsx`의 `handleStartRecording()`에서 미인증 상태일 때 `local/{datasetName}` 형태로 네임스페이스를 붙이도록 변경(`push_to_hub=false`라 실제 Hub 호출에는 영향 없음).
- **비고**: 이 버그는 양팔 기능과 무관한 **기존(pre-existing) 버그** — 단일팔에서도 HF 미로그인 상태로 레코딩하면 동일하게 실패했을 것으로 추정.

### 5.4 레코딩 중 리더를 움직여도 팔로워가 안 움직임 (프레임도 0개)
- **증상**: "Recording Session Failed — You must add one or several frames with `add_frame` before calling `add_episode`."
- **원인**: `lerobot.scripts.lerobot_record.record_loop()`가 텔레옵 객체를 `isinstance(teleop, Teleoperator)`로 검사하는데, `BimanualTeleoperator`/`BimanualRobot`은 (당시) duck-typing만 한 별도 클래스라 이 검사를 통과하지 못함. `record_loop`는 "텔레옵이 없다"고 오판하고 매 반복마다 `get_action`/`send_action`/`add_frame`을 전부 건너뜀(`continue`) — 에러 없이 조용히 실패.
- **수정**: `lelab/utils/bimanual.py`에서 `BimanualRobot(_BimanualDevice, Robot)`, `BimanualTeleoperator(_BimanualDevice, Teleoperator)`로 **실제 상속**하도록 변경. `Robot.__init__`/`Teleoperator.__init__`(파일 기반 캘리브레이션 로직, 좌우 합성 구조엔 안 맞음)은 호출하지 않고, 추상 메서드(`is_calibrated`, `feedback_features`, `send_feedback` 등)만 추가 구현.
- **검증**: 트레이스백이 `robot.get_observation()` 등 실제 하드웨어 호출까지 도달하는 것으로 확인 — isinstance 문제 해결됨.

### 5.5 (하드웨어) 리더 팔 시리얼 통신 간헐적 끊김
- **증상**: "Failed to sync read 'Present_Position' ... There is no status packet!" — 텔레오퍼레이션/레코딩 중 간헐적으로 발생.
- **원인**: 모터 ID 1~6(OMX-AI 리더 팔의 모터 ID 체계, 팔로워는 11~16)에서 통신 응답이 없거나 깨짐. 하루 종일 간헐적으로 반복 발생한 것으로 보아 **코드 문제가 아니라 케이블/전원 접촉 불량 등 하드웨어 신뢰성 문제**로 판단.
- **조치**: 코드 수정 없음 — 사용자에게 케이블 재체결/직결 USB 포트 사용을 안내.

### 5.6 캘리브레이션 재적용 시 "Homing_Offset 쓰기 실패"로 텔레오퍼레이션 시작 불가
- **증상**: "Error Starting Teleoperation — Failed to write 'Homing_Offset' ... [RxPacketError] Writing or Reading is not available to target address!" — 재시도해도 100% 재현.
- **원인**: `Homing_Offset`은 EEPROM 영역 레지스터라 **토크(torque)가 켜진 상태에서는 쓰기가 거부됨**. lerobot의 자체 캘리브레이션(`calibrate()`)은 쓰기 전에 `disable_torque()`를 호출하지만, 이미 캘리브레이션 파일이 있어 재사용하는 경로(`write_calibration()`만 호출)는 토크를 끄지 않음 — **원래 단일팔 코드에도 있던 기존 버그**. 이전 세션이 비정상 종료되어 토크가 켜진 채 남으면 이후 모든 연결 시도가 실패.
- **수정**: `teleoperate.py`(단일팔+양팔), `record.py`, `utils/bimanual.py`(`_BimanualBusShim`에 `disable_torque()` 추가)에서 `write_calibration()` 직전에 `bus.disable_torque()`를 먼저 호출하도록 수정.

### 5.7 (5.6의 부작용) 레코딩 중 캘리브레이션 재적용 후 팔로워가 안 움직임
- **증상**: 5.6 수정 이후, 레코딩은 에러 없이 진행되고 프레임도 저장되지만 **팔로워가 실제로는 움직이지 않음**(일반 텔레오퍼레이션은 정상 동작).
- **원인**: `record.py`는 `robot.connect()`(전체 메서드)를 먼저 호출하는데, 이 안에서 이미 `configure()`가 실행되어 토크가 켜짐. 그 후 5.6에서 추가한 `disable_torque()` → `write_calibration()`이 실행되지만, **토크를 다시 켜주는 코드가 없어서** 그 이후 세션 내내 토크가 꺼진 채로 남음. `send_action()`은 예외 없이 "성공"하지만 모터는 토크 꺼짐 상태라 실제로 움직이지 않음.
  (일반 텔레오퍼레이션 코드는 원래부터 "캘리브레이션 쓰기 → `configure()`" 순서라 이 문제가 없었음.)
- **수정**: `record.py`의 캘리브레이션 쓰기 직후 `robot.configure()`/`teleop.configure()`를 다시 호출해 토크 상태를 정상 복구.
- **검증**: 사용자가 실제 하드웨어로 레코딩 확인 — **정상 동작 확인**("잘 됩니다").

---

## 6. 최종 상태

- pytest 215개 통과(환경 특이적인 무관 테스트 1개 제외), ruff/tsc 통과.
- 실제 SO-101/OMX-AI 양팔 하드웨어로 캘리브레이션 → 텔레오퍼레이션 → 레코딩 전체 플로우 정상 동작 확인.
- 후속 과제로 남은 것: 정책 추론(rollout, `lerobot_rollout`)의 양팔 지원은 이번 범위에서 제외됨.
