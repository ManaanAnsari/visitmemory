# Phase 2: Edge Agent Capture Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Windows-first edge agent that activates with a distributor code and activation code, samples a CCTV source at 2 FPS, detects and quality-gates faces, selects a best shot, deduplicates locally, spools uploads durably, and ships as a single executable.

**Architecture:** A pure-Python pipeline behind small Protocols: `FrameSource`, `FaceDetector`, `SecretStore`, `Spool`, and `CloudClient`. Production adapters are OpenCV/RTSP, ONNX Runtime SCRFD, and HTTP; tests use synthetic frames, a fake detector, a temporary secret store, and SQLite. The pipeline never blocks capture on network I/O: qualified crops go to a SQLite spool drained by a separate worker.

**Tech Stack:** Python 3.12, OpenCV (`opencv-python-headless` on the agent), ONNX Runtime, NumPy, httpx, SQLite (stdlib), PyInstaller, pytest.

**Spec:** `docs/tech.md` §1-§2, §5, §6, §10; `plans/00-roadmap.md`.

## Global Constraints

- Python 3.12, Windows 10+ target, CPU-only inference.
- Exact dependency pins only (`==`); no unpinned model downloads at runtime.
- Sample rate is 2 FPS. Detection runs only on sampled frames.
- Resolution gate: bounding box at least `80 x 80` pixels.
- Pose gate: yaw ratio at most `2.2`.
- Sharpness gate: Laplacian variance at least `80.0`.
- Best-shot window: `1.5` seconds per track.
- Local dedup window: `15` minutes.
- Retry delays: `5, 15, 60, 300` seconds, then every `300` seconds up to 24 hours, then dead-letter.
- Raw video is never written to disk or uploaded. Only the selected JPEG crop leaves the machine.
- The API key is stored through a `SecretStore`; it never appears in logs, exception text, or crash reports.
- The agent must keep capturing when the network is down and must not lose spooled items across restarts.
- Model files are bundled and verified by SHA-256 at load time.

## File Structure

```
edge/
  requirements.txt
  requirements-dev.txt
  pyproject.toml
  agent/
    __init__.py
    config.py                  AgentConfig load/save + default paths
    secrets.py                 SecretStore Protocol, FileSecretStore, WindowsDpapiSecretStore
    api_client.py              CloudClient Protocol + HttpCloudClient
    activation.py              activate_agent(config, distributor_code, activation_code)
    sampler.py                 FrameSampler
    pipeline.py                AgentPipeline
    worker.py                  spool drain loop
    spool.py                   SqliteSpool
    dedup.py                   LocalDedupCache (pHash)
    models.py                  Detection, GateResult, TrackedFace, VisitPayload
    capture/
      base.py                  FrameSource Protocol
      synthetic.py             SyntheticFrameSource, VideoFileSource
      rtsp.py                  RtspFrameSource
      windows_window.py        WindowsWindowSource (Windows only)
    detect/
      base.py                  FaceDetector Protocol, FakeDetector
      scrfd.py                 ScrfdDetector (ONNX)
      quality.py               size/pose/sharpness gates
      tracker.py               BestShotBuffer
    cli.py                     activate, run, status commands
  tests/
    conftest.py
    test_config.py
    test_secrets.py
    test_api_client.py
    test_activation.py
    test_sampler.py
    test_quality.py
    test_best_shot.py
    test_spool.py
    test_dedup.py
    test_pipeline.py
    test_cli.py
    test_windows_adapters.py
  packaging/
    visitmemory-agent.spec
    build_windows.ps1
    models/README.md
    models/scrfd_500m.sha256
```

## Interfaces

- `FrameSource.open() -> None`, `FrameSource.read() -> np.ndarray | None`, `FrameSource.close() -> None`, `FrameSource.fps: float`
- `FaceDetector.detect(frame: np.ndarray) -> list[Detection]`
- `Detection(bbox: tuple[int, int, int, int], landmarks: np.ndarray, confidence: float, track_id: int | None = None)`
- `SimpleTracker.update(detections: list[Detection], now: datetime) -> list[Detection]` assigns stable `track_id`s by IoU
- `best_shot_buffer.add(track_id: int, crop: np.ndarray, sharpness: float, now: datetime) -> None`
- `best_shot_buffer.pop_ready(now: datetime) -> list[TrackedFace]`
- `quality_gate(frame: np.ndarray, detection: Detection) -> GateResult`
- `LocalDedupCache.seen(crop: np.ndarray, now: datetime) -> bool`; `remember(crop, now) -> None`
- `Spool.enqueue(crop_jpeg: bytes, meta: dict) -> str`; `Spool.due(now, limit) -> list[SpoolItem]`; `Spool.mark_sent(item_id)`; `Spool.mark_failed(item_id, now)`
- `CloudClient.activate(req: ActivationRequest) -> ActivationResponse`
- `CloudClient.upload_visit(api_key: str, crop_jpeg: bytes, meta: VisitMeta) -> VisitUploadResponse`
- `CloudClient.heartbeat(api_key: str) -> HeartbeatResponse`
- `AgentConfig(api_base_url, api_key, shop_id, installation_id, source_type, source_target, sample_fps, camera_role, model_path, spool_path)`
- `VisitMeta(client_event_id: str, captured_at: datetime, camera_role: Literal["ENTRANCE", "COUNTER"], agent_version: str)`

## Review Focus

1. **Spool loss or duplication.** Process kill during upload, network timeout after the server accepted, or a corrupt SQLite file causes a lost visit or a duplicate alert. Expected: item remains queued until acknowledged, and retries are idempotent.
2. **Clock and timezone drift.** The store PC clock is wrong or in local time, so receipt correlation later fails. Expected: timestamps sent as UTC with an explicit offset, and the server trusts its own receipt time for correlation.
3. **Bad capture accepted.** A backlit or blurred frame passes the gates and reaches recognition. Expected: the gates reject it using the spec thresholds.
4. **API key exposure.** The key appears in a log line, a traceback, or the spool database. Expected: only the secret store holds it; logs redact it.
5. **Model tampering.** A replaced ONNX file runs arbitrary inference content or degrades accuracy silently. Expected: SHA-256 mismatch refuses startup.

---

### Task 1: Edge package, configuration, and secret storage

**Files:**
- Create: `edge/requirements.txt`
- Create: `edge/requirements-dev.txt`
- Create: `edge/pyproject.toml`
- Create: `edge/agent/__init__.py`
- Create: `edge/agent/config.py`
- Create: `edge/agent/secrets.py`
- Create: `edge/agent/models.py`
- Create: `edge/tests/conftest.py`
- Create: `edge/tests/test_config.py`
- Create: `edge/tests/test_secrets.py`

**Interfaces:**
- Produces:
  - `default_config_dir() -> Path` (`%ProgramData%\VisitMemory` on Windows, `~/.config/visitmemory` elsewhere)
  - `AgentConfig.load(path: Path) -> AgentConfig`, `AgentConfig.save(path: Path) -> None`
  - `SecretStore.set(name: str, value: str) -> None`, `SecretStore.get(name: str) -> str | None`, `SecretStore.delete(name: str) -> None`
  - `FileSecretStore(path: Path)` with `0600` permissions and atomic replace
  - `WindowsDpapiSecretStore(path: Path)` using `CryptProtectData`/`CryptUnprotectData`

- [ ] **Step 1: Pin dependencies**

`edge/requirements.txt` pins `numpy`, `opencv-python-headless`, `onnxruntime`, `httpx`, `pydantic`, and `pydantic-settings` exactly. `edge/requirements-dev.txt` adds `pytest`, `pytest-asyncio`, and `ruff`.

- [ ] **Step 2: Write the failing tests**

`edge/tests/test_config.py`:
```python
def test_config_roundtrip(tmp_path):
    config = AgentConfig(
        api_base_url="https://api.example.test",
        api_key=None,
        shop_id=None,
        installation_id=None,
        source_type="video_file",
        source_target="/tmp/sample.mp4",
        sample_fps=2.0,
        camera_role="ENTRANCE",
        model_path=tmp_path / "scrfd.onnx",
        spool_path=tmp_path / "spool.sqlite3",
    )
    path = tmp_path / "config.json"
    config.save(path)
    assert AgentConfig.load(path) == config


def test_config_rejects_unknown_camera_role(tmp_path):
    with pytest.raises(ValidationError):
        AgentConfig(..., camera_role="STOCKROOM")
```

`edge/tests/test_secrets.py`:
```python
def test_file_secret_store_roundtrip(tmp_path):
    store = FileSecretStore(tmp_path / "secret.json")
    store.set("api_key", "vm_live_example")
    assert store.get("api_key") == "vm_live_example"
    assert store.get("missing") is None


def test_file_secret_store_permissions(tmp_path):
    path = tmp_path / "secret.json"
    FileSecretStore(path).set("api_key", "vm_live_example")
    assert (path.stat().st_mode & 0o077) == 0
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd edge && pytest tests/test_config.py tests/test_secrets.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 4: Implement config, models, and secret stores**

`models.py` defines `Detection`, `GateResult`, `TrackedFace`, `VisitMeta`, `ActivationRequest`, `ActivationResponse`, `VisitUploadResponse`, and `HeartbeatResponse` as dataclasses or Pydantic models. `VisitMeta` contains `client_event_id: str`, `captured_at: datetime`, `camera_role: Literal["ENTRANCE", "COUNTER"]`, and `agent_version: str`. `ActivationRequest` contains `installation_kind: Literal["CAMERA", "POS_HELPER"] = "CAMERA"`.

`WindowsDpapiSecretStore` is imported lazily; on non-Windows it raises `RuntimeError("DPAPI is only available on Windows")`. Its tests are marked `windows`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd edge && pytest tests/test_config.py tests/test_secrets.py -v`
Expected: PASS (4 tests, 1 skipped on non-Windows if the DPAPI test is present).

- [ ] **Step 6: Commit**

```bash
git add edge
git commit -m "feat: scaffold edge agent config and secret storage"
```

---

### Task 2: Cloud client and activation command

**Files:**
- Create: `edge/agent/api_client.py`
- Create: `edge/agent/activation.py`
- Create: `edge/agent/cli.py`
- Create: `edge/tests/test_api_client.py`
- Create: `edge/tests/test_activation.py`
- Create: `edge/tests/test_cli.py`

**Interfaces:**
- Consumes: `AgentConfig`, `SecretStore`, `ActivationRequest`.
- Produces:
  - `CloudClient` Protocol with `activate`, `upload_visit`, `heartbeat`
  - `HttpCloudClient(base_url: str, client: httpx.Client, timeout_seconds: float = 10.0)`
  - `activate_agent(config: AgentConfig, store: SecretStore, cloud: CloudClient, distributor_code: str, activation_code: str, machine_fingerprint: str, agent_version: str) -> ActivationResponse`
  - CLI: `python -m agent.cli activate --config PATH --distributor-code CODE`, `python -m agent.cli status --config PATH`

- [ ] **Step 1: Write the failing tests**

`edge/tests/test_api_client.py` uses `httpx.MockTransport`:
```python
def test_upload_visit_sends_bearer_key_and_multipart():
    ...
    assert request.headers["Authorization"] == "Bearer vm_live_test"
    assert b'name="image"' in request.read()


def test_upload_visit_retries_on_503_then_succeeds():
    ...


def test_client_does_not_log_api_key(caplog):
    ...
```

`edge/tests/test_activation.py`:
```python
def test_activation_persists_key_and_identifiers(tmp_path):
    ...
    assert config.api_key == "vm_live_new"
    assert store.get("api_key") == "vm_live_new"


def test_activation_failure_leaves_config_unchanged(tmp_path):
    ...
```

`edge/tests/test_cli.py` covers prompt-driven activation without echoing the code, and `status` reporting configured/missing without printing the key.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd edge && pytest tests/test_api_client.py tests/test_activation.py tests/test_cli.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement the client and activation**

`HttpCloudClient` calls:
- `POST /api/v1/installations/activate` with JSON `{distributor_code, activation_code, machine_fingerprint, agent_version, installation_kind}` where the agent sends `"CAMERA"`
- `GET /api/v1/installations/me` with bearer auth
- `POST /api/v1/visits` multipart with `image` plus form fields `client_event_id`, `captured_at` (ISO 8601 with offset), `camera_role`, `agent_version`

Retry only on connection errors, `429`, and `5xx`, with delays `1, 2, 4` seconds and a maximum of 3 attempts. Do not retry `4xx` other than `429`. Redact `Authorization` in any exception message.

`machine_fingerprint` is `sha256(platform.node() + platform.machine() + machine_id)` where `machine_id` is the Windows `MachineGuid` when available, otherwise the hostname. Hash the result before sending.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd edge && pytest tests/test_api_client.py tests/test_activation.py tests/test_cli.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add edge/agent edge/tests
git commit -m "feat: add edge cloud client and activation command"
```

---

### Task 3: Frame sources and 2 FPS sampler

**Files:**
- Create: `edge/agent/capture/base.py`
- Create: `edge/agent/capture/synthetic.py`
- Create: `edge/agent/capture/rtsp.py`
- Create: `edge/agent/sampler.py`
- Create: `edge/tests/test_sampler.py`

**Interfaces:**
- Consumes: `FrameSource`.
- Produces:
  - `SyntheticFrameSource(frames: list[np.ndarray], fps: float)`
  - `VideoFileSource(path: Path, fps: float)`
  - `RtspFrameSource(url: str, fps: float)`
  - `FrameSampler(source: FrameSource, target_fps: float = 2.0)`
  - `FrameSampler.frames(now: Callable[[], float]) -> Iterator[np.ndarray]`

- [ ] **Step 1: Write the failing test**

```python
def test_sampler_emits_two_frames_per_second():
    source = SyntheticFrameSource(frames=[np.zeros((480, 640, 3), dtype=np.uint8)] * 20, fps=30.0)
    sampler = FrameSampler(source, target_fps=2.0)
    emitted = list(sampler.frames(now=iter([0.0, 0.1, 0.5, 0.9, 1.0, 1.1]).__next__))
    assert len(emitted) == 2


def test_sampler_skips_until_next_interval():
    ...
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd edge && pytest tests/test_sampler.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement sources and sampler**

`SyntheticFrameSource` cycles deterministic frames and is test-only. `VideoFileSource` uses `cv2.VideoCapture`; `RtspFrameSource` sets `CAP_PROP_BUFFERSIZE=1` and reconnects after a read failure with a 5-second delay. The sampler keeps a `next_due` monotonic timestamp and yields at most one frame per due interval; it never sleeps inside the pipeline beyond the source read.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd edge && pytest tests/test_sampler.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add edge/agent/capture edge/agent/sampler.py edge/tests/test_sampler.py
git commit -m "feat: add frame sources and 2 FPS sampler"
```

---

### Task 4: Face detector protocol, fake detector, and SCRFD adapter

**Files:**
- Create: `edge/agent/detect/base.py`
- Create: `edge/agent/detect/scrfd.py`
- Create: `edge/packaging/models/README.md`
- Create: `edge/packaging/models/scrfd_500m.sha256`
- Create: `edge/tests/test_detector.py`

**Interfaces:**
- Consumes: `Detection`, `np.ndarray`.
- Produces:
  - `FaceDetector` Protocol
  - `FakeDetector(frames_to_detections: list[list[Detection]])`
  - `ScrfdDetector(model_path: Path, expected_sha256: str, confidence_threshold: float = 0.75)`
  - `ScrfdDetector.detect(frame: np.ndarray) -> list[Detection]`

- [ ] **Step 1: Write the failing tests**

```python
def test_detector_rejects_modified_model(tmp_path):
    model = tmp_path / "scrfd.onnx"
    model.write_bytes(b"not a real model")
    with pytest.raises(ModelIntegrityError):
        ScrfdDetector(model_path=model, expected_sha256="0" * 64)


def test_fake_detector_returns_configured_detections():
    detector = FakeDetector([[Detection(bbox=(10, 10, 120, 120), landmarks=..., confidence=0.9)]])
    assert len(detector.detect(np.zeros((480, 640, 3), dtype=np.uint8))) == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd edge && pytest tests/test_detector.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement detector**

`ScrfdDetector` verifies SHA-256 before creating an `onnxruntime.InferenceSession` with `CPUExecutionProvider`. Preprocess per SCRFD: letterbox resize, BGR to RGB, normalize, NCHW float32. Decode boxes and 5 landmarks; apply NMS with IoU 0.4. Return `Detection` objects with `track_id=None`. The model file is not committed; `models/README.md` records the exact upstream URL, license, version, and checksum.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd edge && pytest tests/test_detector.py -v`
Expected: PASS. The ONNX inference test is marked `model` and skipped when the model file is absent.

- [ ] **Step 5: Commit**

```bash
git add edge/agent/detect edge/packaging/models edge/tests/test_detector.py
git commit -m "feat: add SCRFD detector adapter with model integrity check"
```

---

### Task 5: Quality gates

**Files:**
- Create: `edge/agent/detect/quality.py`
- Create: `edge/tests/test_quality.py`

**Interfaces:**
- Consumes: `Detection`.
- Produces:
  - `crop_face(frame: np.ndarray, bbox: tuple[int, int, int, int], margin: float = 0.2) -> np.ndarray`
  - `yaw_ratio(landmarks: np.ndarray) -> float`
  - `laplacian_variance(crop: np.ndarray) -> float`
  - `quality_gate(frame, detection) -> GateResult`
  - `GateResult(passed: bool, reason: Literal["too_small", "bad_pose", "blurry"] | None, crop_jpeg: bytes | None, sharpness: float)`

- [ ] **Step 1: Write the failing tests**

```python
def test_bbox_below_80px_fails():
    detection = Detection(bbox=(0, 0, 79, 80), landmarks=FRONTAL_LANDMARKS, confidence=0.9)
    result = quality_gate(np.zeros((480, 640, 3), dtype=np.uint8), detection)
    assert result.passed is False
    assert result.reason == "too_small"


def test_bbox_at_80px_can_pass():
    ...


def test_yaw_ratio_above_2_2_fails():
    landmarks = np.array([[10, 20], [80, 20], [10, 50], [40, 60], [70, 60]], dtype=np.float32)
    assert yaw_ratio(landmarks) > 2.2
    ...


def test_blurred_crop_below_80_variance_fails():
    crop = cv2.GaussianBlur(np.full((120, 120, 3), 127, dtype=np.uint8), (31, 31), 0)
    assert laplacian_variance(crop) < 80.0
    ...


def test_sharp_crop_passes():
    ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd edge && pytest tests/test_quality.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement gates**

Apply gate order: size, pose, sharpness. Convert to grayscale, compute `cv2.Laplacian(gray, cv2.CV_64F).var()`. Compute yaw ratio exactly as `max(d_left, d_right) / min(d_left, d_right)` using the nose and eye landmarks, where `d_left = norm(left_eye - nose)` and `d_right = norm(right_eye - nose)`. Crop with a 20 percent margin, clamp to frame bounds, encode JPEG quality 90. Return the encoded crop only when all gates pass.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd edge && pytest tests/test_quality.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add edge/agent/detect/quality.py edge/tests/test_quality.py
git commit -m "feat: add size, pose, and sharpness quality gates"
```

---

### Task 6: IoU tracker and best-shot buffer

**Files:**
- Create: `edge/agent/detect/tracker.py`
- Create: `edge/tests/test_tracker.py`
- Create: `edge/tests/test_best_shot.py`

**Interfaces:**
- Consumes: `TrackedFace`.
- Produces:
  - `SimpleTracker(iou_threshold: float = 0.3, max_age_seconds: float = 2.0)`
  - `SimpleTracker.update(detections: list[Detection], now: datetime) -> list[Detection]` assigning stable `track_id`s
  - `BestShotBuffer(window_seconds: float = 1.5)`
  - `BestShotBuffer.add(track_id: int, crop_jpeg: bytes, sharpness: float, now: datetime) -> None`
  - `BestShotBuffer.pop_ready(now: datetime) -> list[TrackedFace]`

- [ ] **Step 1: Write the failing tests**

`edge/tests/test_tracker.py`:
```python
def test_same_face_across_frames_keeps_track_id():
    tracker = SimpleTracker()
    first = tracker.update([det(bbox=(10, 10, 100, 100))], at(0.0))
    second = tracker.update([det(bbox=(14, 12, 104, 102))], at(0.5))
    assert first[0].track_id == second[0].track_id


def test_two_overlapping_people_get_different_ids():
    ...


def test_track_expires_after_max_age():
    tracker = SimpleTracker(max_age_seconds=2.0)
    tracker.update([det(bbox=(10, 10, 100, 100))], at(0.0))
    fresh = tracker.update([det(bbox=(10, 10, 100, 100))], at(3.0))
    assert fresh[0].track_id != 1
```

`edge/tests/test_best_shot.py`:
```python
def test_no_result_before_window_elapses():
    buffer = BestShotBuffer(window_seconds=1.5)
    buffer.add(track_id=1, crop_jpeg=b"a", sharpness=100.0, now=at(0.0))
    assert buffer.pop_ready(now=at(1.4)) == []


def test_returns_sharpest_frame_after_window():
    buffer = BestShotBuffer(window_seconds=1.5)
    buffer.add(1, b"soft", 90.0, at(0.0))
    buffer.add(1, b"sharp", 180.0, at(0.8))
    buffer.add(1, b"medium", 120.0, at(1.4))
    ready = buffer.pop_ready(now=at(1.6))
    assert len(ready) == 1
    assert ready[0].crop_jpeg == b"sharp"


def test_track_is_removed_after_pop():
    ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd edge && pytest tests/test_tracker.py tests/test_best_shot.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement tracker and buffer**

`SimpleTracker` computes IoU between each detection and existing track boxes, greedily assigns the highest IoU above `0.3`, and gives unmatched detections new monotonic ids. Tracks unseen for more than `max_age_seconds` are removed. This is the MVP tracker; a ByteTrack adapter can replace it behind the same `update` signature later. Keep the best-shot dict keyed by `track_id` with `first_seen`, `best_crop_jpeg`, and `best_sharpness`. `pop_ready` returns tracks where `now - first_seen >= window_seconds` and removes them. A track can be re-added after removal, which creates a fresh window.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd edge && pytest tests/test_tracker.py tests/test_best_shot.py -v`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add edge/agent/detect/tracker.py edge/tests/test_tracker.py edge/tests/test_best_shot.py
git commit -m "feat: add ioU tracker and 1.5 second best-shot buffer"
```

---

### Task 7: Local dedup cache

**Files:**
- Create: `edge/agent/dedup.py`
- Create: `edge/tests/test_dedup.py`

**Interfaces:**
- Produces:
  - `LocalDedupCache(window_minutes: int = 15, hamming_threshold: int = 8)`
  - `LocalDedupCache.seen(crop_jpeg: bytes, now: datetime) -> bool`
  - `LocalDedupCache.remember(crop_jpeg: bytes, now: datetime) -> None`

- [ ] **Step 1: Write the failing tests**

```python
def test_identical_crop_within_window_is_seen():
    cache = LocalDedupCache()
    cache.remember(crop, at(0))
    assert cache.seen(crop, at(5)) is True


def test_identical_crop_after_window_is_not_seen():
    cache = LocalDedupCache()
    cache.remember(crop, at(0))
    assert cache.seen(crop, at(16)) is False


def test_different_crop_is_not_seen():
    ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd edge && pytest tests/test_dedup.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement pHash dedup**

Resize to 32x32 grayscale, apply DCT, take the top-left 8x8 excluding DC, compare to the median, and produce a 64-bit hash. Two hashes are "same" when Hamming distance is at most 8. Store `(hash, last_seen)` in memory; evict entries older than the window on each call. Document that this is a heuristic and that the cloud's per-customer cooldown is authoritative.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd edge && pytest tests/test_dedup.py -v`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add edge/agent/dedup.py edge/tests/test_dedup.py
git commit -m "feat: add 15 minute local perceptual dedup cache"
```

---

### Task 8: Durable SQLite spool and retry worker

**Files:**
- Create: `edge/agent/spool.py`
- Create: `edge/agent/worker.py`
- Create: `edge/tests/test_spool.py`

**Interfaces:**
- Produces:
  - `SpoolItem(id: str, crop_jpeg: bytes, meta: dict, created_at: datetime, attempts: int, next_attempt_at: datetime, state: Literal["pending", "dead"])`
  - `SqliteSpool(path: Path)` with `enqueue`, `due`, `mark_sent`, `mark_failed`, `pending_count`, `dead_count`
  - `SpoolWorker(spool: SqliteSpool, cloud: CloudClient, api_key_provider: Callable[[], str], clock: Callable[[], datetime])` with `drain_once() -> int`

- [ ] **Step 1: Write the failing tests**

```python
def test_item_survives_reopen(tmp_path):
    spool = SqliteSpool(tmp_path / "spool.sqlite3")
    spool.enqueue(b"jpeg", {"camera_role": "ENTRANCE"})
    del spool
    reopened = SqliteSpool(tmp_path / "spool.sqlite3")
    assert reopened.pending_count() == 1


def test_failed_item_backs_off_and_retries(tmp_path):
    ...
    assert [item.next_attempt_at for item in spool.due(now)] == [now + 5s, ...]


def test_item_dead_letters_after_24_hours(tmp_path):
    ...


def test_mark_sent_removes_from_due(tmp_path):
    ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd edge && pytest tests/test_spool.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement spool and worker**

SQLite schema: `spool(id TEXT PRIMARY KEY, crop BLOB NOT NULL, meta_json TEXT NOT NULL, created_at TEXT NOT NULL, attempts INTEGER NOT NULL DEFAULT 0, next_attempt_at TEXT NOT NULL, state TEXT NOT NULL)`. Enable WAL and `synchronous=FULL`. `drain_once` selects due pending items, calls `upload_visit`, and on success deletes/`mark_sent`; on failure increments attempts, sets the next delay from the sequence, and dead-letters after 24 hours from `created_at`. A `401` response marks the item pending but stops the drain so a revoked key does not spin. The worker never raises out of `drain_once`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd edge && pytest tests/test_spool.py -v`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add edge/agent/spool.py edge/agent/worker.py edge/tests/test_spool.py
git commit -m "feat: add durable sqlite spool with exponential retry"
```

---

### Task 9: Pipeline orchestration and end-to-end agent test

**Files:**
- Create: `edge/agent/pipeline.py`
- Create: `edge/agent/main.py`
- Modify: `edge/agent/cli.py`
- Create: `edge/tests/test_pipeline.py`

**Interfaces:**
- Consumes: all prior components.
- Produces:
  - `AgentPipeline(source, detector, buffer, dedup, spool, clock)` with `run_once() -> PipelineResult`
  - `PipelineResult(accepted: int, suppressed: int, rejected: int)`
  - CLI: `python -m agent.cli run --config PATH`

- [ ] **Step 1: Write the failing test**

```python
def test_pipeline_enqueues_only_qualified_unique_faces(tmp_path):
    frames = [frame_with_face, frame_with_face, frame_with_small_face, frame_without_face]
    detector = FakeDetector([...])
    spool = SqliteSpool(tmp_path / "spool.sqlite3")
    pipeline = AgentPipeline(...)
    result = pipeline.run_once()
    assert result.accepted == 1
    assert result.rejected == 1
    assert spool.pending_count() == 1


def test_pipeline_suppresses_duplicate_within_fifteen_minutes(tmp_path):
    ...
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd edge && pytest tests/test_pipeline.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement pipeline and run command**

Per sampled frame: detect, run `SimpleTracker.update` to assign track ids, gate each detection, feed passing crops into the best-shot buffer keyed by `detection.track_id`, pop ready tracks, check dedup, enqueue with `VisitMeta` whose `client_event_id` is the spool item UUID. `run_once` processes exactly one sampled frame so tests are deterministic; the CLI loop calls it at the sampler cadence. `visited` count includes suppressed and rejected for `status`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd edge && pytest tests/test_pipeline.py -v`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add edge/agent edge/tests/test_pipeline.py
git commit -m "feat: wire edge capture pipeline and run command"
```

---

### Task 10: Windows capture adapters, packaging, and installer

**Files:**
- Create: `edge/agent/capture/windows_window.py`
- Modify: `edge/agent/capture/rtsp.py`
- Create: `edge/packaging/visitmemory-agent.spec`
- Create: `edge/packaging/build_windows.ps1`
- Create: `edge/tests/test_windows_adapters.py`

**Interfaces:**
- Produces:
  - `WindowsWindowSource(window_title_substring: str, fps: float, zone: tuple[float, float, float, float] | None = None)`
  - `WindowsWindowSource` restores a minimized window with `ShowWindow(SW_SHOWNOACTIVATE)` and keeps the system awake with `SetThreadExecutionState`
  - PyInstaller one-file build `visitmemory-agent.exe`
  - `build_windows.ps1` verifying the model SHA-256 and embedding `models/scrfd_500m.onnx`

- [ ] **Step 1: Write failing pure-logic tests**

```python
def test_zone_slice_uses_normalized_coordinates():
    frame = np.zeros((1080, 1920, 3), dtype=np.uint8)
    cropped = slice_zone(frame, (0.0, 0.0, 0.5, 0.5))
    assert cropped.shape == (540, 960, 3)


def test_zone_slice_rejects_out_of_range():
    with pytest.raises(ValueError):
        slice_zone(frame, (-0.1, 0.0, 0.5, 0.5))
```

`test_windows_adapters.py` contains Windows-only integration tests marked `windows` and skipped elsewhere.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd edge && pytest tests/test_windows_adapters.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement adapters and packaging**

`WindowsWindowSource` uses `pywin32` and `mss` (both pinned exactly in `edge/requirements.txt` in this task) to locate the window, capture its client area via `mss` or `PrintWindow`, and slice the configured normalized zone. The watchdog checks `IsIconic` every 3 seconds and restores without focus. The PyInstaller spec bundles the ONNX model and a version manifest; the build script runs the full test suite before packaging and fails on any model checksum mismatch.

- [ ] **Step 4: Run tests and build**

Run: `cd edge && pytest -v && powershell -File packaging/build_windows.ps1`
Expected: PASS; `dist/visitmemory-agent.exe --version` prints the agent version.

- [ ] **Step 5: Commit**

```bash
git add edge
git commit -m "feat: add windows capture, packaging, and installer build"
```

---

## Phase 2 Exit Checklist

- [ ] `python -m agent.cli activate` stores the API key via the secret store and never prints it after activation.
- [ ] The agent samples at 2 FPS, rejects sub-80px faces, blurry crops, and side profiles, and enqueues one best shot per track.
- [ ] Duplicate crops within 15 minutes are suppressed locally.
- [ ] Killing the process with pending spool items loses nothing; reopening the database resumes.
- [ ] Network failure does not stop capture; retries back off and dead-letter after 24 hours.
- [ ] The Windows build runs on a clean machine without Python installed.
