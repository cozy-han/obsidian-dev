# 미디어서버 비교: OvenMediaEngine vs MediaMTX

> 작성일: 2025-02-13
> 용도: UTM 프로젝트 미디어 서버 선정 참고

---

## 개요

| 항목     | **OvenMediaEngine (OME)**                                           | **MediaMTX**                                       |
| ------ | ------------------------------------------------------------------- | -------------------------------------------------- |
| 라이선스   | GPLv2                                                               | MIT                                                |
| 언어     | C++                                                                 | Go                                                 |
| 개발사    | AirenSoft (한국)                                                      | bluenviron (개인/커뮤니티)                               |
| 설계 철학  | **풀스택 미디어 서버**                                                      | **미디어 라우터/프록시**                                    |
| 의존성    | 있음 (빌드 필요 or Docker)                                                | **제로 디펜던시** (단일 바이너리)                              |
| GitHub | [OvenMediaEngine](https://github.com/OvenMediaLabs/OvenMediaEngine) | [MediaMTX](https://github.com/bluenviron/mediamtx) |

---

## 프로토콜 지원

| 프로토콜 | **OME** | **MediaMTX** |
|----------|---------|-------------|
| RTSP (입력) | O | O |
| RTSP (출력) | X | O |
| RTMP (입력) | O | O |
| RTMP (출력) | X (Push로 가능) | O |
| SRT | O | O |
| WebRTC (입력) | O | O |
| WebRTC (출력) | O | O |
| HLS (출력) | O | O |
| LL-HLS (출력) | O | O |
| MPEG-TS | O | O |
| RTP | X | O |
| MPEG-DASH | O | X |

---

## 핵심 기능 비교

| 기능                  | **OME**                   | **MediaMTX**          |
| ------------------- | ------------------------- | --------------------- |
| **트랜스코딩**           | O (내장, H.264/H.265/VP8)   | X                     |
| **GPU 가속 인코딩**      | O (NVENC)                 | X                     |
| **ABR (적응형 비트레이트)** | O (다중 해상도/비트레이트)          | X                     |
| **녹화**              | O (MP4/TS)                | O (fMP4)              |
| **썸네일 생성**          | O                         | X                     |
| **클러스터링**           | O (Origin-Edge 구조)        | X (단일 인스턴스)           |
| **REST API**        | O (풍부)                    | O (기본)                |
| **인증/접근제어**         | O (Webhook, SignedPolicy) | O (내부 인증, 외부 Webhook) |
| **모니터링/메트릭**        | O                         | O (Prometheus)        |
| **Push 재스트리밍**      | O (SRT/RTMP/MPEG-TS)      | O (FFmpeg 연동)         |
| **프록시/릴레이**         | X (본래 목적 아님)              | **O (핵심 기능)**         |

---

## 코덱 지원

| 코덱 | **OME** | **MediaMTX** |
|------|---------|-------------|
| H.264 | O (인코딩/디코딩) | O (패스스루) |
| H.265/HEVC | O (GPU 인코딩) | O (패스스루) |
| AV1 | X | O (패스스루, WebRTC 포함) |
| VP8 | O (인코딩) | O (패스스루) |
| VP9 | X | O (패스스루) |
| AAC | O (인코딩) | O (패스스루) |
| Opus | O (인코딩) | O (패스스루) |
| G.711 | X | O (패스스루) |
| AC-3 | X | O (패스스루) |

> [!note]
> MediaMTX는 코덱을 **변환하지 않고 그대로 전달**(패스스루)하기 때문에 더 많은 코덱을 "지원"한다.

---

## 확장성 & 성능

| 항목 | **OME** | **MediaMTX** |
|------|---------|-------------|
| 대규모 시청자 | **수십만 동시 시청** (Origin-Edge) | 중소규모 (단일 인스턴스) |
| 지연시간 | WebRTC ~0.5s / LL-HLS ~1-4s | WebRTC ~0.5s / LL-HLS ~1-3s |
| 리소스 사용량 | 높음 (트랜스코딩 시) | **매우 낮음** |
| Docker 지원 | O | O |
| 설정 복잡도 | 중간 (XML 설정) | **매우 간단** (YAML, 한 파일) |

---

## 설치 방법

### MediaMTX

다운로드 후 바로 실행 가능:

```bash
./mediamtx
```

### OvenMediaEngine

Docker 사용 권장:

```bash
docker run --name ome -d -e OME_HOST_IP=your_ip \
  -p 1935:1935 -p 3333:3333 -p 3478:3478 \
  -p 8080:8080 -p 10000-10009:10000-10009/udp \
  airensoft/ovenmediaengine:latest
```

---

## 선택 가이드

| 상황 | 추천 |
|------|------|
| IP 카메라 RTSP → WebRTC/HLS 단순 릴레이 | **MediaMTX** |
| 소규모 스트리밍, 빠른 프로토타이핑 | **MediaMTX** |
| IoT/임베디드 환경 (라즈베리파이 등) | **MediaMTX** |
| 트랜스코딩 + ABR이 필요한 라이브 방송 | **OvenMediaEngine** |
| 대규모 시청자 서비스 (수만~수십만) | **OvenMediaEngine** |
| Wowza 대체 (상용 수준 기능) | **OvenMediaEngine** |
| 두 서버 조합 사용 | MediaMTX(수집) → **OME(처리/배포)** |

---

## 한 줄 요약

> [!summary]
> **MediaMTX** = 가볍고 빠른 "미디어 스위치" (변환 없이 전달)
> **OvenMediaEngine** = 풀 기능 "미디어 팩토리" (변환 + 가공 + 대규모 배포)

---

## 용어 설명

### 트랜스코딩 (Transcoding)

영상/음성의 **코덱, 해상도, 비트레이트를 변환**하는 과정.

```
입력: 4K H.265 30Mbps 카메라 원본
         ↓ 트랜스코딩
출력: 1080p H.264 5Mbps (브라우저 재생용)
```

왜 필요한가:
- 카메라는 H.265로 보내는데 브라우저는 H.264만 지원할 때
- 원본 4K를 모바일용 720p로 줄여야 할 때
- 대역폭이 부족해서 비트레이트를 낮춰야 할 때

> [!info]
> MediaMTX는 트랜스코딩 없이 **받은 그대로 전달**(패스스루)만 한다. 코덱이 안 맞으면 재생 불가.

### GPU 가속 인코딩 (GPU Accelerated Encoding)

트랜스코딩은 **CPU를 많이 잡아먹는 작업**이다. 이걸 GPU(NVIDIA NVENC 등)에 넘겨서 처리 속도를 높이는 것.

```
CPU 인코딩:  1080p 1채널 → CPU 80% 점유
GPU 인코딩:  1080p 10채널 → CPU 10% + GPU 처리
```

- CPU만으로는 동시 다채널 트랜스코딩이 어려움
- NVIDIA GPU가 있으면 OME에서 NVENC을 활용해 **수십 채널 동시 변환** 가능

### ABR (Adaptive Bitrate Streaming, 적응형 비트레이트)

**시청자의 네트워크 상태에 따라 화질을 자동 전환**하는 기술. YouTube나 Netflix에서 화질이 자동으로 올라가거나 내려가는 것이 바로 ABR.

```
서버에서 하나의 원본을 여러 품질로 동시 트랜스코딩:

  원본 4K 30Mbps
       ├─→ 1080p  5Mbps   (Wi-Fi 시청자)
       ├─→  720p  2Mbps   (LTE 시청자)
       └─→  480p  800Kbps (3G 시청자)

시청자의 플레이어가 대역폭을 감지해서 자동으로 적절한 품질을 선택
```

- ABR이 없으면 → 대역폭 부족 시 **버퍼링** 발생
- ABR이 있으면 → 화질은 떨어져도 **끊김 없이** 재생

> [!important]
> ABR을 하려면 트랜스코딩이 필수 (여러 해상도를 동시에 만들어야 하므로)

### 클러스터링 (Clustering)

**여러 대의 서버를 묶어서 하나의 서비스처럼 동작**시키는 구조. OME는 **Origin-Edge** 방식을 사용한다.

```
                    ┌─── Edge 서버 1 ─── 시청자 1만명
카메라 → Origin 서버 ─┼─── Edge 서버 2 ─── 시청자 1만명
                    ├─── Edge 서버 3 ─── 시청자 1만명
                    └─── Edge 서버 N ─── ...
```

| 역할 | 설명 |
|------|------|
| **Origin** | 카메라 스트림을 수신하고 트랜스코딩 처리 |
| **Edge** | Origin에서 받은 스트림을 시청자에게 배포 |

- Origin 1대로는 시청자 수천 명이 한계
- Edge를 늘리면 **수십만 동시 시청자**까지 확장 가능
- Edge 서버 추가/제거가 자유로움 (Redis 기반 자동 연동)

> [!info]
> MediaMTX는 단일 인스턴스만 지원하므로, 시청자가 많아지면 서버 한 대의 한계에 부딪힌다.

### 용어 요약

| 용어 | 한 줄 설명 |
|------|-----------|
| 트랜스코딩 | 영상 포맷/해상도/비트레이트 변환 |
| GPU 가속 | 트랜스코딩을 GPU로 처리해서 성능 향상 |
| ABR | 네트워크 상태에 따라 화질 자동 조절 |
| 클러스터링 | 서버 여러 대로 대규모 시청자 지원 |

---

## MediaMTX 동시 스트림 수

소프트웨어 자체에는 하드 리밋이 없다. 병목은 MediaMTX가 아니라 **네트워크 대역폭과 하드웨어**.

### 실제 테스트 사례

| 환경 | 스트림 수 | 비고 |
|------|----------|------|
| i9 CPU, 128GB RAM, LACP NIC 본딩 | **2,800개 퍼블리시 + 2,800개 수신** (720p RTSP/WebRTC) | [GitHub Discussion #2719](https://github.com/bluenviron/mediamtx/discussions/2719) |
| 녹화 병행 시 | 퍼블리시 ~100개, 수신 ~500개 권장 | 디스크 I/O가 병목 |
| 500명 동시 시청 | 대역폭 부족 시 크래시 보고 | [Discussion #3024](https://github.com/bluenviron/mediamtx/discussions/3024) |

### 병목 요인

```
1. 네트워크 대역폭 ← 가장 큰 병목
   예) 720p 2Mbps × 100채널 = 200Mbps 필요

2. OS 파일 디스크립터 제한
   Linux 기본값 1024 → ulimit -n 65535로 상향 필요

3. 디스크 I/O (녹화 시)
   SSD/NVMe 권장, HDD는 수십 채널에서 한계

4. CPU/RAM
   패스스루만 하므로 부하 매우 적음 → 거의 병목이 안 됨
```

### 서버 사양별 가이드라인

| 서버 사양 | 예상 동시 스트림 (720p, 패스스루) |
|----------|-------------------------------|
| 소형 (라즈베리파이, 1Gbps) | ~50~100개 |
| 중형 (4코어, 16GB, 1Gbps) | ~200~500개 |
| 대형 (i9, 128GB, 10Gbps NIC) | ~2,000~3,000개 |

### 성능 튜닝 (mediamtx.yml)

```yaml
readBufferCount: 2048       # 읽기 버퍼 크기 (기본 2048)
writeQueueSize: 512         # 쓰기 큐 크기
udpMaxPayloadSize: 1472     # UDP 페이로드 크기
```

---

## MediaMTX 인코딩 변환

MediaMTX 자체는 **패스스루(passthrough) 전용**으로 내장 트랜스코딩 기능이 없다. 대신 **FFmpeg 연동**으로 변환 가능.

### FFmpeg 연동 변환 예시

| 변환 | FFmpeg 옵션 |
|------|------------|
| H.265 → H.264 | `-c:v libx264` |
| H.264 → VP8 | `-c:v libvpx` |
| H.264 → VP9 | `-c:v libvpx-vp9` |
| 4K → 720p | `-vf scale=1280:720` |
| AAC → Opus | `-c:a libopus` |

### 한계

| 항목 | 내용 |
|------|------|
| CPU 부하 | 채널당 FFmpeg 프로세스 1개 → 채널 많으면 CPU 과부하 |
| GPU 가속 | FFmpeg에서 NVENC 사용 가능하지만 직접 설정 필요 |
| 관리 | 채널마다 수동 설정, 자동화 없음 |

> [!warning]
> 채널이 많고 트랜스코딩이 핵심이라면 **OvenMediaEngine**(내장 트랜스코딩 + GPU 가속 + 자동 ABR)이 더 적합하다.

---

## 브라우저 미지원 코덱 처리

MediaMTX는 패스스루 전용이므로, 브라우저가 코덱을 지원하지 않으면 재생 불가. **FFmpeg를 연동**해서 해결한다.

### 문제 상황

```
카메라 (H.265/HEVC) → MediaMTX → 브라우저 (WebRTC)
                                    ❌ 재생 불가!
                                    (Chrome은 WebRTC에서 H.265 미지원)
```

### 해결: MediaMTX + FFmpeg 트랜스코딩

MediaMTX 설정 파일에서 **runOnReady**로 FFmpeg를 자동 실행:

```yaml
paths:
  camera1:
    source: rtsp://카메라IP/stream
    runOnReady: >
      ffmpeg -i rtsp://localhost:8554/camera1
      -c:v libx264 -preset ultrafast -tune zerolatency
      -c:a aac
      -f rtsp rtsp://localhost:8554/camera1_h264
    runOnReadyRestart: yes

  camera1_h264:
    # 변환된 H.264 스트림 (브라우저에서 재생 가능)
```

```
카메라 (H.265) → MediaMTX [camera1] → FFmpeg 트랜스코딩
                                          ↓
                 MediaMTX [camera1_h264] → 브라우저 (WebRTC/HLS)
                                          ✅ H.264로 재생 가능
```

### 브라우저별 코덱 지원 현황

| 코덱 | Chrome | Firefox | Safari | WebRTC 지원 |
|------|--------|---------|--------|-----------|
| H.264 | O | O | O | **O (가장 안전)** |
| H.265/HEVC | O (MSE) | X | O (MSE) | **X** |
| VP8 | O | O | O | O |
| VP9 | O | O | X | O |
| AV1 | O | O | O (17+) | O (최신) |

> [!tip]
> 브라우저 WebRTC로 재생하려면 **H.264 또는 VP8**이 가장 안전한 선택.

### 해결 방법별 비교

| 방법 | 장점 | 단점 |
|------|------|------|
| **MediaMTX + FFmpeg** | 간단, 유연 | FFmpeg가 CPU 소모, 채널당 프로세스 필요 |
| **OvenMediaEngine 사용** | 내장 트랜스코딩, GPU 가속, ABR | 설정 복잡, 리소스 더 필요 |
| **카메라 설정 변경** | 가장 간단, 서버 부하 없음 | 카메라가 H.264 출력을 지원해야 함 |

> [!important]
> **실무 권장 순서:**
> 1. 카메라가 H.264 출력을 지원하는지 먼저 확인 (듀얼 스트림 설정 가능한 카메라가 많음)
> 2. 카메라 변경 불가 → MediaMTX + FFmpeg 조합
> 3. 채널 많아서 CPU 부하 감당 불가 → OvenMediaEngine(GPU 트랜스코딩)으로 전환

---

## FFmpeg

커맨드라인에서 실행하는 **오픈소스 만능 영상/음성 변환기**. 미디어 분야의 사실상 표준(de facto standard).

### 할 수 있는 것

```
영상 변환:    MP4 → AVI, MKV → MP4, H.265 → H.264
해상도 변경:  4K → 1080p → 720p
코덱 변환:    H.265 → H.264, AAC → Opus
스트리밍:     RTSP → RTMP, 파일 → HLS
녹화:        스트림 → MP4 파일 저장
자르기/합치기: 영상 편집 (트림, 병합)
추출:        영상에서 오디오만 뽑기, 썸네일 생성
```

### 사용 예시

```bash
# H.265 영상을 H.264로 변환
ffmpeg -i input_h265.mp4 -c:v libx264 output_h264.mp4

# RTSP 카메라 스트림을 H.264로 변환하며 다시 RTSP로 출력
ffmpeg -i rtsp://카메라IP/stream \
  -c:v libx264 -preset ultrafast \
  -f rtsp rtsp://localhost:8554/output

# 영상에서 오디오만 추출
ffmpeg -i video.mp4 -vn -c:a mp3 audio.mp3
```

### MediaMTX와의 관계

```
MediaMTX 단독:     카메라 → [패스스루] → 브라우저 (코덱 변환 불가)
MediaMTX + FFmpeg:  카메라(H.265) → FFmpeg(H.264 변환) → MediaMTX → 브라우저 (변환 가능)
```

| 항목 | 내용 |
|------|------|
| 라이선스 | LGPL / GPL |
| 사용 방식 | CLI (커맨드라인) |
| 설치 (macOS) | `brew install ffmpeg` |
| 설치 (Linux) | `apt install ffmpeg` |

---

## 기타 오픈소스 미디어 서버

| 이름 | 라이선스 | 특징 | 비고 |
|------|---------|------|------|
| Ant Media Server CE | Apache 2.0 | WebRTC 저지연 | 핵심 기능은 Enterprise(유료)에만 있음 |
| Janus Gateway | GPLv3 | WebRTC 전문, 양방향 통신 | 화상회의/SFU 용도에 적합 |
| MistServer | AGPL | 경량, 다중 프로토콜 | 낮은 하드웨어 사양에서 동작 |
| Red5 | Apache 2.0 | Java 기반, RTMP 중심 | 오래된 프로젝트, 안정적 |
| Kaltura | AGPL | 교육기관 특화 | 플러그인 풍부 |

---

## 참고 링크

- [OvenMediaEngine 공식 문서](https://docs.ovenmediaengine.com)
- [OvenMediaEngine GitHub](https://github.com/OvenMediaLabs/OvenMediaEngine)
- [MediaMTX GitHub](https://github.com/bluenviron/mediamtx)
- [MediaMTX 프로토콜 상세 (DeepWiki)](https://deepwiki.com/bluenviron/mediamtx/1.2-supported-protocols)
- [OME 성능 튜닝 가이드](https://docs.ovenmediaengine.com/0.17.3/performance-tuning)
- [OME 리뷰 - Streaming Media](https://www.streamingmedia.com/Articles/Editorial/Featured-Articles/Review-AirenSoft-OvenMediaEngine-165156.aspx)
