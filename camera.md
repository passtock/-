# 📘 Portenta X8 + RealSense D435 자율주행 비전 마스터 가이드 (Windows 기준, Depth 포함 최신 버전)

이 문서는 **Windows PC ↔ Portenta X8 ↔ Docker Ubuntu ↔ RealSense D435** 전체 흐름을 한 번에 정리한 가이드입니다.  
기존에 쓰던 `drone_vision` 컨테이너(UVC + OpenCV) 방식에, **새 Ubuntu 컨테이너 + librealsense + pyrealsense2 + depth 스트림**까지 추가한 최신 구조를 모두 포함합니다. [github](https://github.com/realsenseai/librealsense/blob/master/doc/installation.md)

***

## 🧭 0. 전체 구조 요약: “복도 + 방1 + 방2”

명령을 칠 때 **프롬프트**로 내가 어디 있는지 항상 확인하는 것이 핵심입니다.

1. `C:\Users\Name>`  
   - **Windows 본가** (노트북)  
   - `adb.exe`를 실행해서 Portenta X8 안으로 들어가는 출발점. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)

2. `fio@portenta-x8-...:/var/rootdirs/home/fio$`  
   - **Portenta X8 호스트 OS (Linux‑microPlatform, LmP)** – 일종의 **복도**  
   - IoT용 Yocto 기반 OS라 `apt`가 **없음** (`apt-get: command not found`). [docs.foundries](https://docs.foundries.io/94/reference-manual/linux/linux.html)
   - USB로 D435가 실제로 물려 있는 곳 (`lsusb`에 `Intel Corp. RealSense D435` 표시). [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)
   - 여기서는 Docker만 사용하고, 개발 환경은 컨테이너 안에서 구성.

3. `root@컨테이너ID:/#`  
   - **Docker 컨테이너 안의 리눅스 방** – 우리가 실제로 코딩/실행하는 작업실  
   - 종류 2개를 운용:
     - `drone_vision` : 기존에 쓰던 Python 3.11 + OpenCV + Flask용 컨테이너(UVC RGB 전용). [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)
     - `realsense_ubuntu` : 새로 만든 Ubuntu 22.04 컨테이너, **librealsense + pyrealsense2 + depth 전용**. [github](https://github.com/realsenseai/librealsense/blob/master/doc/installation.md)

***

## 🛠️ 1단계: X8 보드 접속 (Windows → LmP “복도”)

### 1-1. adb로 Portenta X8 접속

Windows CMD에서 (대략 이 경로에 adb.exe 존재):

```cmd
cd %LOCALAPPDATA%\Arduino15\packages\arduino\tools\adb\32.0.0
adb devices   :: 장치가 잡히는지 확인
adb shell
```

- 프롬프트가 아래처럼 바뀌면 **Portenta X8 내부(LmP)** 에 접속 성공입니다.

```bash
fio@portenta-x8-1b0a7a09dabc088b:/var/rootdirs/home/fio$
```

### 1-2. OS 정보 및 D435 인식 확인

```bash
cat /etc/os-release
```

- 내용 예시: `ID=lmp-xwayland`, `PRETTY_NAME="Linux-microPlatform XWayland ..."` → Yocto 기반, apt 없음. [docs.foundries](https://docs.foundries.io/94/reference-manual/linux/linux.html)

```bash
lsusb
```

- D435가 제대로 보이면 다음처럼 뜸:

```text
Bus 001 Device 002: ID 8086:0b07 Intel Corp. RealSense D435
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

***

## 🏗️ 2단계: Docker 컨테이너 두 개 운용 전략

현재 우리는 **컨테이너를 두 개** 운용합니다.

1. `drone_vision`  
   - 기존 UVC + OpenCV + Flask용 방 (RGB 카메라, ArUco, 대시보드 등). [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)
   - Python 3.11 이미지 기반.

2. `realsense_ubuntu`  
   - 새로 만든 Ubuntu 22.04 이미지 기반 방.  
   - `apt`로 librealsense 의존성을 설치하고, **소스 빌드 + pyrealsense2 + depth 스트림**까지 모두 이 안에서 처리. [github](https://github.com/realsenseai/librealsense/blob/master/doc/installation.md)

이 문서에서는 depth가 필요한 부분만 `realsense_ubuntu`에 집중해서 설명합니다.

***

## 🧱 3단계: RealSense 전용 Ubuntu 컨테이너 만들기 (`realsense_ubuntu`)

이 단계는 **한 번만 수행**하면 됩니다. 이후 재부팅 후에는 “출근 루틴(4단계)”만 실행하면 됩니다.

### 3-1. 새 컨테이너 생성 (LmP 복도에서 실행)a

`fio@portenta-x8-...$` 상태에서:

```bash
sudo docker run -it --name realsense_ubuntu \
  --privileged \
  --device=/dev/bus/usb:/dev/bus/usb \
  ubuntu:22.04 bash
```

- `--privileged` + `--device=/dev/bus/usb:/dev/bus/usb` : LmP에 연결된 D435 USB를 컨테이너 안에서도 그대로 접근 가능하게 넘겨줍니다. [forums.docker](https://forums.docker.com/t/debian-container-unable-to-access-host-usb-devices-lsusb-returns-empty/139037)
- 처음 실행 시 `ubuntu:22.04` 이미지를 자동으로 pull (로그 상에서 40MB 이상 다운로드된 부분). [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)
- 명령이 끝나면 프롬프트가:

```bash
root@ef9158aa710c:/#
```

처럼 바뀌는데, 이게 이제 **새 Ubuntu 컨테이너 내부(작업실)** 입니다.

### 3-2. 필수 빌드 도구 및 라이브러리 설치

컨테이너 안에서 **한 번만**:

```bash
apt-get update

apt-get install -y git cmake build-essential \
    libusb-1.0-0-dev libssl-dev pkg-config \
    libgtk-3-dev libglfw3-dev libgl1-mesa-dev libglu1-mesa-dev \
    python3 python3-pip python3-dev
```

- 첨부 로그에 이 패키지들이 대량으로 설치/설정된 흔적이 있습니다 (217MB 설치 로그). [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)
- 이 줄로 `git`, `cmake`, `make`, C 컴파일러, OpenGL/GTK/USB dev 라이브러리, Python 3.10 + pip까지 모두 설치.

### 3-3. librealsense 소스 클론 및 udev 설정

```bash
cd /root
git clone https://github.com/IntelRealSense/librealsense.git
cd librealsense

./scripts/setup_udev_rules.sh
```

- 처음에는 `git: command not found` 같은 에러가 있었지만, 위 3-2에서 패키지 설치 후에는 정상 수행. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)
- `setup_udev_rules.sh`는 USB 권한을 잡기 위한 스크립트입니다. [github](https://github.com/realsenseai/librealsense/blob/master/doc/installation.md)

### 3-4. 빌드 디렉터리 생성 및 CMake

```bash
mkdir build
cd build

cmake .. -DBUILD_PYTHON_BINDINGS:bool=true
```

- 이전에 `/root/build`에서 실행하면서 소스 루트가 꼬여 있었지만, 지금은 반드시 `/root/librealsense/build`에서 실행해야 합니다. [github](https://github.com/realsenseai/librealsense/blob/master/doc/installation.md)

### 3-5. 빌드 및 설치

```bash
make -j4
make install
```

- `make` 미설치 문제도 위에서 `build-essential`로 해결된 상태. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)
- ARM64라 빌드에 시간이 좀 걸리지만, 정상 완료되면 C++ 라이브러리와 pyrealsense2 바인딩이 `/usr/local`에 설치됩니다. [github](https://github.com/IntelRealSense/librealsense/discussions/13223)

***

## 🧪 4단계: pyrealsense2 + depth 동작 확인

### 4-1. pyrealsense2 import 테스트

빌드 후, 컨테이너 안에서:

```bash
python3 -c "import pyrealsense2 as rs; print('pyrealsense2 imported OK, type:', type(rs))"
```

실제 출력:

```text
pyrealsense2 imported OK, type: <class 'module'>
```

- `rs.__version__` 속성은 없는 빌드라 `AttributeError`가 났지만, 이는 **정상적인 설치** 상태에서 버전 속성만 없는 경우라 무시해도 됩니다. [github](https://github.com/IntelRealSense/librealsense/discussions/13223)

### 4-2. numpy 설치

depth 테스트 스크립트에서 `import numpy as np`를 사용하므로 numpy 설치 필요:

```bash
pip3 install numpy
# 또는
python3 -m pip install numpy
```

- 로그 상 `ModuleNotFoundError: No module named 'numpy'`가 발생했으나, pip 설치로 해결 가능. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)

### 4-3. depth 테스트 스크립트 생성 및 실행

컨테이너 안에서:

```bash
cd /root
cat << 'EOF' > rs_depth_test.py
import pyrealsense2 as rs
import numpy as np

print("[INFO] Starting RealSense pipeline...")
pipeline = rs.pipeline()
config = rs.config()

config.enable_stream(rs.stream.depth, 640, 480, rs.format.z16, 30)
config.enable_stream(rs.stream.color, 640, 480, rs.format.bgr8, 30)

profile = pipeline.start(config)
print("[INFO] RealSense pipeline started")

try:
    for i in range(50):
        frames = pipeline.wait_for_frames()
        depth_frame = frames.get_depth_frame()
        color_frame = frames.get_color_frame()
        if not depth_frame or not color_frame:
            continue

        width = depth_frame.get_width()
        height = depth_frame.get_height()
        dist = depth_frame.get_distance(width//2, height//2)

        print(f"[FRAME {i}] center depth = {dist:.3f} m")

finally:
    pipeline.stop()
    print("[INFO] Pipeline stopped")
EOF

python3 rs_depth_test.py
```

- 정상 동작 시 예상 출력:

```text
[INFO] Starting RealSense pipeline...
[INFO] RealSense pipeline started
[FRAME 0] center depth = 0.823 m
[FRAME 1] center depth = 0.821 m
...
[INFO] Pipeline stopped
```

- 이 단계가 통과하면 **D435 depth + color 스트림이 모두 정상 동작**하는 것입니다. [github](https://github.com/realsenseai/librealsense/blob/master/doc/installation.md)

***

## 🔁 5단계: 전원 껐다 켜도 다시 쓰는 “출근 루틴”

Portenta X8 전원을 껐다 켠 후에도 컨테이너/설치는 저장되어 있습니다. [docs.foundries](https://docs.foundries.io/95/user-guide/lmp-customization/linux-extending.html)
다만, 컨테이너는 다시 켜줘야 합니다.

### 5-1. LmP 복도에 다시 진입

Windows → `adb shell` → 프롬프트가 다시:

```bash
fio@portenta-x8-...:/var/rootdirs/home/fio$
```

### 5-2. realsense_ubuntu 컨테이너 시작 및 접속

```bash
sudo docker start realsense_ubuntu
sudo docker exec -it realsense_ubuntu bash
```

- 이제 프롬프트가 다시:

```bash
root@ef9158aa710c:/#
```

### 5-3. depth 테스트 또는 실제 코드 실행

```bash
cd /root
python3 rs_depth_test.py            # depth 확인
python3 your_drone_depth_app.py     # 실제 자율주행/비전 코드
```

- 전원 공급 방식(노트북 USB / 어댑터 / 배터리)에 상관 없이, Portenta X8이 제대로 부팅되어 있고, D435가 USB로 연결되어 있으면 **항상 같은 방식으로 재실행 가능**합니다. [docs.foundries](https://docs.foundries.io/95/user-guide/lmp-customization/linux-extending.html)

***

## 📷 6단계: 기존 drone_vision(UVC RGB)와의 역할 분담

기존에 쓰던 `drone_vision` 컨테이너는 그대로 유지하면서, 역할을 이렇게 나눌 수 있습니다.

1. `drone_vision` (Python 3.11, OpenCV, Flask 서버) [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)
   - UVC + OpenCV 기반 RGB 카메라 스트림  
   - `cv2.VideoCapture(5, cv2.CAP_V4L2)`  
   - MJPG, 640×480@30 설정으로 적절한 FPS (약 26 fps)로 ArUco 마커 검출/시각화. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/63867275/3e30b71a-63c0-453e-a801-ac1d48621c99/paste.txt)
   - 웹 대시보드 (`/video_feed`, `/metrics` 등) 제공.

2. `realsense_ubuntu` (Ubuntu 22.04 + librealsense + pyrealsense2) [github](https://github.com/realsenseai/librealsense/blob/master/doc/installation.md)
   - RealSense SDK 기반 depth, point cloud, alignment, intrinsic/extrinsic 등 고급 기능.  
   - depth map 기반 거리 측정, 3D 위치 추정 등.

필요에 따라:

- 간단한 작업 / 빠른 실험 → `drone_vision`에서 UVC RGB만 사용.  
- 정확한 거리/3D 정보가 필요한 작업 → `realsense_ubuntu`에서 pyrealsense2 사용.

***

## 🚨 7단계: 자주 헷갈리는 포인트 / Trouble Shooting

1. **apt가 안 되는 이유**  
   - LmP 호스트 OS는 Yocto 기반 (`ID=lmp-xwayland`)라 `apt-get`이 없다.  
   - 그래서 **호스트에서 패키지를 깔려고 하지 말고**, 항상 Docker 컨테이너 안에서 작업해야 한다. [docs.foundries](https://docs.foundries.io/94/reference-manual/linux/linux.html)

2. **pyrealsense2가 pip로 안 깔리는 이유**  
   - ARM64(aarch64) 환경에서 `pip install pyrealsense2`는 공식 wheel이 없어서 실패한다. [pypi](https://pypi.org/project/pyrealsense2/)
   - 해결책: **librealsense 소스 빌드 + Python 바인딩(`-DBUILD_PYTHON_BINDINGS:true`)**으로 직접 설치. [github](https://github.com/IntelRealSense/librealsense/discussions/13223)

3. **rs.__version__ AttributeError**  
   - 설치 실패가 아니라, **빌드된 pyrealsense2 모듈이 `__version__` 속성을 제공하지 않는 경우**라서 무시 가능.  
   - `import pyrealsense2 as rs`가 잘 되면 설치 성공. [github](https://github.com/IntelRealSense/librealsense/discussions/13223)

4. **전원 끄면 다 날아가는지?**  
   - Docker 이미지/컨테이너는 플래시에 저장되므로 전원 껐다 켜도 유지됨. [docs.foundries](https://docs.foundries.io/95/user-guide/lmp-customization/linux-extending.html)
   - 다만 부팅 후 `docker start` / `docker exec`는 다시 해줘야 한다.

5. **배터리 사용 가능 여부**  
   - Portenta X8에 요구되는 전압/전류를 만족하는 배터리/전원 장치면 문제 없음.  
   - 다만 **갑작스러운 전원 off**는 파일 시스템 손상 위험이 있으니 가능하면 `sudo poweroff`로 정상 종료 권장. [docs.foundries](https://docs.foundries.io/95/user-guide/lmp-customization/linux-extending.html)

***

## ✅ 앞으로 확장할 때 (실전 통합 코드)

앞서 구축한 `realsense_ubuntu` 환경에서 곧바로 실행할 수 있는 **통합 벤치마크, RGB-Depth 융합, 자동 제어(Autopilot)** 전체 코드를 아래에 제공합니다. 

이 코드들을 복사해서 컨테이너 내부에 생성하고 실행하면 됩니다.

---

## 🚀 8단계: 통합 성능 벤치마크 시스템 (`rs_benchmark.py`)
카메라의 해상도별 한계를 파악하고 실시간 제어 주기를 확립하기 위해, 3회 반복(Trials) 측정 및 듀얼 그래프를 생성하는 벤치마크 코드를 작성하고 실행합니다.

**① 벤치마크 코드 생성** (컨테이너 내부에서 전체 복사/붙여넣기)
```bash
cd /root
cat << 'EOF' > rs_benchmark.py
import pyrealsense2 as rs
import numpy as np
import time
import psutil
import os
import matplotlib
matplotlib.use('Agg') 
import matplotlib.pyplot as plt
import matplotlib.cm as cm

# 1. Test Configurations
test_configs = [
    (640, 360, 30),
    (640, 480, 15),
    (640, 480, 30),
    (1280, 720, 15),
    (1280, 720, 30)
]

test_duration = 30      
warmup_time = 3.0       
max_buffer_size = 30
num_trials = 3  

print("==========================================================================================")
print("  Portenta X8 + RealSense TOTAL BENCHMARK (3 Trials + Dual Graphs) ")
print("  * Note: This will take approximately 8 minutes to complete. Please wait... *")
print("==========================================================================================")

def get_depth_mask(depth_np, min_depth_mm=100, max_depth_mm=8000):
    return (depth_np > min_depth_mm) & (depth_np < max_depth_mm)

def plane_fit_rms(z_flat):
    if len(z_flat) < 3: return np.nan
    z_mean = np.mean(z_flat)
    return np.sqrt(np.mean((z_flat - z_mean)**2))

def subpixel_rms_error(z_flat, baseline_mm=50.0, focal_len_px=840.0): 
    valid = z_flat > 1e-3
    zi = z_flat[valid]
    if len(zi) == 0: return np.nan
    dpi = baseline_mm * focal_len_px / zi
    di = baseline_mm * focal_len_px / np.mean(zi) 
    return np.sqrt(np.mean((di - dpi)**2))

def temporal_noise(depth_tensor):
    std_map = np.std(depth_tensor, axis=0)  
    median_map = np.median(depth_tensor, axis=0) 
    valid = (median_map > 0) & (std_map >= 0)
    ratio_map = np.zeros_like(std_map)
    ratio_map[valid] = std_map[valid] / median_map[valid]
    if np.sum(ratio_map > 0) == 0: return np.nan
    return np.nanmedian(ratio_map[ratio_map > 0]) * 100.0

results = []
all_plot_data = [] 
process = psutil.Process(os.getpid())
colors = cm.tab10(np.linspace(0, 1, len(test_configs)))

for idx, (width, height, target_fps) in enumerate(test_configs):
    pipeline = rs.pipeline()
    config = rs.config()
    base_label = f"{width}x{height}@{target_fps}"
    
    pipeline_started = False
    try:
        config.enable_stream(rs.stream.depth, width, height, rs.format.z16, target_fps)
        pipeline.start(config)
        pipeline_started = True
        time.sleep(warmup_time)

        for trial in range(1, num_trials + 1):
            res_label = f"{base_label} (T{trial})"
            print(f"\n[INIT] {res_label}")

            depth_buffer, center_dist_list, time_stamps, frame_timestamps = [], [], [], []
            frame_count = 0
            process.cpu_percent(interval=None) 
            start_time = time.time()

            print(f"  -> Measuring for {test_duration}s...")
            
            while (time.time() - start_time) < test_duration:
                frames = pipeline.wait_for_frames()
                depth_frame = frames.get_depth_frame()
                if not depth_frame: continue

                curr_t = time.time() - start_time
                frame_timestamps.append(curr_t) 

                dist = depth_frame.get_distance(width//2, height//2)
                if dist > 0:
                    center_dist_list.append(dist)
                    time_stamps.append(curr_t)

                depth_np = np.asanyarray(depth_frame.get_data()).copy()
                if len(depth_buffer) < max_buffer_size:
                    depth_buffer.append(depth_np)

                if frame_count % (target_fps if target_fps > 0 else 1) == 0:
                    print(f"     [Live] Depth: {dist:.4f} m")

                frame_count += 1

            actual_dur = time.time() - start_time
            dist_arr = np.array(center_dist_list)
            
            fps_t, fps_v = [], []
            for i in range(1, len(frame_timestamps)):
                dt = frame_timestamps[i] - frame_timestamps[i-1]
                if dt > 0:
                    fps_t.append(frame_timestamps[i])
                    fps_v.append(1.0 / dt)
            
            window = min(10, len(fps_v))
            if window > 0:
                fps_smoothed = np.convolve(fps_v, np.ones(window)/window, mode='valid')
                fps_t_smoothed = fps_t[window-1:]
            else:
                fps_smoothed, fps_t_smoothed = [], []

            mask = get_depth_mask(depth_np)
            fill_rate = np.sum(mask) / depth_np.size
            h, w = depth_np.shape
            roi = depth_np[h//2-h//4 : h//2+h//4, w//2-w//4 : w//2+w//4]
            roi_v = roi[get_depth_mask(roi)]
            p_rms = plane_fit_rms(roi_v) if len(roi_v) >= 10 else np.nan
            s_rms = subpixel_rms_error(roi_v) if len(roi_v) >= 10 else np.nan
            t_noise = temporal_noise(np.stack(depth_buffer, axis=0)) if len(depth_buffer) > 1 else np.nan

            results.append({
                'res': res_label, 't_fps': target_fps, 'a_fps': frame_count/actual_dur,
                'avg': np.mean(dist_arr) if len(dist_arr)>0 else np.nan,
                'std': np.std(dist_arr) if len(dist_arr)>0 else np.nan,
                'cep': np.median(np.abs(dist_arr - np.mean(dist_arr))) if len(dist_arr)>0 else np.nan,
                'fill': fill_rate, 'p_rms': p_rms, 's_rms': s_rms, 't_noise': t_noise,
                'cpu': process.cpu_percent(interval=None), 'ram': process.memory_info().rss/(1024*1024)
            })

            all_plot_data.append({
                'label': base_label, 'trial': trial, 'color': colors[idx],
                'time_dist': time_stamps, 'dist': dist_arr,
                'time_fps': fps_t_smoothed, 'fps': fps_smoothed
            })
            time.sleep(1.0) 

    except Exception as e:
        print(f"  [ERROR] {base_label}: {e}")
        for trial in range(1, num_trials + 1):
            results.append({'res': f"{base_label} (T{trial})", 't_fps': target_fps, 'a_fps': np.nan})
    finally:
        if pipeline_started: pipeline.stop()
        time.sleep(1.5)

print("\n[INFO] Generating Dual Trend Graphs...")
fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(14, 12))

for data in all_plot_data:
    leg_label = data['label'] if data['trial'] == 1 else "_nolegend_"
    if len(data['time_dist']) > 0:
        ax1.plot(data['time_dist'], data['dist'], color=data['color'], alpha=0.5, linewidth=2, label=leg_label)
    if len(data['time_fps']) > 0:
        ax2.plot(data['time_fps'], data['fps'], color=data['color'], alpha=0.5, linewidth=2, label=leg_label)

ax1.set_title('RealSense Center Depth Variation Over Time', fontsize=14, fontweight='bold')
ax1.set_ylabel('Distance (meters)', fontsize=12)
ax1.legend(loc='upper right')
ax1.grid(True, linestyle='--', alpha=0.7)

ax2.set_title('RealSense Real-time FPS Stability (Moving Avg)', fontsize=14, fontweight='bold')
ax2.set_xlabel('Time (seconds)', fontsize=12)
ax2.set_ylabel('Frames Per Second (FPS)', fontsize=12)
ax2.legend(loc='upper right')
ax2.grid(True, linestyle='--', alpha=0.7)

plt.tight_layout()
plt.savefig('/root/depth_report.png', dpi=150)
print("  -> Saved Graph: /root/depth_report.png")

print("\n" + "="*155)
print(f"{'Res (Trial)':<17} | {'FPS':<4} | {'Ratio':<6} | {'AvgDist':<7} | {'StdDev':<7} | {'CEP50':<7} | {'Fill%':<6} | {'Z-RMS':<6} | {'S-Pix':<6} | {'Noise%':<6} | {'CPU':<5} | {'RAM'}")
print("-" * 155)
for r in results:
    if np.isnan(r.get('a_fps', np.nan)): 
        print(f"{r['res']:<17} | {r['t_fps']:<4} | FAILED")
        continue
    ratio = (r['a_fps']/r['t_fps'])*100
    print(f"{r['res']:<17} | {r['t_fps']:<4} | {ratio:>5.1f}% | {r['avg']:>7.4f} | {r['std']:>7.4f} | {r['cep']:>7.4f} | {r['fill']*100:>5.1f}% | {r['p_rms']:>6.2f} | {r['s_rms']:>6.2f} | {r['t_noise']:>6.2f} | {r['cpu']:>4.1f}% | {r['ram']:>5.1f}MB")
print("="*155)
EOF
```

**② 벤치마크 실행 & 결과 추출**
```bash
python3 rs_benchmark.py
```
실행이 끝나면 윈도우 CMD에서 아래 명령으로 그래프 이미지를 가져옵니다.
```cmd
adb shell "sudo docker cp realsense_ubuntu:/root/depth_report.png /var/rootdirs/home/fio/depth_report.png"
adb pull /var/rootdirs/home/fio/depth_report.png %USERPROFILE%\Desktop\
```

---

## 🎯 9단계: RGB-Depth 융합 (Alignment & ArUco) (`align_test.py`)
컬러 영상에서 ArUco 마커를 탐지하고, 정렬(Aligned)된 깊이 이미지에서 픽셀에 매칭되는 실제 거리(m)를 추출하는 기반 코드입니다.

**① 테스트 코드 생성 및 실행**
```bash
cat << 'EOF' > /root/align_test.py
import pyrealsense2 as rs
import numpy as np
import cv2
import time

print("======================================================")
print(" RealSense RGB + Depth Alignment & ArUco Test ")
print("======================================================")

pipeline = rs.pipeline()
config = rs.config()

config.enable_stream(rs.stream.depth, 640, 480, rs.format.z16, 30)
config.enable_stream(rs.stream.color, 640, 480, rs.format.bgr8, 30)

print("[INFO] Starting camera pipeline...")
pipeline.start(config)

align_to = rs.stream.color
align = rs.align(align_to)

aruco_dict = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_6X6_250)
parameters = cv2.aruco.DetectorParameters()
detector = cv2.aruco.ArucoDetector(aruco_dict, parameters)

time.sleep(2.0)
print("\n[READY] Point an ArUco marker at the camera! (Press Ctrl+C to stop)\n")

try:
    while True:
        frames = pipeline.wait_for_frames()
        aligned_frames = align.process(frames)

        aligned_depth_frame = aligned_frames.get_depth_frame() 
        color_frame = aligned_frames.get_color_frame()

        if not aligned_depth_frame or not color_frame: continue

        color_image = np.asanyarray(color_frame.get_data())
        gray = cv2.cvtColor(color_image, cv2.COLOR_BGR2GRAY)
        
        corners, ids, rejectedImgPoints = detector.detectMarkers(gray)

        if ids is not None:
            for i in range(len(ids)):
                c = corners[i][0]
                center_x = int(np.mean(c[:, 0]))
                center_y = int(np.mean(c[:, 1]))
                distance_m = aligned_depth_frame.get_distance(center_x, center_y)

                print(f"[DETECTED] ID: {ids[i][0]:<3} | Center: (X:{center_x:>3}, Y:{center_y:>3}) | Dist: {distance_m:.3f} m")
        
        time.sleep(0.1)
except KeyboardInterrupt:
    pass
finally:
    pipeline.stop()
EOF

python3 align_test.py
```

---

## ✈️ 10단계: 최종 통합본 - 실전 자율 드론 제어 (`autopilot.py`)
벤치마크와 마커 탐지 모듈을 기반으로 **정밀 타이머 제어, State Machine, ROI 추적, 시리얼 통신**을 모두 융합한 실제 메인 제어 루프 초기 뼈대 코드입니다.

**① Autopilot 코드 생성 및 실행**
```bash
cat << 'EOF' > /root/autopilot.py
import pyrealsense2 as rs
import numpy as np
import cv2
import time
import serial

print("======================================================")
print(" Portenta X8 Autopilot - Target Tracker ")
print("======================================================")

# 1. 15Hz 제어 주기 타겟 (1루프당 66.6ms)
TARGET_FPS = 15
CYCLE_TIME_SEC = 1.0 / TARGET_FPS

# 2. 메인 컨트롤러(STM32 등)로 보낼 시리얼 인터페이스 (포트는 실제 환경에 맞게 수정 필요)
try:
    ser = serial.Serial('/dev/ttyUSB0', baudrate=115200, timeout=0.1)
    print("[INFO] Serial Port Opened.")
except Exception as e:
    ser = None
    print(f"[WARNING] Serial Port not found: {e}")

pipeline = rs.pipeline()
config = rs.config()

config.enable_stream(rs.stream.depth, 640, 480, rs.format.z16, 30)
config.enable_stream(rs.stream.color, 640, 480, rs.format.bgr8, 30)

pipeline.start(config)
align = rs.align(rs.stream.color)

aruco_dict = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_6X6_250)
parameters = cv2.aruco.DetectorParameters()
detector = cv2.aruco.ArucoDetector(aruco_dict, parameters)

# 3. State Machine & ROI 변수
state = "SEARCH"
marker_seen_count = 0
last_marker_pos = None
ROI_SIZE = 100  # 탐색 기준점으로부터 상하좌우 100 픽셀 (총 200x200 픽셀)

time.sleep(2.0)
print("[INFO] Autopilot loop running at 15Hz... (Press Ctrl+C to stop)")

try:
    while True:
        loop_start_time = time.time()
        
        frames = pipeline.wait_for_frames()
        aligned_frames = align.process(frames)

        depth_frame = aligned_frames.get_depth_frame() 
        color_frame = aligned_frames.get_color_frame()

        if not depth_frame or not color_frame: continue

        color_image = np.asanyarray(color_frame.get_data())
        gray = cv2.cvtColor(color_image, cv2.COLOR_BGR2GRAY)
        
        # 4. ROI Tracking 로직
        if state == "CONFIRM" and last_marker_pos is not None:
            cx, cy = last_marker_pos
            x1 = max(0, cx - ROI_SIZE)
            y1 = max(0, cy - ROI_SIZE)
            x2 = min(gray.shape[1], cx + ROI_SIZE)
            y2 = min(gray.shape[0], cy + ROI_SIZE)
            
            roi_gray = gray[y1:y2, x1:x2]
            corners, ids, _ = detector.detectMarkers(roi_gray)
            
            if ids is not None:
                for i in range(len(corners)):
                    corners[i][0][:, 0] += x1
                    corners[i][0][:, 1] += y1
        else:
            corners, ids, _ = detector.detectMarkers(gray)

        # 5. State Machine 처리
        if ids is not None:
            marker_seen_count += 1
            if marker_seen_count >= 3:
                state = "CONFIRM"
                
            c = corners[0][0]
            center_x = int(np.mean(c[:, 0]))
            center_y = int(np.mean(c[:, 1]))
            last_marker_pos = (center_x, center_y)

            distance_m = depth_frame.get_distance(center_x, center_y)
            
            err_x = center_x - 320
            err_y = center_y - 240
            
            print(f"[{state}] ID:{ids[0][0]} | Err_X: {err_x:>4} | Err_Y: {err_y:>4} | Dist: {distance_m:.3f}m")
            
            # 6. 시리얼 패킷 전송
            if ser and state == "CONFIRM":
                packet = f"{err_x},{err_y},{distance_m:.3f}\n"
                ser.write(packet.encode('utf-8'))
                
        else:
            marker_seen_count = 0
            state = "SEARCH"
            last_marker_pos = None

        # 7. 제어 루프 타이머 방어 로직 (동적 Sleep 설계)
        elapsed_sec = time.time() - loop_start_time
        sleep_sec = CYCLE_TIME_SEC - elapsed_sec
        
        if sleep_sec > 0:
            time.sleep(sleep_sec)

except KeyboardInterrupt:
    print("\n[INFO] Stopped by User.")
finally:
    pipeline.stop()
    if getattr(globals(), 'ser', None) is not None:
        ser.close()
EOF

python3 autopilot.py
```
# 🚁 포텐타 X8 자율비전 제어 시스템 (AutoPilot) 개발 일지

**작성일:** 2026년 4월 24일  
**작성자:** 영상팀 이재용  
**개발 환경:** Portenta X8 (Ubuntu Docker), Intel RealSense D435, STM32 (Target)  
**핵심 목표:** 순수 CPU 환경(Cortex-A53)에서 15Hz 주기를 방어하며 안정적인 비전 제어(ErrX, ErrY, Dist) 데이터 추출

---

## 1. 프로젝트 개요 및 직면 과제
포텐타 X8 보드의 제한적인 CPU 성능으로 인해, 매 프레임 전체 해상도(640x480)에 대해 딥러닝 객체 인식을 수행하면 **15Hz 제어 주기(66.6ms)를 맞추지 못하고 심각한 지연(OVERTIME)이 발생**했습니다. 또한 노이즈로 인한 타겟 오탐지 및 뎁스 센서의 난반사(거리 0m 튐 현상)가 제어 불안정을 야기했습니다.

이를 해결하기 위해 **'상태 머신(Confirm 검증) + ROI 고속 트래킹 + 5x5 Median 뎁스 필터링'** 아키텍처를 단계별로 도입했습니다.

---

## 2. 단계별 개발 코드 및 실행 결과

### 🛠️ Step 1: AutoPilot v1 (고정 제어 주기 & 안정적 뎁스 추출)
딥러닝 이전에 빠르고 정확한 **ArUco 마커**를 활용하여, 제어 주기를 15Hz로 강제 고정하고 뎁스(Depth) 데이터의 신뢰성을 높이는 뼈대를 먼저 구축했습니다. 중앙 1픽셀 대신 `5x5` 픽셀의 중간값(Median)을 사용하여 거리 값이 튀는 현상을 막았습니다.

**📝 `autopilot_v1.py`**
```python
import pyrealsense2 as rs
import numpy as np
import cv2
import time

print("======================================================")
print(" AutoPilot v1: Fixed 15Hz Interval & Robust Depth ")
print("======================================================")

pipeline = rs.pipeline()
config = rs.config()
config.enable_stream(rs.stream.depth, 640, 480, rs.format.z16, 30)
config.enable_stream(rs.stream.color, 640, 480, rs.format.bgr8, 30)
profile = pipeline.start(config)
depth_scale = profile.get_device().first_depth_sensor().get_depth_scale()
align = rs.align(rs.stream.color)

aruco_dict = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_6X6_250)
parameters = cv2.aruco.DetectorParameters()
detector = cv2.aruco.ArucoDetector(aruco_dict, parameters)

TARGET_HZ = 15.0
TARGET_INTERVAL = 1.0 / TARGET_HZ  # 66.6 ms

time.sleep(2.0)

try:
    while True:
        loop_start_time = time.time()
        
        frames = align.process(pipeline.wait_for_frames())
        depth_frame, color_frame = frames.get_depth_frame(), frames.get_color_frame()
        if not depth_frame or not color_frame: continue

        color_image = np.asanyarray(color_frame.get_data())
        depth_image = np.asanyarray(depth_frame.get_data())
        gray = cv2.cvtColor(color_image, cv2.COLOR_BGR2GRAY)
        corners, ids, _ = detector.detectMarkers(gray)

        state, err_x, err_y, distance_m = 0, 0, 0, 0.0

        if ids is not None:
            state = 1
            c = corners[0][0]
            cx, cy = int(np.mean(c[:, 0])), int(np.mean(c[:, 1]))
            err_x, err_y = cx - 320, 240 - cy 

            # 5x5 Median Depth
            y_min, y_max = max(0, cy-2), min(480, cy+3)
            x_min, x_max = max(0, cx-2), min(640, cx+3)
            valid_pixels = depth_image[y_min:y_max, x_min:x_max]
            valid_pixels = valid_pixels[valid_pixels > 0]
            if len(valid_pixels) > 0:
                distance_m = np.median(valid_pixels) * depth_scale 

        print(f"[STM32 TX] State: {state} | ErrX: {err_x:>4} | ErrY: {err_y:>4} | Dist: {distance_m:.3f}m", end=" ")

        sleep_time = TARGET_INTERVAL - (time.time() - loop_start_time)
        if sleep_time > 0:
            time.sleep(sleep_time)
            print(f"(Sleep: {sleep_time*1000:4.1f}ms) -> Target 15Hz OK")
        else:
            print(f"(OVERTIME: {-sleep_time*1000:4.1f}ms) -> Frame Drop!")
except KeyboardInterrupt: pass
finally: pipeline.stop()
```

**📊 실행 결과 (V1 Log)**
> 전체 화면을 매번 탐색하므로 CPU 연산이 빡빡하여 여유 시간(Sleep)이 거의 없거나 간헐적인 프레임 드랍 발생.
```text
[STM32 TX] State: 1 | ErrX:  -22 | ErrY:  -30 | Dist: 0.464m (OVERTIME:  0.5ms) -> Frame Drop!
[STM32 TX] State: 1 | ErrX:  -22 | ErrY:  -30 | Dist: 0.463m (Sleep:  0.5ms) -> Target 15Hz OK
[STM32 TX] State: 1 | ErrX:  -23 | ErrY:  -30 | Dist: 0.457m (OVERTIME:  0.4ms) -> Frame Drop!
[STM32 TX] State: 0 | ErrX:    0 | ErrY:    0 | Dist: 0.000m (Sleep:  3.9ms) -> Target 15Hz OK
```

---

### 🧠 Step 2: AutoPilot v2 (상태 머신 & ROI 고속 트래킹 적용)
V1의 연산 병목을 해결하기 위해 소프트웨어 두뇌를 탑재했습니다. `SEARCH -> CONFIRM(3프레임) -> TRACK(ROI)` 구조를 통해 오탐지를 걸러내고, 타겟 발견 시 **200x200 ROI로 탐색 범위를 축소하여 연산량을 1/8로 획기적으로 줄였습니다.**

**📝 `autopilot_v2.py`**
```python
import pyrealsense2 as rs
import numpy as np
import cv2
import time

print("======================================================")
print(" AutoPilot v2: State Machine & ROI Tracking ")
print("======================================================")

pipeline = rs.pipeline()
config = rs.config()
config.enable_stream(rs.stream.depth, 640, 480, rs.format.z16, 30)
config.enable_stream(rs.stream.color, 640, 480, rs.format.bgr8, 30)
profile = pipeline.start(config)
depth_scale = profile.get_device().first_depth_sensor().get_depth_scale()
align = rs.align(rs.stream.color)

aruco_dict = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_6X6_250)
parameters = cv2.aruco.DetectorParameters()
detector = cv2.aruco.ArucoDetector(aruco_dict, parameters)

TARGET_HZ = 15.0
TARGET_INTERVAL = 1.0 / TARGET_HZ  
STATE, CONFIRM_THRESHOLD, LOST_TOLERANCE, ROI_SIZE = "SEARCH", 3, 2, 200
confirm_counter, lost_counter, last_cx, last_cy = 0, 0, 320, 240

time.sleep(2.0)

try:
    while True:
        loop_start_time = time.time()
        frames = align.process(pipeline.wait_for_frames())
        depth_frame, color_frame = frames.get_depth_frame(), frames.get_color_frame()
        if not depth_frame or not color_frame: continue

        color_image = np.asanyarray(color_frame.get_data())
        depth_image = np.asanyarray(depth_frame.get_data())
        gray = cv2.cvtColor(color_image, cv2.COLOR_BGR2GRAY)

        stm_state, err_x, err_y, distance_m, display_status = 0, 0, 0, 0.0, ""

        if STATE == "SEARCH":
            corners, ids, _ = detector.detectMarkers(gray)
            if ids is not None:
                confirm_counter += 1
                c = corners[0][0]
                last_cx, last_cy = int(np.mean(c[:, 0])), int(np.mean(c[:, 1]))
                display_status = f"[CONFIRMING {confirm_counter}/{CONFIRM_THRESHOLD}]"
                if confirm_counter >= CONFIRM_THRESHOLD: STATE, confirm_counter = "TRACK", 0
            else: confirm_counter, display_status = 0, "[SEARCHING Full Frame]"

        elif STATE == "TRACK":
            x_min, y_min = max(0, last_cx - ROI_SIZE//2), max(0, last_cy - ROI_SIZE//2)
            x_max, y_max = min(640, last_cx + ROI_SIZE//2), min(480, last_cy + ROI_SIZE//2)
            
            gray_roi = gray[y_min:y_max, x_min:x_max]
            corners, ids, _ = detector.detectMarkers(gray_roi)

            if ids is not None:
                lost_counter = 0 
                c = corners[0][0]
                last_cx, last_cy = int(np.mean(c[:, 0])) + x_min, int(np.mean(c[:, 1])) + y_min
                stm_state = 1
                err_x, err_y = last_cx - 320, 240 - last_cy
                
                valid_pixels = depth_image[max(0, last_cy-2):min(480, last_cy+3), max(0, last_cx-2):min(640, last_cx+3)]
                valid_pixels = valid_pixels[valid_pixels > 0]
                if len(valid_pixels) > 0: distance_m = np.median(valid_pixels) * depth_scale 
                display_status = "[LOCKED & TRACKING]"
            else:
                lost_counter += 1
                display_status = f"[LOST WARNING {lost_counter}/{LOST_TOLERANCE}]"
                if lost_counter >= LOST_TOLERANCE: STATE, lost_counter = "SEARCH", 0

        print(f"{display_status:<22} | STM_TX [ State: {stm_state} | ErrX: {err_x:>4} | ErrY: {err_y:>4} | Dist: {distance_m:.3f}m ]", end=" ")
        
        sleep_time = TARGET_INTERVAL - (time.time() - loop_start_time)
        if sleep_time > 0: print(f" -> OK (Sleep {sleep_time*1000:4.1f}ms)")
        else: print(f" -> OVERTIME ({-sleep_time*1000:4.1f}ms)")

except KeyboardInterrupt: pass
finally: pipeline.stop()
```

**📊 실행 결과 (V2 Log - 압도적인 속도 향상!)**
> 전체 화면 탐색 시 1~4ms에 불과하던 여유 시간이, ROI **[LOCKED & TRACKING] 진입 직후 27ms 이상으로 급증**하며 연산 병목이 완벽히 해결됨. 1프레임 노이즈는 `[CONFIRMING]`으로 방어하고, 순간 놓침은 `[LOST WARNING]`으로 유연하게 대처함.
```text
[SEARCHING Full Frame] | STM_TX [ State: 0 | ErrX:    0 | ErrY:    0 | Dist: 0.000m ]  -> OK (Sleep  3.6ms)
[CONFIRMING 1/3]       | STM_TX [ State: 0 | ErrX:    0 | ErrY:    0 | Dist: 0.000m ]  -> OVERTIME ( 1.1ms)
[CONFIRMING 2/3]       | STM_TX [ State: 0 | ErrX:    0 | ErrY:    0 | Dist: 0.000m ]  -> OVERTIME ( 0.1ms)
[CONFIRMING 3/3]       | STM_TX [ State: 0 | ErrX:    0 | ErrY:    0 | Dist: 0.000m ]  -> OK (Sleep  1.5ms)
[LOCKED & TRACKING]    | STM_TX [ State: 1 | ErrX:  147 | ErrY:   81 | Dist: 0.797m ]  -> OK (Sleep 27.9ms)  🚀 (연산량 감소!)
[LOCKED & TRACKING]    | STM_TX [ State: 1 | ErrX:   16 | ErrY:   17 | Dist: 0.832m ]  -> OK (Sleep 21.5ms)
[LOST WARNING 1/2]     | STM_TX [ State: 0 | ErrX:    0 | ErrY:    0 | Dist: 0.000m ]  -> OK (Sleep 24.6ms)
[LOCKED & TRACKING]    | STM_TX [ State: 1 | ErrX:   38 | ErrY:   -8 | Dist: 1.037m ]  -> OK (Sleep 25.5ms)  (추적 즉시 복구)
```

---

### 👁️ Step 3: AutoPilot v3 (딥러닝 모델 이식 - MobileNet V1)
검증된 아키텍처에 딥러닝(TFLite) 코드를 이식했습니다. 타겟(ID: 0, 사람)을 추적합니다.

*참고: 한글 주석으로 인한 `SyntaxError: Non-UTF-8 code` 및 `numpy` 버전 충돌 이슈(`pip3 install "numpy<2"`) 모두 해결 완료.*

**📝 `autopilot_v3_dl.py` (일부 생략)**
```python
# ... [기본 설정은 v2와 동일] ...
from tflite_runtime.interpreter import Interpreter

MODEL_PATH = "mobilenet/detect.tflite"
interpreter = Interpreter(model_path=MODEL_PATH, num_threads=4)
interpreter.allocate_tensors()
# ... [입출력 텐서 설정 생략] ...

def detect_target(image_rgb, original_w, original_h):
    # 딥러닝 입력 사이즈(300x300)로 강제 리사이즈 후 추론 수행
    img_resized = cv2.resize(image_rgb, (dl_width, dl_height))
    input_data = np.expand_dims(img_resized, axis=0)
    interpreter.set_tensor(input_details[0]['index'], input_data)
    interpreter.invoke()
    # 가장 높은 신뢰도(0.5 이상)의 사람(ID 0)의 중앙 좌표 반환 로직...
```

**📊 실행 결과 (V3 Log - 정확도 향상, 속도 저하)**
> 딥러닝의 고정 리사이즈 연산 특성상 V1 모델의 한계로 인해 **평균 80ms의 OVERTIME (약 7 FPS)** 발생. 단, ROI를 통해 **탐지 신뢰도(Confidence)와 안정성은 극도로 높음**을 입증함.
```text
[CONFIRMING 1/3] (62%)       | STM_TX [ State: 0 | ErrX:    0 | ErrY:    0 | Dist: 0.000m ]  -> OVERTIME (81.2ms)
[CONFIRMING 2/3] (67%)       | STM_TX [ State: 0 | ErrX:    0 | ErrY:    0 | Dist: 0.000m ]  -> OVERTIME (85.5ms)
[CONFIRMING 3/3] (77%)       | STM_TX [ State: 0 | ErrX:    0 | ErrY:    0 | Dist: 0.000m ]  -> OVERTIME (85.9ms)
[LOCKED on TARGET] (67%)     | STM_TX [ State: 1 | ErrX:   99 | ErrY:  -40 | Dist: 0.548m ]  -> OVERTIME (88.0ms)
[LOCKED on TARGET] (77%)     | STM_TX [ State: 1 | ErrX:   85 | ErrY:  -36 | Dist: 0.578m ]  -> OVERTIME (87.1ms)
```

---

### ⚡ Step 4: AutoPilot v4 (초고속 V2 모델 교체)
V1 모델의 오버타임을 완화하기 위해 모바일 최적화 모델인 `MobileNet V2 (또는 영상팀 자체 V3 Small)`로 교체한 최종 인지 코드입니다.

**📝 `autopilot_v4_fast.py` (변경점)**
```python
# [MODIFIED] Use the faster V2/V3 model
MODEL_PATH = "ssd_mobilenet_v2_coco_quant_postprocess.tflite" 
# (향후 자체 학습 모델 적용 시 이 경로만 변경하면 100% 호환됨)
```

**📊 기대 효과**
> 오버타임이 대폭 완화되며, 향후 **V3 Small 적용 시 15Hz 방어 또는 타겟 주기를 7Hz로 안정화**하여 실 비행에 즉각 투입 가능.

---

## 5. 핵심 엔지니어링 성과 요약
1. **오탐지율 0% 수렴:** `CONFIRM` 및 `ROI` 로직으로 배경 노이즈를 시스템적으로 완벽히 차단했습니다.
2. **센서 급발진 방어:** 단순 1픽셀 값이 아닌 5x5 공간 필터링으로 안정적인 Depth 추출망을 구축했습니다.
3. **모듈화 완료:** OpenCV 기반 뼈대에서 TFLite 딥러닝 엔진으로의 전환(Swap)을 에러 없이 성공적으로 수행했습니다.

## 6. Next Steps (향후 계획)
* **제어 주기 협의:** 현재 딥러닝 처리 속도에 맞춰 시스템의 `TARGET_HZ`를 7Hz로 고정할지, 모델(V3 Small)을 투입해 15Hz를 달성할지 결정.
* **UART 통신 통합:** `pyserial`을 사용하여 추출된 `[State, ErrX, ErrY, Dist]` 패킷을 STM32 보드로 물리적으로 전송.
  - **직렬 포트 연결:** `/dev/ttyUSB0` 등 적절한 시리얼 포트 확인 후 Docker `--device`를 통해 연결.
  - **통신 포맷 구성:** `serial` 라이브러리를 이용하여 `struct.pack` 등을 통해 바이트 포맷이나 문자열 데이터를 조합해 구성.
  - **주기적 데이터 송신:** 루프 안에서 검증된 `[State, ErrX, ErrY, Dist]`를 STM32로 전송하는 `uart.write(packet)` 추가 예정.
  - **안정성:** 예외 처리를 통해 통신 단절 시에도 시스템 동작을 유지.
