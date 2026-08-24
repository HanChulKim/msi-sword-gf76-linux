# NVIDIA 565 → 595 업그레이드

`rainnam-msi`의 NVIDIA 드라이버를 565.57.01에서 595.84로 올린 기록. 목적은 버전 올리기 자체가 아니라, 20.04 시절 저장소에 묶여 있던 드라이버를 24.04 공식 저장소로 되돌리고 그 부작용으로 깨져 있던 GPU 컨테이너 경로를 정상화하는 것이었다.

| | |
|---|---|
| **일자** | 2026-08-24 |
| **호스트** | rainnam-msi.local |
| **OS** | Ubuntu 24.04.4 LTS / 커널 6.8.0-138-generic |
| **GPU** | NVIDIA RTX 4060 Max-Q (AD107M) |
| **이전** | 565.57.01-0ubuntu1 (third-party) |
| **이후** | 595.84-0ubuntu0.24.04.1 (noble 공식) |
| **상태** | 완료 |

> [!IMPORTANT]
> 부수 효과가 본 목적만큼 중요했다. 드라이버 교체로 `nvidia-ctk`의 세그폴트가 사라져, Docker 29의 CDI 경로가 정상 동작하게 됐다. 이전 작업에서 임시로 걸어둔 `mode = "legacy"` 우회를 걷어내고 `auto`로 되돌렸다.

---

## 배경 — 왜 올렸나

세 가지가 겹쳐 있었다.

**출처가 20.04 저장소였다.** 565.57.01은 `developer.download.nvidia.com/.../ubuntu2004/`에서 설치된 것이라 `ubuntu-drivers`에 `third-party non-free`로 잡혔다. 그 저장소는 dist-upgrade 때 비활성화됐으므로, 드라이버는 업데이트를 받을 수 없는 고아 상태였다.

**빌드가 오래됐다.** 565.57.01은 2024년 10월 빌드로, 실행 중인 커널(2026년 7월 빌드)과 1년 9개월 차이가 났다. 2026-08-21의 X 서버 장애 때 이미 DKMS 모듈이 로드되지 않는 문제가 있었고([X 복구 기록](x-server-recovery-2026-08-21.md) 참조), 강제 재빌드로 넘겼지만 다음 커널 업데이트 때 재발할 여지가 남아 있었다.

**GPU 컨테이너가 우회 모드였다.** Docker를 focal에서 noble로 올린 뒤 `--gpus all`이 실패했다. Docker 29는 GPU를 CDI로 해결하는데 `nvidia-ctk cdi generate`가 `nvSandboxUtilsShutdown`에서 세그폴트로 죽어 CDI 스펙을 만들 수 없었다. nvidia-container-toolkit 1.19.0과 드라이버 565의 `libnvidia-sandboxutils` 비호환이었다. 당시에는 툴킷을 `legacy` 모드로 고정해 넘겼다.

---

## 버전 선택

`ubuntu-drivers devices`가 이 GPU에 대해 제시한 후보는 아래와 같았다.

| 버전 | 후보 | 비고 |
|---|---|---|
| `nvidia-driver-610-open` | 610.43.02-0ubuntu0.24.04.1 | 최신 |
| `nvidia-driver-595-open` | 595.84-0ubuntu0.24.04.1 | **recommended** |
| `nvidia-driver-580-open` | 580.173.02-0ubuntu0.24.04.1 | |
| `nvidia-driver-565` | 565.57.01-0ubuntu1 | 설치되어 있던 것 (third-party) |

**595-open을 택했다.** Ubuntu가 이 하드웨어에 recommended로 지정한 버전이고, 565에서 이미 충분히 큰 도약이며, 복구 직후의 CUDA 작업 머신이라 배포판 검증을 거친 쪽이 안전하다고 판단했다. 610도 설치는 가능하다.

`-open` 접미사는 오픈 커널 모듈을 뜻한다. RTX 4060(Ada)은 Turing 이후 세대라 오픈 모듈이 정식 지원되며, NVIDIA와 Ubuntu 모두 이쪽을 기본으로 권한다. 설치 후 `modinfo nvidia`의 `license: Dual MIT/GPL`로 확인된다.

---

## 절차

```bash
# 사전 확인 — 후보 버전과 CUDA 스택 영향
ubuntu-drivers devices
sudo apt-get install -s nvidia-driver-595-open   # 시뮬레이션

# 설치 (565 패키지 20개 제거 + 595 패키지 24개 설치)
sudo apt-get install -y nvidia-driver-595-open

# 재부팅 — 커널 모듈 교체에 필수
sudo systemctl reboot
```

시뮬레이션에서 제거 대상은 `nvidia-*-565` / `libnvidia-*-565` 계열뿐이었다. CUDA 툴킷, cuDNN, nvidia-container-toolkit, sdkmanager, vpi3는 모두 유지된다는 것을 설치 전에 확인했다.

재부팅 전 구간에서는 사용자 영역이 595인데 커널 모듈이 565라 `nvidia-smi`가 `Driver/library version mismatch`로 실패한다. 정상적인 중간 상태이며 재부팅으로 해소된다.

### 재부팅 후 — CDI 복원

```bash
# 세그폴트 없이 생성되는지 먼저 확인
sudo nvidia-ctk cdi generate --output=/tmp/cdi-test.yaml

# 정식 경로에 생성
sudo mkdir -p /etc/cdi
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
sudo nvidia-ctk cdi list

# 툴킷을 legacy 우회에서 auto 로 되돌림
sudo sed -i 's|^mode = "legacy"|mode = "auto"|' /etc/nvidia-container-runtime/config.toml
```

---

## 검증

| 항목 | 이전 | 이후 |
|---|---|---|
| 드라이버 | 565.57.01 | **595.84** |
| 패키지 출처 | ubuntu2004 CUDA 저장소 (third-party) | noble-updates / noble-security |
| 커널 모듈 | proprietary | **오픈** (`Dual MIT/GPL`) |
| DKMS | `nvidia/565.57.01 ... installed` | `nvidia/595.84 ... installed` |
| 드라이버 지원 CUDA | 12.7 | **13.2** |
| `nvidia-ctk cdi generate` | 세그폴트 (`nvSandboxUtilsShutdown`) | **정상** (spec v0.5.0) |
| 툴킷 모드 | `legacy` (우회) | **`auto`** |
| CDI 장치 | 0개 | **3개** (`gpu=0`, `gpu=<UUID>`, `gpu=all`) |
| Intel iGPU | `i915` | `i915` (변화 없음) |
| NVIDIA 바인딩 | `nvidia` | `nvidia` |
| Xorg 가속 | `glamor on Mesa Intel (RPL-P)` | 동일 |
| 디스플레이 | `eDP-1 connected 1920x1080` | 동일 |
| PRIME 오프로드 | RTX 4060 | RTX 4060 |
| CUDA 툴킷 | 12.6 (V12.6.85) | 12.6 (변화 없음) |

GPU 컨테이너는 두 경로 모두 확인했다.

```bash
docker run --rm --gpus all ubuntu:20.04 nvidia-smi -L
docker run --rm --device nvidia.com/gpu=all ubuntu:20.04 nvidia-smi -L
```

둘 다 `GPU 0: NVIDIA GeForce RTX 4060 Laptop GPU (UUID: GPU-433e8608-...)`를 반환했다. 앞은 기존 문법이 CDI로 해결되는 경로, 뒤는 CDI를 직접 지정하는 경로다.

---

## 남은 사항

**`nvidia-firmware-565-565.57.01`이 남아 있다.** 제거되지 않은 잔여 패키지다. 동작에 영향은 없다.

> [!WARNING]
> **`sudo apt autoremove`를 그대로 실행하지 말 것.** 이 머신에서 autoremove 대상에 TensorRT 패키지(`libnvinfer*` 10.13.0.35+cuda12.9)가 포함된다. 시뮬레이션(`apt-get -s autoremove`)으로 목록을 먼저 확인하고, 필요한 것은 `apt-mark manual`로 보호한 뒤 실행해야 한다.

**드라이버가 보고하는 CUDA는 13.2, 설치된 툴킷은 12.6이다.** 불일치가 아니라 정상이다. 드라이버는 자신이 지원하는 최대 CUDA 버전을 보고하며, 하위 호환이므로 12.6 툴킷으로 빌드한 코드는 그대로 동작한다. 툴킷을 올릴 필요는 없다.

**610은 여전히 선택지로 남아 있다.** 필요해지면 `sudo apt install nvidia-driver-610-open`으로 올릴 수 있다.

---

## 후속 조치 — CUDA 저장소 복원 (같은 날 16:00)

드라이버 업그레이드 뒤 TensorRT 패키지가 `dpkg/status` 외에 어떤 저장소도 출처로 갖지 못하는 고아 상태였다. 재설치도 업그레이드도 불가능한 상태였고, 근본 원인은 `cuda-keyring` 패키지가 20.04 시절에 설치된 것이었기 때문이다.

**핵심 오해 정리** — `cuda-keyring`은 이름과 달리 GPG 키만 설치하는 패키지가 아니다. `dpkg -L cuda-keyring`을 보면:

```
/usr/share/keyrings/cuda-archive-keyring.gpg        ← GPG 서명 키
/etc/apt/preferences.d/cuda-repository-pin-600      ← apt pin
/etc/apt/sources.list.d/cuda-ubuntu2004-x86_64.list ← 저장소 URL (배포판 이름 박힘)
```

세 번째 파일이 문제였다. **NVIDIA는 배포판마다 별도의 `.deb`를 배포하며**, 파일 이름은 모두 `cuda-keyring_1.1-1_all.deb`로 같지만 안의 `.list` URL이 다르다. 그래서 "apt로 업그레이드"할 대상이 아니라 "24.04용 `.deb`로 교체"해야 한다.

```bash
# 1. 기존 20.04 keyring 제거 (연관 백업 파일도 정리)
sudo apt purge cuda-keyring
sudo rm -f /etc/apt/sources.list.d/cuda-ubuntu2004-x86_64.list.distUpgrade
sudo rm -f /etc/apt/sources.list.d/cuda-ubuntu2004-x86_64.list.save

# 2. 24.04 keyring 다운로드 및 설치
curl -fsSLo /tmp/cuda-keyring.deb \
  https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i /tmp/cuda-keyring.deb
rm /tmp/cuda-keyring.deb

# 3. apt 새로고침
sudo apt update
```

새 파일이 `/etc/apt/sources.list.d/cuda-ubuntu2404-x86_64.list`에 생성되고, apt가 `developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/`에서 InRelease를 받아온다.

### 검증 — 저장소가 붙자 TensorRT/CUDA 패키지가 다시 잡힘

```
libnvinfer10:  10.16.1.11-1+cuda13.2   (이전엔 dpkg/status만; 이제 저장소 출처)
tensorrt:      11.2.1.2-1+cuda13.3     (신규 후보)
cuda-toolkit-12-6: 12.6.3-1 (installed) ← 유지
```

> [!NOTE]
> 저장소만 활성화했을 뿐 이번 작업에서 TensorRT를 **업그레이드하지는 않았다.** 필요 시 `sudo apt install --only-upgrade libnvinfer10 libnvinfer-dev ...`로 갱신할 수 있고, `tensorrt-dev` 메타패키지를 새로 설치하면 이후 자동 관리도 붙는다. 다만 이 시점부터 커널·드라이버 업데이트 때 CUDA 저장소도 함께 후보에 오르므로, 다음 `apt upgrade` 전에는 목록을 한 번 훑어보는 편이 좋다.

`libnvinfer10`이 이제 저장소 출처를 갖는다는 것은, "남은 사항"의 autoremove 경고가 여전히 유효하되 이유가 하나 바뀌었다는 뜻이다 — 지금 지워도 원한다면 apt로 되깔 수 있게 됐다.

---

## 관련 문서

- [X 복구 기록 (2026-08-21)](x-server-recovery-2026-08-21.md) — 이 드라이버가 처음 문제가 됐던 장애

---

*업그레이드 2026-08-24 13:0x–13:48 KST · 후속 조치 16:00 KST · rainnam-msi.local · 상태: 완료*
