# 소프트웨어 설계 명세서
## 반려 로봇 (AI ROOKIE 2026)

**버전:** 1.0  
**작성일:** 2026-04-29  
**상태:** DRAFT  
**참조 설계 문서:** `~/.gstack/projects/ICT/home-unknown-design-20260429-165641.md`

---

## 목차

1. [시스템 개요](#1-시스템-개요)
2. [아키텍처 다이어그램](#2-아키텍처-다이어그램)
3. [모듈 명세](#3-모듈-명세)
   - 3.1 InputManager
   - 3.2 EmotionFusion
   - 3.3 StateMachine
   - 3.4 MemoryManager
   - 3.5 OutputManager
   - 3.6 PersonalityEngine
4. [데이터 구조](#4-데이터-구조)
5. [스레드 모델](#5-스레드-모델)
6. [설정 파일](#6-설정-파일)
7. [에러 처리 전략](#7-에러-처리-전략)
8. [테스트 계획](#8-테스트-계획)
9. [개발 환경 설정](#9-개발-환경-설정)
10. [파일 구조](#10-파일-구조)
11. [개발 단계별 구현 순서](#11-개발-단계별-구현-순서)

---

## 1. 시스템 개요

### 1.1 핵심 철학

"말하지 않는 반려 로봇" — 사용자의 감정 상태를 시각/청각으로 감지하고,  
**TTS 없이** 표정, 몸짓, 소리로만 공감을 표현한다.

| 항목 | 값 |
|------|-----|
| 언어 | Python 3.11 |
| 실행 환경 | 노트북(개발) → Raspberry Pi 5(배포) |
| 주요 입력 | 카메라(얼굴 감정) + 마이크(음성 내용) |
| 주요 출력 | OLED 눈 + 서보 모터 + 효과음 |
| 응답 목표 지연 | 5초 이내 (감정 감지 → 반응 완료) |
| 대화 기능 | 없음. STT는 감정 분석 전용 |

### 1.2 핵심 데이터 플로우

```
카메라 프레임 ──→ DeepFace (3fps)    ──────────────────┐
             └──→ MediaPipe Pose (15fps) ─────────────┤
                                                       ├──→ EmotionFusion ──→ StateMachine ──→ OutputManager
마이크 오디오 ──→ VAD ──→ Whisper STT ──→              │     face=0.5          │
                            감성사전 분석 ──────────────┘     gesture=0.2        │
                                                               text=0.3          ↓
                                                                           MemoryManager
                                                                           (감정 로그 저장)
                                                                                 │
                                                                                 ↓
                                                                          PersonalityEngine
                                                                          (반응 강도 조정)
```

---

## 2. 아키텍처 다이어그램

### 2.1 전체 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────────────┐
│                        COMPANION ROBOT SW                           │
├─────────────────┬───────────────────────────────────────────────────┤
│   INPUT LAYER   │   PROCESSING LAYER          │   OUTPUT LAYER      │
│                 │                             │                     │
│  ┌───────────┐  │  ┌──────────────────────┐  │  ┌───────────────┐  │
│  │  Camera   │──┼─▶│   EmotionFusion      │  │  │  OLED Eyes    │  │
│  │(picamera2)│  │  │  face+gesture+text   │  │  │  (SSD1306)    │  │
│  └───────────┘  │  │  3-channel weighted  │  │  └───────────────┘  │
│        │        │  └──────────┬───────────┘  │        ▲            │
│        ├──▶ DeepFace (3fps)   │               │        │            │
│        └──▶ MediaPipe (15fps) │               │        │            │
│                 │  ┌──────────▼───────────┐  │  ┌─────┴─────────┐  │
│                 │  │    StateMachine      │──┼─▶│  OutputMgr    │  │
│                 │  │  5 states + timeout  │  │  │               │  │
│  ┌───────────┐  │  └──────────┬───────────┘  │  │  ┌─────────┐  │  │
│  │   Mic     │  │  ┌──────────▼───────────┐  │  │  │ Servos  │  │  │
│  │  (PyAudio)│  │  │    MemoryManager     │  │  │  │(PCA9685)│  │  │
│  └───────────┘  │  │  SQLite log +        │  │  │  └─────────┘  │  │
│        │        │  │  pattern analysis    │  │  │               │  │
│        ▼        │  └──────────┬───────────┘  │  │  ┌─────────┐  │  │
│  ┌───────────┐  │  ┌──────────▼───────────┐  │  │  │ Sound   │  │  │
│  │  VAD +    │  │  │  PersonalityEngine   │  │  │  │(pygame) │  │  │
│  │  Whisper  │  │  │  intensity adjuster  │  │  │  └─────────┘  │  │
│  └───────────┘  │  └──────────────────────┘  │  └───────────────┘  │
└─────────────────┴───────────────────────────────────────────────────┘
```

### 2.2 상태 기계 다이어그램

```
                    ┌─────────────────┐
          ┌────────▶│    NEUTRAL      │◀────────┐
          │         │   (대기/스캔)    │         │
          │         └────────┬────────┘         │
          │  [timeout 10s]   │                  │ [confidence < 0.5]
          │                  │                  │
    ┌─────┴──────┐    ┌──────┴──────┐    ┌──────┴──────┐
    │   HAPPY    │    │     SAD     │    │    ANGRY    │
    │ (기쁨 반응) │    │  (위로 모드) │    │  (진정 모드) │
    └─────┬──────┘    └──────┬──────┘    └──────┬──────┘
          │                  │                  │
          └──────────────────┴──────────────────┘
                             │
                    ┌────────▼────────┐
                    │      FEAR       │
                    │   (안심 모드)    │
                    └─────────────────┘

  전환 조건: 동일 감정이 3초 이상 지속 AND confidence ≥ 0.5
  복귀 조건: 다른 감정 3초 이상 OR 얼굴 미감지 5초 이상 → NEUTRAL
```

### 2.3 스레드 아키텍처

```
Main Thread
│
├── Thread: CameraWorker (daemon)
│     └── 30fps 캡처 → frame_queue (maxsize=3, 오래된 프레임 드롭)
│
├── Thread: VideoAnalyzer (daemon)          ← EmotionDetector에서 이름 변경, 역할 확장
│     └── frame_queue 소비
│     └── 매 10번째 프레임 (3fps): DeepFace 추론
│     │     └── shared_state.update_visual(EmotionResult) 업데이트
│     └── 매 2번째 프레임 (15fps): MediaPipe Pose 추론
│           └── shared_state.update_gesture(GestureResult) 업데이트
│     (DeepFace와 MediaPipe는 같은 스레드에서 순차 실행 — 프레임 카운터로 분기)
│
├── Thread: AudioWorker (daemon)
│     └── silero-vad → 발화 감지 → faster-whisper STT
│     └── 감성사전 매핑 → shared_state.update_text(SentimentResult) 업데이트
│
└── MainLoop (main)
      └── 100ms 주기 polling
      └── EmotionFusion.compute(face, gesture, text) → StateMachine.tick() → OutputManager.execute()
      └── MemoryManager.log() (매 5초)

프레임 카운터 분기 로직 (VideoAnalyzer 내부):
  frame_count % 10 == 0 → DeepFace 실행 (~300ms, 완료까지 블로킹)
  frame_count % 2  == 0 → MediaPipe 실행 (~50ms)
  → 두 조건 겹치는 프레임: DeepFace 먼저, 이후 MediaPipe (총 ~350ms)
  → RPi 5 4코어 기준 허용 범위 내
```

---

## 3. 모듈 명세

### 3.1 InputManager

**역할:** 카메라 및 마이크 입력 스트림 관리

#### 3.1.1 CameraCapture

```python
class CameraCapture:
    """
    단일 OpenCV 캡처 스레드. 10fps 고정.
    frame_queue: Queue(maxsize=2) — 오래된 프레임 자동 드롭.
    """

    def __init__(self, device_id: int = 0, fps: int = 10):
        self.cap = cv2.VideoCapture(device_id)
        self.frame_queue: Queue[np.ndarray] = Queue(maxsize=2)
        self.running = False

    def start(self) -> None:
        """백그라운드 스레드로 캡처 시작"""

    def stop(self) -> None:
        """안전 종료. cap.release() 포함"""

    def get_frame(self, timeout: float = 0.1) -> Optional[np.ndarray]:
        """최신 프레임 반환. timeout 초과 시 None"""
```

#### 3.1.2 VideoAnalyzer

```python
class VideoAnalyzer:
    """
    DeepFace(얼굴 표정)와 MediaPipe Pose(신체 제스처)를 단일 스레드에서
    순차 실행. CameraCapture의 frame_queue를 소비.
    결과를 SharedState에 스레드 안전하게 기록.

    프레임 스케줄:
      매 10번째 프레임 (30fps 기준 → 3fps): DeepFace 추론
      매  2번째 프레임 (30fps 기준 → 15fps): MediaPipe Pose 추론
    """

    FACE_SKIP    = 10  # DeepFace: 30fps ÷ 10 = 3fps 분석
    GESTURE_SKIP = 2   # MediaPipe: 30fps ÷ 2  = 15fps 분석

    EMOTION_MAP = {
        "happy": "happy",
        "sad": "sad",
        "angry": "angry",
        "fear": "fear",
        "surprise": "surprise",
        "neutral": "neutral",
        "disgust": "angry",  # disgust → angry 매핑
    }

    def __init__(self, camera: CameraCapture, state: SharedState):
        self.frame_count = 0
        self.face_detector = FaceEmotionDetector()   # DeepFace 래퍼
        self.gesture_detector = GestureDetector()    # MediaPipe Pose 래퍼

    def run(self) -> None:
        """스레드 메인 루프. CameraWorker 종료까지 실행."""

    def _analyze_face(self, frame: np.ndarray) -> EmotionResult:
        """
        DeepFace로 얼굴 감정 추론.
        Returns: EmotionResult(emotion, confidence, face_detected)
        enforce_detection=False: 얼굴 없을 때 예외 대신 neutral 반환
        """

    def _analyze_gesture(self, frame: np.ndarray) -> GestureResult:
        """MediaPipe Pose로 신체 제스처 감정 추론."""
```

#### 3.1.3 GestureDetector

```python
class GestureDetector:
    """
    MediaPipe Pose로 신체 랜드마크를 분석해 제스처 기반 감정 신호 생성.
    VideoAnalyzer 스레드에서 매 2번째 프레임(15fps) 호출.

    감지 제스처 → 감정 매핑:
      head_down     → sad    (코 y좌표 < 어깨 중심 y좌표 − 임계값)
      hunched       → sad    (어깨 너비 < 기준값 AND 어깨 y좌표 높음)
      lean_forward  → happy  (코 z좌표 감소 추이, 카메라 접근)
      arms_crossed  → angry  (손목이 반대쪽 팔꿈치보다 몸 중심 가까이)
      covering_face → fear   (손 랜드마크 y좌표 ≈ 코 y좌표)
      neutral_pose  → neutral (위 조건 미충족)

    MediaPipe 설정:
      model_complexity=0 (Lite 모델) — RPi 5 CPU 최적화
      min_detection_confidence=0.5
      min_tracking_confidence=0.5
    """

    GESTURE_SENTIMENT = {
        "head_down":     "negative",
        "hunched":       "negative",
        "lean_forward":  "positive",
        "arms_crossed":  "negative",
        "covering_face": "negative",
        "neutral_pose":  "neutral",
    }

    def __init__(self):
        import mediapipe as mp
        self.pose = mp.solutions.pose.Pose(
            model_complexity=0,
            min_detection_confidence=0.5,
            min_tracking_confidence=0.5,
        )
        self._prev_nose_z: Optional[float] = None  # lean_forward 추적용

    def analyze(self, frame: np.ndarray) -> GestureResult:
        """
        Returns: GestureResult(gesture, sentiment, confidence, body_detected)
        body_detected=False → sentinel 값 (EmotionFusion에서 gesture 채널 제외)
        """

    def _classify_pose(self, landmarks) -> tuple[str, float]:
        """
        MediaPipe landmark 좌표 → 제스처 분류.
        Returns: (gesture_name, confidence)
        confidence = landmark visibility 최솟값 (신뢰도 하한)
        """
```

#### 3.1.3 AudioWorker

```python
class AudioWorker:
    """
    silero-vad로 발화 감지 → faster-whisper로 STT → 감성사전으로 감정 추출.
    TTS 없음. 결과는 SharedState의 text_emotion에만 기록.
    """

    VAD_SPEECH_THRESHOLD = 0.5   # 초: 이 이상 발화 감지 시 녹음 시작
    VAD_SILENCE_THRESHOLD = 1.5  # 초: 이 이상 침묵 시 녹음 종료

    NEGATIVE_KEYWORDS = ["힘들", "슬퍼", "무서워", "지쳐", "싫어", "피곤", "우울"]
    POSITIVE_KEYWORDS = ["좋아", "기뻐", "신나", "행복", "즐거워", "고마워"]

    def transcribe(self, audio: np.ndarray) -> str:
        """faster-whisper tiny 모델. language='ko' 고정"""

    def analyze_sentiment(self, text: str) -> SentimentResult:
        """
        키워드 기반 감성 분석.
        Returns: SentimentResult(sentiment, matched_keywords)
        sentiment: "negative" | "positive" | "neutral"
        """
```

---

### 3.2 EmotionFusion

**역할:** 시각 감정(DeepFace)과 텍스트 감정(STT)을 가중 평균으로 합산

```python
class EmotionFusion:
    """
    3채널 퓨전: 얼굴 표정 + 신체 제스처 + 텍스트 감성

    가중치:
      FACE_WEIGHT    = 0.5   # 얼굴 표정 — 가장 직접적인 감정 신호
      GESTURE_WEIGHT = 0.2   # 신체 제스처 — 보완 (body_detected=False 시 비율 재배분)
      TEXT_WEIGHT    = 0.3   # 발화 내용 — 보완

    제스처 미감지(body_detected=False) 시 자동 재배분:
      → face=0.7, text=0.3 (이전 2채널 비율 유지)

    예시:
      face=sad(0.9),  gesture=head_down, text=negative → 최종=sad (강한 확신)
      face=neutral,   gesture=lean_forward, text=positive → 최종=happy (제스처+텍스트 합산)
      face=happy(0.8), gesture=arms_crossed → 최종=neutral (상충, 낮은 강도)
    """

    FACE_WEIGHT    = 0.5
    GESTURE_WEIGHT = 0.2
    TEXT_WEIGHT    = 0.3

    def compute(self, face: EmotionResult, gesture: GestureResult, text: SentimentResult) -> FusedEmotion:
        """
        Returns: FusedEmotion(emotion, confidence, source)
        source: "face_only" | "gesture_only" | "text_only" | "fused"
        gesture.body_detected=False 시 gesture 채널 제외하고 face/text 2채널로 재계산
        """
```

---

### 3.3 StateMachine

**역할:** 감정 입력을 받아 행동 상태를 결정하고 전환을 관리

```python
class RobotState(Enum):
    NEUTRAL  = "neutral"   # 대기, 좌우 스캔
    HAPPY    = "happy"     # 기쁨 반응
    SAD      = "sad"       # 위로 모드
    ANGRY    = "angry"     # 진정 모드
    FEAR     = "fear"      # 안심 모드

class StateMachine:
    """
    전환 규칙:
      - 동일 감정이 CONFIRM_DURATION(3초) 이상 지속 → 상태 전환
      - confidence < MIN_CONFIDENCE(0.5) → neutral 유지
      - 얼굴 미감지 FACE_TIMEOUT(5초) 이상 → neutral 강제 전환

    전환 쿨다운:
      - 같은 상태로 재진입 시 COOLDOWN(10초) 동안 재트리거 방지
      - (동일 반응 반복으로 로봇이 "과반응"하는 것 방지)
    """

    CONFIRM_DURATION = 3.0    # 초
    MIN_CONFIDENCE   = 0.5
    FACE_TIMEOUT     = 5.0    # 초
    COOLDOWN         = 10.0   # 초

    def tick(self, fused: FusedEmotion, dt: float) -> StateTransition:
        """
        100ms 주기로 호출.
        Returns: StateTransition(from_state, to_state, triggered)
        triggered=True 일 때만 OutputManager가 반응 실행
        """
```

---

### 3.4 MemoryManager

**역할:** 감정 로그 저장, 시간대별 패턴 분석, 개인화 반응 강도 조정

#### 스키마

```sql
-- 감정 로그
CREATE TABLE emotion_log (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   REAL NOT NULL,          -- Unix timestamp
    emotion     TEXT NOT NULL,
    confidence  REAL NOT NULL,
    session_id  TEXT NOT NULL,
    hour_of_day INTEGER NOT NULL        -- 0~23, 패턴 분석용
);

-- 사용자 패턴 (집계 테이블)
CREATE TABLE emotion_pattern (
    hour_of_day INTEGER NOT NULL,
    emotion     TEXT NOT NULL,
    count       INTEGER DEFAULT 0,
    avg_conf    REAL DEFAULT 0.0,
    last_updated REAL,
    PRIMARY KEY (hour_of_day, emotion)
);
```

```python
class MemoryManager:
    """
    SQLite 기반 감정 기록 및 패턴 조회.
    DB 파일: data/emotion_memory.db
    """

    LOG_INTERVAL = 5.0   # 초: 5초마다 현재 감정 기록

    def log_emotion(self, emotion: str, confidence: float, session_id: str) -> None:
        """emotion_log에 기록 + emotion_pattern 업데이트"""

    def get_intensity_multiplier(self, emotion: str, hour: int) -> float:
        """
        패턴 분석 기반 반응 강도 배수 반환.
        예: 저녁 9시에 sad가 자주 → 위로 반응 강도 1.5x
        반환 범위: 0.5 ~ 2.0
        """

    def end_session(self, session_id: str) -> None:
        """세션 종료 시 패턴 집계 업데이트"""
```

---

### 3.5 OutputManager

**역할:** 상태 전환을 받아 OLED, 서보, 사운드를 동시에 제어

```python
class OutputManager:
    """
    세 가지 출력 장치를 상태별로 동시 제어.
    각 장치는 비동기로 실행 (스레드 풀).
    """

    def execute(self, state: RobotState, intensity: float = 1.0) -> None:
        """
        state에 따라 세 출력을 동시 실행.
        intensity: MemoryManager에서 받은 강도 배수 (0.5~2.0)
        """
        with ThreadPoolExecutor(max_workers=3) as pool:
            pool.submit(self.eyes.set_expression, state, intensity)
            pool.submit(self.servos.move_to, state, intensity)
            pool.submit(self.sound.play, state, intensity)
```

#### 3.5.1 EyeController

```python
class EyeController:
    """
    SSD1306 OLED 두 개 (왼쪽/오른쪽) 동시 제어.
    노트북 개발 시: pygame 창으로 시뮬레이션.
    """

    EYE_SHAPES = {
        RobotState.NEUTRAL: "circle",        # ●
        RobotState.HAPPY:   "crescent",      # ◡ (초승달)
        RobotState.SAD:     "droopy",        # 처진 눈 + 눈물
        RobotState.ANGRY:   "frown",         # 찡그린 눈
        RobotState.FEAR:    "wide",          # 크게 뜬 눈
    }

    BLINK_INTERVAL = 3.0  # 초: 일반 깜빡임 주기
    COMFORT_BLINK  = 1.5  # 초: 위로 모드 부드러운 깜빡임

    def set_expression(self, state: RobotState, intensity: float) -> None:
        """
        양쪽 OLED에 동일 표정 렌더링.
        I2C 주소: 왼쪽=0x3C, 오른쪽=0x3D
        """

    def draw_eye(self, display, shape: str, intensity: float) -> None:
        """PIL.ImageDraw로 눈 모양 그리기 → OLED 전송"""
```

#### 3.5.2 ServoController

```python
class ServoController:
    """
    PCA9685 I2C 서보 드라이버로 MG996R 서보 제어.
    채널 0: 머리 좌우 (yaw)
    채널 1: 머리 상하 (pitch)
    채널 2: 몸통 기울기 (roll, 선택)

    노트북 개발 시: 콘솔 출력으로 시뮬레이션.
    """

    # 서보 펄스 범위 (PCA9685 기준)
    SERVO_MIN = 150   # ≈ 0도
    SERVO_MID = 375   # ≈ 90도 (중립)
    SERVO_MAX = 600   # ≈ 180도

    # 상태별 목표 자세 (yaw_deg, pitch_deg)
    POSES = {
        RobotState.NEUTRAL: (90, 90),    # 중립, 좌우 스캔
        RobotState.HAPPY:   (90, 80),    # 살짝 앞으로
        RobotState.SAD:     (90, 70),    # 앞으로 기울기 (다가오기)
        RobotState.ANGRY:   (90, 100),   # 뒤로 살짝 물러서기
        RobotState.FEAR:    (90, 85),    # 중립 + 느린 이동
    }

    def move_to(self, state: RobotState, intensity: float) -> None:
        """목표 자세로 부드럽게 이동 (선형 보간, 500ms)"""

    def idle_scan(self) -> None:
        """NEUTRAL 상태의 좌우 스캔 주기 동작"""

    def angle_to_pulse(self, angle: int) -> int:
        """도(°) → PCA9685 펄스값 변환"""
```

#### 3.5.3 SoundManager

```python
class SoundManager:
    """
    pygame.mixer로 wav 파일 재생.
    사운드 파일: assets/sounds/{state}/{variant}.wav
    각 상태별 3개 이상의 변형 (반복 느낌 방지를 위해 랜덤 선택).
    """

    SOUND_PATHS = {
        RobotState.NEUTRAL: "assets/sounds/idle/",
        RobotState.HAPPY:   "assets/sounds/happy/",
        RobotState.SAD:     "assets/sounds/comfort/",
        RobotState.ANGRY:   "assets/sounds/calm/",
        RobotState.FEAR:    "assets/sounds/reassure/",
    }

    def play(self, state: RobotState, intensity: float) -> None:
        """
        상태에 맞는 사운드 랜덤 선택 재생.
        intensity에 따라 볼륨 조절 (0.3 ~ 1.0).
        이미 재생 중이면 페이드아웃 후 재생.
        """
```

---

### 3.6 PersonalityEngine

**역할:** MemoryManager의 패턴 데이터로 반응 강도를 개인화

```python
class PersonalityEngine:
    """
    시간대별 감정 패턴을 반영해 출력 강도 조정.
    예: 저녁 9시 sad 패턴이 높으면 → 해당 시간대 sad 반응을 더 강하게.
    """

    def get_intensity(self, emotion: str, memory: MemoryManager) -> float:
        """
        현재 시간 + 감정 → intensity multiplier (0.5 ~ 2.0).
        학습 데이터 없으면 기본값 1.0.
        """
```

---

## 4. 데이터 구조

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class EmotionResult:
    emotion: str          # "happy" | "sad" | "angry" | "neutral" | "fear" | "surprise"
    confidence: float     # 0.0 ~ 1.0
    face_detected: bool

@dataclass
class GestureResult:
    gesture: str          # "head_down" | "hunched" | "lean_forward" | "arms_crossed" | "covering_face" | "neutral_pose"
    sentiment: str        # "positive" | "negative" | "neutral"
    confidence: float     # 0.0 ~ 1.0 (MediaPipe landmark visibility 기반)
    body_detected: bool   # False 시 EmotionFusion에서 gesture 채널 제외

@dataclass
class SentimentResult:
    sentiment: str        # "positive" | "negative" | "neutral"
    matched_keywords: list[str]

@dataclass
class FusedEmotion:
    emotion: str
    confidence: float
    source: str           # "face_only" | "gesture_only" | "text_only" | "fused"

@dataclass
class StateTransition:
    from_state: RobotState
    to_state: RobotState
    triggered: bool       # True 일 때만 OutputManager 호출

# 공유 상태 (스레드 간 공유)
class SharedState:
    def __init__(self):
        self.lock = threading.Lock()
        self.visual_emotion: EmotionResult = EmotionResult("neutral", 0.0, False)
        self.gesture_emotion: GestureResult = GestureResult("neutral_pose", "neutral", 0.0, False)
        self.text_emotion: SentimentResult = SentimentResult("neutral", [])
        self.last_face_time: float = time.time()
        self.last_body_time: float = time.time()

    def update_visual(self, result: EmotionResult) -> None:
        with self.lock:
            self.visual_emotion = result
            if result.face_detected:
                self.last_face_time = time.time()

    def update_gesture(self, result: GestureResult) -> None:
        with self.lock:
            self.gesture_emotion = result
            if result.body_detected:
                self.last_body_time = time.time()

    def update_text(self, result: SentimentResult) -> None:
        with self.lock:
            self.text_emotion = result

    def snapshot(self) -> tuple[EmotionResult, GestureResult, SentimentResult, float, float]:
        with self.lock:
            return (self.visual_emotion, self.gesture_emotion,
                    self.text_emotion, self.last_face_time, self.last_body_time)
```

---

## 5. 스레드 모델

| 스레드 | 주기 | 우선순위 | 설명 |
|--------|------|---------|------|
| CameraWorker | 33ms (30fps) | Low | 프레임 캡처만 |
| VideoAnalyzer | 가변 | Low | DeepFace 3fps + MediaPipe 15fps (프레임 카운터 분기) |
| AudioWorker | Event-driven | Medium | VAD 감지 시 활성화 |
| MainLoop | 100ms | High | 상태 갱신 + 출력 |

**VideoAnalyzer 처리 시간 분석 (RPi 5 기준):**

```
frame_count % 10 == 0 (DeepFace 프레임):
  DeepFace ~300ms + MediaPipe ~50ms = ~350ms 처리
  → 다음 분석까지 10 × 33ms = 330ms 여유
  → 약간 초과하나 frame_queue 드롭으로 자연 조절됨

그 외 프레임 (MediaPipe만):
  MediaPipe ~50ms = 실시간 처리 가능
```

**경쟁 조건 (race condition) 방지:**
- 모든 SharedState 접근은 `threading.Lock()` 사용
- `Queue(maxsize=3)`: 프레임 큐 포화 시 오래된 프레임 자동 드롭 (`get_nowait` + `put_nowait`)
- OutputManager는 `ThreadPoolExecutor`로 비동기 실행 후 join 대기

---

## 6. 설정 파일

`config/config.yaml`:

```yaml
camera:
  device_id: 0             # USB 웹캠 인덱스 (picamera2 사용 시 무시됨)
  fps: 30                  # 30fps 캡처 (변경: 10 → 30)
  use_picamera2: true      # RPi Camera Module 사용 시 true

emotion:
  confirm_duration: 3.0    # 상태 전환 요구 지속 시간(초)
  min_confidence: 0.5
  face_timeout: 5.0        # 얼굴 미감지 후 neutral 전환(초)
  body_timeout: 10.0       # 신체 미감지 후 gesture 채널 비활성화(초)
  cooldown: 10.0           # 동일 상태 재트리거 금지(초)
  face_weight: 0.5         # 변경: visual_weight 0.7 → face_weight 0.5
  gesture_weight: 0.2      # 신규: MediaPipe Pose 제스처 채널
  text_weight: 0.3         # 유지
  face_skip_frames: 10     # DeepFace 분석: 30fps ÷ 10 = 3fps (변경: skip_frames 3 → 10)
  gesture_skip_frames: 2   # MediaPipe 분석: 30fps ÷ 2 = 15fps (신규)

audio:
  vad_speech_threshold: 0.5
  vad_silence_threshold: 1.5
  whisper_model: "tiny"    # tiny | base | small
  language: "ko"

output:
  volume: 0.7
  servo_speed: 500         # ms: 자세 전환 시간

memory:
  log_interval: 5.0        # 초
  db_path: "data/emotion_memory.db"

hardware:
  mock_mode: true          # 노트북에서는 true, RPi에서는 false
  i2c_bus: 1
  eye_left_addr: 0x3C
  eye_right_addr: 0x3D
  pca9685_addr: 0x40
```

---

## 7. 에러 처리 전략

| 실패 시나리오 | 처리 방법 |
|--------------|---------|
| 카메라 연결 실패 | 경고 로그 + 오디오 전용 모드로 전환 |
| DeepFace 추론 에러 | `enforce_detection=False` → neutral 반환 |
| MediaPipe body_detected=False | gesture 채널 비활성화, face/text 2채널 퓨전으로 자동 전환 |
| MediaPipe 초기화 실패 | 경고 로그, face+text 2채널 모드로 강등 |
| Whisper STT 실패 | 예외 무시, text_emotion은 neutral 유지 |
| 얼굴 5초 이상 미감지 | neutral 강제 전환 + idle 스캔 |
| 신체 10초 이상 미감지 | gesture 채널 가중치 0으로 설정, face/text 재배분 |
| OLED I2C 오류 | 경고 로그, 서보/사운드는 계속 동작 |
| PCA9685 오류 | 경고 로그, 눈/사운드는 계속 동작 |
| 오디오 파일 없음 | 경고 로그, 무음으로 진행 |
| 신뢰도 < 0.5 | neutral 유지, 반응 없음 |

**Fallback 계층:**
```
정상 → [DeepFace + Whisper + OLED + Servo + Sound]
  ↓ 카메라 실패
오디오 모드 → [Whisper + OLED + Servo + Sound]
  ↓ OLED 실패
무표정 모드 → [DeepFace + Whisper + Servo + Sound]
  ↓ 전체 하드웨어 실패
노트북 시뮬레이션 → pygame 창 + 콘솔 출력
```

---

## 8. 테스트 계획

### 8.1 단위 테스트 (tests/unit/)

```
tests/unit/
├── test_emotion_fusion.py
│   ├── [★★★] visual=sad + text=negative → fused=sad, 높은 confidence
│   ├── [★★★] visual=happy + text=negative → fused=neutral (상충)
│   ├── [★★★] visual=neutral + text=negative → fused=sad (텍스트 보완)
│   └── [★★★] confidence < 0.5 → neutral 강제
├── test_state_machine.py
│   ├── [★★★] 3초 미만 감정 지속 → 상태 전환 없음
│   ├── [★★★] 3초 이상 sad → SAD 상태 전환
│   ├── [★★★] cooldown 10초 내 동일 감정 → 재트리거 없음
│   ├── [★★★] 얼굴 미감지 5초 → NEUTRAL 강제 전환
│   └── [★★  ] 상태 전환 시 StateTransition.triggered=True
├── test_audio_worker.py
│   ├── [★★★] 부정 키워드 → sentiment=negative
│   ├── [★★★] 긍정 키워드 → sentiment=positive
│   └── [★★  ] 키워드 없음 → sentiment=neutral
└── test_memory_manager.py
    ├── [★★★] 감정 로그 저장 → DB 레코드 생성
    ├── [★★  ] 패턴 없음 → intensity_multiplier=1.0
    └── [★★  ] 반복 패턴 → intensity_multiplier > 1.0
```

### 8.2 통합 테스트 (tests/integration/)

```
tests/integration/
├── test_pipeline.py
│   ├── [★★★] [→E2E] 전체 루프: 가짜 카메라 프레임 → 상태 전환 → 출력 명령
│   ├── [★★  ] mock_mode=True 에서 서보/OLED mock 호출 확인
│   └── [★★  ] 5초 무얼굴 → NEUTRAL 전환
└── test_concurrency.py
    ├── [★★★] 다중 스레드 SharedState 동시 접근 → 데이터 손상 없음
    └── [★★  ] 프레임 큐 포화 → 오래된 프레임 드롭, 블로킹 없음
```

### 8.3 수동 데모 테스트 체크리스트

```
[ ] 슬픈 표정 3초 → 눈이 처지고, 멜로디 재생, 몸이 앞으로 기울어짐
[ ] "힘들어" 발화 → sad 반응 (표정 neutral이어도)
[ ] 기쁜 표정 3초 → 초승달 눈, 경쾌한 소리, 살짝 앞으로
[ ] 아무도 없을 때 → 5초 후 idle 스캔 + 중립 눈
[ ] 조명 어두운 환경 → graceful degradation (에러 없이 neutral)
[ ] 시끄러운 환경 → STT 실패해도 시각 감정으로만 동작
[ ] 전체 응답 지연 측정 → 5초 이내
```

---

## 9. 개발 환경 설정

### 9.1 필수 의존성

```bash
# requirements.txt
deepface==0.0.93
opencv-python==4.9.0.80
mediapipe==0.10.14         # 신규: MediaPipe Pose 제스처 감지 (CPU 최적화)
faster-whisper==1.0.3
silero-vad==5.1
pygame==2.5.2
PyAudio==0.2.14
Pillow==10.3.0
luma.oled==3.13.0          # SSD1306 OLED 드라이버
adafruit-circuitpython-pca9685==3.4.7  # 서보 드라이버 (RPi만)
RPi.GPIO==0.7.1            # RPi만
picamera2==0.3.19          # RPi Camera Module 3 드라이버 (RPi만)
pyyaml==6.0.1
```

### 9.2 노트북 개발 환경

```bash
# 1. 가상환경 생성
python3.11 -m venv .venv
source .venv/bin/activate

# 2. 기본 의존성 설치 (RPi 전용 제외)
pip install deepface opencv-python faster-whisper silero-vad pygame PyAudio Pillow pyyaml

# 3. 개발 모드 실행 (mock_mode: true)
python main.py --config config/config.yaml

# 4. 단위 테스트
pytest tests/unit/ -v

# 5. 감정 감지 단독 테스트
python -m tools.test_emotion_detect
```

### 9.3 RPi 환경 설정

```bash
# RPi OS 설정
sudo raspi-config
# → Interface Options → I2C → Enable
# → Interface Options → Camera → Enable

# cmake (dlib 컴파일용)
sudo apt install -y cmake build-essential libopenblas-dev

# 의존성 설치 (1~2시간 소요: dlib ARM 컴파일)
pip install dlib face-recognition

# 대안: dlib 컴파일 없이 InsightFace 사용
pip install insightface

# I2C 연결 확인
i2cdetect -y 1
# 출력에 0x3C, 0x3D(OLED), 0x40(PCA9685) 확인

# RPi 모드 실행
python main.py --config config/config_rpi.yaml
```

---

## 10. 파일 구조

```
companion_robot/
├── main.py                    # 진입점: 스레드 시작, 메인 루프
├── config/
│   ├── config.yaml            # 노트북 개발용 (mock_mode: true)
│   └── config_rpi.yaml        # RPi 배포용 (mock_mode: false)
├── src/
│   ├── __init__.py
│   ├── input/
│   │   ├── camera_capture.py  # CameraCapture 클래스 (30fps)
│   │   ├── video_analyzer.py  # VideoAnalyzer (DeepFace + MediaPipe, 통합)
│   │   ├── gesture_detector.py# GestureDetector (MediaPipe Pose)
│   │   └── audio_worker.py    # AudioWorker (VAD + Whisper)
│   ├── processing/
│   │   ├── emotion_fusion.py  # EmotionFusion
│   │   ├── state_machine.py   # StateMachine + RobotState
│   │   └── shared_state.py    # SharedState (스레드 안전 공유)
│   ├── memory/
│   │   ├── memory_manager.py  # MemoryManager (SQLite)
│   │   └── personality_engine.py  # PersonalityEngine
│   └── output/
│       ├── output_manager.py  # OutputManager (조합)
│       ├── eye_controller.py  # EyeController (OLED / pygame mock)
│       ├── servo_controller.py# ServoController (PCA9685 / mock)
│       └── sound_manager.py   # SoundManager (pygame.mixer)
├── assets/
│   └── sounds/
│       ├── idle/              # neutral 상태 주변음
│       ├── happy/             # 기쁨 효과음 × 3+
│       ├── comfort/           # 위로 멜로디 × 3+
│       ├── calm/              # 진정 저주파 × 3+
│       └── reassure/          # 안심 효과음 × 3+
├── data/
│   └── emotion_memory.db      # SQLite (gitignore)
├── tests/
│   ├── unit/
│   └── integration/
├── tools/
│   └── test_emotion_detect.py # 단독 감정 감지 테스트 도구
├── requirements.txt
├── requirements_rpi.txt       # RPi 추가 의존성
└── README.md
```

---

## 11. 개발 단계별 구현 순서

### Phase 1 (Week 1~2) — 소프트웨어 코어

```
Day 1-2:  CameraCapture (30fps) + DeepFace 얼굴 감지 + SharedState
          → python tools/test_emotion_detect.py 로 감정 감지 확인

Day 3:    GestureDetector (MediaPipe Pose) 구현
          → 고개 숙임, 몸 기울기, 팔짱 제스처 감지 확인
          → VideoAnalyzer에 DeepFace + MediaPipe 통합 (프레임 카운터 분기)

Day 4:    AudioWorker (VAD + Whisper + 키워드 분석)
          → 터미널에서 "힘들어" 말하면 negative 출력 확인

Day 5:    EmotionFusion (3채널) + StateMachine
          → face=0.5, gesture=0.2, text=0.3 퓨전 결과 단위 테스트
          → body_detected=False 시 2채널 자동 전환 확인

Day 6:    SoundManager + EyeController (pygame mock)
          → 감정별 소리 + 화면 눈 표정 확인

Day 7:    통합 + 첫 완전 데모
          → 슬픈 표정 + 고개 숙임 → SAD 강한 반응
          → 표정 없어도 고개 숙임만으로 SAD 트리거 확인

Day 8-10: MemoryManager (SQLite 로그 + 패턴)
          → DB에 감정 기록, 시간대별 intensity 조회 확인

Day 11-12: PersonalityEngine
           → 반복 패턴 학습 후 intensity multiplier 변화 확인

Day 13-14: Phase 1 완전 데모 + 통합 테스트 실행
```

### Phase 2 (Week 3~4) — RPi 이식

```
Day 15:   RPi OS + SSH + 카메라/마이크/스피커 연결 확인
Day 16-17: pip install (ARM 의존성) + 코드 이식
Day 18-19: ServoController (PCA9685) 연결 + 동작 테스트
Day 20-21: EyeController (SSD1306 OLED) 연결 + 표정 테스트
Day 22-24: 전체 통합 (mock_mode: false)
Day 27-28: Phase 2 완전 데모 (RPi 엔드투엔드)
```

### Phase 3 (Week 5~6) — 품질 향상

```
Week 5: 개인화 강화 + 행동 다양화 (상태별 3가지 이상 행동 변형)
Week 6: 몸체 완성 + 사운드 라이브러리 확장 + OLED 표정 세밀화
```

### Phase 4 (Week 7~8) — 완성

```
Week 7: 안정성 테스트 (조명/소음/장시간 실행 엣지 케이스)
Week 8: 데모 리허설 × 5 + README 정리 + 경진대회 제출
```

---

*이 문서는 소프트웨어 명세서입니다. 하드웨어 명세는 `hardware_spec.md`를 참조하세요.*
