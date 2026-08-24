# 로그인 화면이 멈춘 이유는 마우스가 아니라 GPU였다

MSI 노트북 `rainnam-msi`의 GNOME 로그인 화면 장애 진단 및 복구 기록. 증상은 입력 장치 문제처럼 보였지만, 실제로는 GPU 드라이버가 하나도 바인딩되지 않아 X가 소프트웨어 렌더링으로 떨어진 것이 원인이었다.

| | |
|---|---|
| **일자** | 2026-08-21 |
| **호스트** | rainnam-msi.local |
| **OS** | Ubuntu 24.04.4 LTS |
| **커널** | 6.8.0-138-generic |
| **iGPU** | Intel Raptor Lake-P UHD Graphics |
| **dGPU** | NVIDIA RTX 4060 Max-Q |
| **상태** | 해결됨 |

> [!IMPORTANT]
> 근본 원인은 `/etc/default/grub`의 `i915.modeset=0`이었다. 여기에 낡은 `xorg.conf`와 로드에 실패하는 `nvidia.ko`가 겹쳐 있었다. 셋을 모두 정리하고 재부팅한 뒤 콘솔 로그인에 성공했으며, Intel iGPU 하드웨어 가속과 NVIDIA PRIME 오프로드가 모두 정상 동작하는 것을 확인했다.

---

## 증상 — 보고된 현상과, 그것이 가리키지 않았던 것

- 콘솔과 SSH는 정상. 시스템 자체는 살아 있었다.
- GNOME 로그인 화면에서 **마우스 커서가 보이지 않음**.
- 키보드로 로그인을 시도했으나 반응이 없어 **실패한 것처럼 보임**.
- `gdm.service`는 `active (running)`, `graphical.target`도 `active`. 서비스 계층에는 이상이 없었다.

입력 장치는 처음부터 정상이었다. Xorg 로그를 보면 libinput이 AT 키보드, 터치패드 2종, Logitech USB 마우스를 모두 잡고 있었다. 커서가 없었던 것은 입력을 못 받아서가 아니라 **커서를 그릴 하드웨어가 없었기 때문**이고, 키 입력이 안 먹힌 것처럼 보인 것은 화면 갱신이 사실상 멈춰 있었기 때문이다.

---

## 진단 — X가 하드웨어에서 소프트웨어로 떨어진 경로

아래는 `/var/lib/gdm3/.local/share/xorg/Xorg.0.log`에 남은 실제 폴백 순서다. 각 단계가 다음 단계를 강제하므로 순서 자체가 정보다.

### 1. Intel 드라이버가 i915를 기다리다 포기

```
intel: waited 2020 ms for i915.ko driver to load
```

`i915.modeset=0` 때문에 모듈은 메모리에 있으나 어떤 장치에도 바인딩되지 않는다.

### 2. NVIDIA DRM 장치 열기 실패

```
(EE) [drm] Failed to open DRM device for pci:0000:01:00.0: -19
```

`-19`는 ENODEV. nvidia 커널 모듈이 아예 로드되지 않은 상태였다.

### 3. 쓸 수 있는 GPU가 없다고 판정

```
(EE) No devices detected.
```

여기서 `xorg.conf`에 적힌 장치 설정이 전부 무효가 되고 자동 설정으로 넘어간다.

### 4. EFI 프레임버퍼로 폴백

```
(==) Matched modesetting as autoconfigured driver 0
```

`/dev/dri/card0`의 `DRIVER`는 `simple-framebuffer`. 진짜 GPU가 아니라 펌웨어가 남긴 프레임버퍼다.

### 5. 소프트웨어 래스터라이저로 렌더링

```
glamor X acceleration enabled on llvmpipe (LLVM 20.1.2, 256 bits)
```

llvmpipe에는 하드웨어 커서 평면이 없다. **여기서 마우스가 사라진다.**

> [!NOTE]
> **교차 확인** — `gnome-shell`도 같은 이야기를 하고 있었다: `failed to load driver: simpledrm`, `glx: failed to create dri3 screen`. 그리고 `lspci -k` 출력의 두 GPU 어디에도 `Kernel driver in use` 줄이 없었다. 이것이 결정적 증거였다.

---

## 원인 — 겹겹이 쌓인 수동 우회 설정 세 건

셋은 서로 다른 시점에 각각 추가된 별개의 문제이며 순서가 없다. 다만 화면을 죽인 직접적 원인은 첫 번째 하나다.

### `/etc/default/grub`

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.modeset=0 nvidia-drm.modeset=0"
```

**이 한 줄이 직접적인 원인이다.** `i915.modeset=0`은 내장 그래픽의 커널 모드 설정을 끈다. 디스플레이를 담당하는 것은 Intel iGPU이므로, 이 순간 노트북 패널을 구동할 드라이버가 사라진다.

다만 이것은 **Ubuntu 20.04 시절에 넣은 설정**이며, 당시 환경에서는 성립했다. NVIDIA가 화면을 담당하는 구성이었다면 iGPU를 꺼두는 것이 의도된 선택일 수 있다. 문제는 이 설정이 dist-upgrade를 타고 24.04 / 커널 6.8까지 그대로 따라온 것이다. `dpkg -l`에 `5.15.0-*~20.04.1` 커널 잔재가 남아 있는 것이 그 업그레이드 이력을 보여준다.

| | |
|---|---|
| 유래 | Ubuntu 20.04 환경에 맞춘 설정. dist-upgrade로 잔존 |
| 조치 | 두 파라미터 제거 후 `update-grub` |
| 백업 | `/etc/default/grub.bak.20260821-173456` |

### `/etc/X11/xorg.conf`

`Xorg -configure`로 생성된 낡은 파일이 남아 실제로 로드되고 있었다 (`(==) Using config file: "/etc/X11/xorg.conf"`). 내용은 현재 하드웨어와 맞지 않았다.

- `Card0` → `Driver "intel"`. xf86-video-intel은 Raptor Lake를 지원하지 않는다.
- `Card1` → `Driver "nouveau"`. 그런데 nouveau는 `/etc/modprobe.d/`에서 블랙리스트되어 있다.
- `InputDevice`가 레거시 `kbd` / `mouse` 드라이버 지정. 해당 `.so` 파일은 시스템에 존재하지도 않는다.

i915만 되살렸다면 이 설정이 다시 X를 깨뜨렸을 것이다. 이번 부팅에서는 GPU가 아예 없어 자동 설정으로 넘어가는 바람에 가려져 있었다.

| | |
|---|---|
| 조치 | 비활성화, 자동 설정에 위임 |
| 백업 | `/etc/X11/xorg.conf.disabled-20260821-173830` |

### `/lib/modules/6.8.0-138-generic/updates/dkms/nvidia.ko`

```
nvidia: disagrees about version of symbol sme_me_mask
nvidia: Unknown symbol sme_me_mask (err -22)
```

까다로운 지점은 **모든 정황 증거가 정상이라고 말했다**는 것이다. `dkms status`는 해당 커널에 `installed`로 보고했고, `vermagic`도 일치했으며, 모듈 파일의 심볼 CRC `0x8a35b432`는 헤더의 `Module.symvers` 값과 정확히 같았다. `dpkg -V`로 커널·헤더·드라이버 패키지를 검증해도 변조가 없었다. 그럼에도 실행 중인 커널은 로드를 거부했다.

설치된 파일이 비압축 `.ko`인 반면 정상 빌드 산출물은 `.ko.zst`라는 점이 단서였다. 이전 설치 과정에서 남은 낡은 사본이 자리를 차지하고 있었던 것으로 **추정**된다. 강제 재빌드로 해결됐다.

| | |
|---|---|
| 조치 | `dkms build/install --force` 후 `update-initramfs -u` |
| 결과 | `nvidia-smi` 정상, RTX 4060 인식 |

> [!NOTE]
> **왜 이렇게 됐나** — 셋 다 과거 환경에 맞춰 넣은 설정이 그대로 남은 것이다. `i915.modeset=0`은 20.04 시절 설정이고, `xorg.conf`도 2025년 1월 파일이다. 하드웨어와 커널은 계속 바뀌었는데 설정만 따라오지 못했다.
>
> NVIDIA가 화면을 담당하는 동안에는 iGPU가 꺼져 있어도 티가 나지 않는다. 그러다 nvidia 모듈이 로드에 실패하는 순간, 넘어갈 곳이 없어진다 — 폴백이 되어야 할 Intel이 이미 꺼져 있기 때문이다. **고장 하나가 아니라, 안전망을 미리 걷어둔 상태에서 고장이 난 것**이 이번 장애의 구조다.

---

## 조치 — 적용한 변경과 실행 순서

`update-initramfs`는 빠뜨리기 쉬운 단계였다. 모듈 재빌드 직후에도 initramfs 안에는 로드에 실패하던 예전 `nvidia.ko`가 그대로 들어 있었다.

```bash
# 1. GRUB — 백업 후 modeset 파라미터 제거
sudo cp -a /etc/default/grub /etc/default/grub.bak.$(date +%Y%m%d-%H%M%S)
sudo sed -i 's|^GRUB_CMDLINE_LINUX_DEFAULT=.*|GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"|' /etc/default/grub
sudo update-grub

# 2. NVIDIA — 강제 재빌드
sudo dkms build   nvidia/565.57.01 -k 6.8.0-138-generic --force
sudo dkms install nvidia/565.57.01 -k 6.8.0-138-generic --force

# 3. xorg.conf — 비활성화하고 자동 설정에 위임
sudo mv /etc/X11/xorg.conf /etc/X11/xorg.conf.disabled-$(date +%Y%m%d-%H%M%S)

# 4. initramfs — 낡은 nvidia.ko가 남아 있으므로 재생성
sudo update-initramfs -u -k 6.8.0-138-generic

# 5. 재부팅
sudo systemctl reboot
```

---

## 검증 — 재부팅 전후 상태

| 항목 | 이전 | 현재 |
|---|---|---|
| 커널 파라미터 | `i915.modeset=0 nvidia-drm.modeset=0` | `quiet splash` |
| Intel iGPU | 바인딩 없음 | `driver in use: i915` |
| NVIDIA dGPU | 모듈 로드 실패 | `driver in use: nvidia` |
| `/dev/dri` | `card0 = simple-framebuffer` | `card1` · `card2` · `renderD128` · `renderD129` |
| Xorg 가속 | `glamor on llvmpipe` | `glamor on Mesa Intel (RPL-P)` |
| 디스플레이 | 출력 없음 | `eDP-1 connected, 1920x1080` |
| OpenGL | — | `Mesa Intel(R) Graphics (RPL-P)` |
| PRIME 오프로드 | — | `GeForce RTX 4060 Laptop GPU` |
| 포인터 | 커서 없음 | 마우스 · 터치패드 2종 연결 |
| 콘솔 로그인 | 실패 | 성공 (`seat0` / `tty2` / `x11`) |

PRIME 프로파일은 `on-demand`를 유지했다. 데스크톱은 Intel iGPU가 그리고, NVIDIA는 오프로드와 CUDA 작업에만 쓰인다.

---

## 무시해도 되는 로그

정상 동작 중에도 계속 나오는 메시지들이다.

```
[drm:nv_drm_master_set [nvidia_drm]] *ERROR* Failed to grab modeset ownership
```

on-demand PRIME에서는 정상이다. Xorg가 `10-nvidia.conf`의 OutputClass 때문에 nvidia를 먼저 시도했다가, 디스플레이를 구동하지 않으므로 DRM master 획득에 실패하고 Intel로 넘어가면서 남기는 메시지다.

```
xe 0000:00:02.0: Your graphics device a7a8 is not officially supported
```

역시 정상이다. 신규 `xe` 드라이버가 이 iGPU를 `i915`에게 올바르게 양보하고 있다는 뜻이다.

---

## 재발 시 — GUI가 다시 깨지면 이 순서로 확인

이번 건의 교훈은 **증상이 가리키는 계층을 믿지 말 것**이다. 입력 장치를 의심하기 전에 GPU가 바인딩되어 있는지부터 본다.

1. `cat /proc/cmdline` — `modeset=0`류 파라미터가 다시 들어와 있는지 확인한다.
2. `lspci -k | grep -A3 VGA` — `Kernel driver in use` 줄이 두 GPU 모두에 있어야 한다. 없으면 여기서 원인이 끝난다.
3. `ls -l /dev/dri/` — `simple-framebuffer`가 보이면 드라이버 미바인딩 상태다.
4. `ls /etc/X11/xorg.conf` — 되살아났는지 확인. 이 머신에서는 없는 편이 맞다.
5. `grep llvmpipe ~/.local/share/xorg/Xorg.0.log` — 걸리면 소프트웨어 렌더링으로 떨어진 것이다.

---

## 권고

NVIDIA 565.57.01은 2024년 10월 빌드로, 현재 커널(2026년 7월 빌드) 대비 상당히 오래됐다. 지금은 정상 동작하지만 다음 커널 업데이트 때 같은 DKMS 문제가 재발할 여지가 있으므로 최신 드라이버 브랜치로의 이전을 검토할 만하다.

---

*진단 · 복구 2026-08-21 17:22–17:43 KST · rainnam-msi.local · 상태: 해결*
