# Kiosk Capture Client and Camera Control

**Environment** · Unattended kiosk at a venue, no engineer present
**Hardware** · A full-frame mirrorless camera driven over USB by the manufacturer's remote-control SDK
**Stakes** · The customer is standing there. There is one chance to capture.

## Context

The kiosk runs the customer-facing flow: preview, capture, upload, and result. It drives a real camera rather than a webcam, because the output is the product.

The camera is controlled through the manufacturer's remote SDK, which is a native C++ library. The desktop application is C#. The venue has no technical staff, and updates have to reach machines nobody will log into.

## Problem

**The SDK is C++ and the app is not.** Native remote-control SDKs are C++ libraries with their own initialization lifecycle, their own callback threading model, and their own opinions about process state. Binding that directly into a desktop UI application means a fault in vendor code takes the interface down with it, in front of a customer.

**Capture failures are not recoverable by the user.** There is no operator to reconnect a camera or restart an application. Whatever the app does when the SDK fails, it has to do by itself.

**Duplicate captures are a real failure mode.** A customer presses the button twice. A retry fires after a request actually succeeded. Both produce a second session — and downstream, a second paid generation job.

**The application had to change platform.** The flow originally existed as a macOS application and had to become Windows-first for the kiosk hardware.

**Vendor SDK binaries cannot go in the repository.** They are redistributables with their own licensing, and they are large.

## Approach

**The camera lives in a separate process.** A dedicated agent owns the SDK — initialization, connection, live property reads, shutter, and file transfer — and communicates with the application over a defined protocol. The application never links the vendor library.

This is the central decision. It costs a process boundary and a protocol, and it buys three things: a vendor SDK crash takes down the agent rather than the customer-facing UI; the agent can be restarted and reconnected without restarting the session; and the C++ and C# sides can be built, tested, and reasoned about independently. For unattended hardware, isolating the component most likely to fail is worth more than the convenience of an in-process call.

**The protocol between app and agent is a shared, explicit contract.** Kept as its own artifact rather than implied by whichever side was written first, because the two are built with different toolchains and cannot share types.

**Session state is local and durable.** Capture sessions, their retry state, and their upload progress persist on the machine. The kiosk is a client of a remote backend over a venue network — an unreliable one — so an interrupted upload has to resume rather than lose the capture that has already happened.

**Duplicate protection lives at the client.** Each session is identified locally before it is sent, so a double press or an ambiguous retry maps to the same session rather than creating a new one. This matters more here than in most clients: a duplicate session at the kiosk becomes a duplicate paid generation job in [the backend queue](02-job-queue-for-paid-work.md). Deduplicating at the point where the intent originates is cheaper and more reliable than trying to reconcile it afterwards.

**Installation and updates were treated as product surface.** The kiosk software ships as an installer with a documented update path, written up for the client's operators rather than for engineers. Software that has to run on machines nobody will log into needs an update story that does not involve logging into them.

**Vendor SDK binaries stay out of the repository, with the expected path documented.** The build looks for them in an ignored local directory, and setup documentation states where they belong. Camera access credentials live in an ignored environment file rather than in code or shell history.

## Verification

The camera integration was validated end to end before the application was built on top of it: connect, read live properties from the body, trigger the shutter remotely, and transfer the resulting image to the host. Proving the full path on real hardware first meant the application was written against known behaviour instead of against the SDK documentation's description of it — which, with native camera SDKs, is not reliably the same thing.

---

## 한국어 요약

**환경** — 현장의 무인 키오스크, 엔지니어 없음
**하드웨어** — 제조사 원격 제어 SDK로 USB를 통해 구동되는 풀프레임 미러리스 카메라
**리스크** — 고객이 앞에 서 있고, **촬영 기회는 한 번**

키오스크가 고객 대면 흐름(프리뷰 → 촬영 → 업로드 → 결과)을 돌립니다. 웹캠이 아니라 실제 카메라를 제어합니다. **결과물 자체가 제품**이기 때문입니다. 카메라는 제조사 원격 SDK로 제어되는데 이는 네이티브 C++ 라이브러리이고, 데스크톱 애플리케이션은 C#입니다. 현장에 기술 인력이 없고, 업데이트는 **아무도 로그인하지 않을 장비**에 도달해야 합니다.

**어려웠던 지점**

- **SDK는 C++이고 앱은 아닙니다.** 네이티브 원격 제어 SDK는 자체 초기화 생애주기, 자체 콜백 스레딩 모델, 프로세스 상태에 대한 자체 의견을 가진 C++ 라이브러리입니다. 이걸 데스크톱 UI 애플리케이션에 직접 바인딩하면 **벤더 코드의 결함이 고객 앞에서 인터페이스를 함께 내립니다.**
- **촬영 실패를 사용자가 복구할 수 없습니다.** 카메라를 다시 연결하거나 앱을 재시작해줄 운영자가 없습니다. SDK가 실패했을 때 앱이 할 일은 **앱이 혼자 해야** 합니다.
- **중복 촬영은 실재하는 실패 모드입니다.** 고객이 버튼을 두 번 누릅니다. 실제로 성공한 요청 뒤에 재시도가 나갑니다. 둘 다 두 번째 세션을 만들고, 하류에서 **두 번째 유료 생성 잡**이 됩니다.
- **플랫폼을 옮겨야 했습니다.** 원래 macOS 애플리케이션이던 흐름을 키오스크 하드웨어에 맞춰 Windows 우선으로 전환했습니다.
- **벤더 SDK 바이너리를 레포에 넣을 수 없습니다.** 자체 라이선스가 있는 재배포물이고 용량도 큽니다.

**접근**

- **카메라는 별도 프로세스에 삽니다.** 전용 에이전트가 SDK를 소유합니다 — 초기화·연결·라이브 프로퍼티 조회·셔터·파일 전송 — 그리고 정의된 프로토콜로 애플리케이션과 통신합니다. **애플리케이션은 벤더 라이브러리를 링크하지 않습니다.**

  이게 핵심 결정입니다. 프로세스 경계와 프로토콜이라는 비용을 치르고 세 가지를 삽니다 — 벤더 SDK 크래시가 고객 대면 UI가 아니라 에이전트를 내리고, 세션 재시작 없이 에이전트만 재시작·재연결할 수 있으며, C++ 쪽과 C# 쪽을 독립적으로 빌드·테스트·추론할 수 있습니다. **무인 하드웨어에서는 가장 실패할 것 같은 구성요소를 격리하는 값어치가 in-process 호출의 편의보다 큽니다.**
- **앱 ↔ 에이전트 프로토콜은 공유된 명시적 계약입니다.** 먼저 작성된 쪽이 암묵적으로 정하는 게 아니라 별도 산출물로 관리합니다. 두 쪽의 툴체인이 달라 타입을 공유할 수 없기 때문입니다.
- **세션 상태는 로컬에 영속합니다.** 촬영 세션·재시도 상태·업로드 진행이 장비에 남습니다. 키오스크는 **불안정한 현장 네트워크** 너머 원격 백엔드의 클라이언트이므로, 끊긴 업로드는 **이미 일어난 촬영을 잃는 게 아니라 이어서 재개**해야 합니다.
- **중복 방지는 클라이언트에 있습니다.** 각 세션이 전송 전에 로컬에서 식별되므로, 이중 클릭이나 애매한 재시도가 새 세션이 아니라 **같은 세션**으로 매핑됩니다. 여기서는 이게 보통의 클라이언트보다 더 중요합니다 — 키오스크의 중복 세션은 [백엔드 큐](02-job-queue-for-paid-work.md)에서 **중복 유료 생성 잡**이 됩니다. **의도가 발생한 지점에서 중복을 제거하는 것이 사후에 대사하는 것보다 싸고 확실합니다.**
- **설치와 업데이트를 제품 표면으로 취급했습니다.** 키오스크 소프트웨어는 설치 프로그램으로 배포되고, **엔지니어가 아니라 고객사 운영자를 위해 쓰인** 업데이트 절차 문서가 함께 갑니다. 아무도 로그인하지 않을 장비에서 돌아야 하는 소프트웨어에는 **로그인을 필요로 하지 않는 업데이트 이야기**가 필요합니다.
- **벤더 SDK 바이너리는 레포 밖에, 기대 경로는 문서화.** 빌드는 무시 처리된 로컬 디렉터리에서 찾고, 셋업 문서에 어디에 두어야 하는지 명시합니다. 카메라 접근 자격증명은 코드나 셸 히스토리가 아니라 무시 처리된 환경 파일에 둡니다.

**검증** — 애플리케이션을 그 위에 올리기 **전에** 카메라 연동을 종단간으로 검증했습니다: 연결 → 바디에서 라이브 프로퍼티 읽기 → 원격 셔터 → 촬영 이미지를 호스트로 전송. 실제 하드웨어에서 전 경로를 먼저 증명했기 때문에, 애플리케이션은 **SDK 문서가 설명하는 동작이 아니라 실제로 확인된 동작**을 상대로 작성됐습니다. 네이티브 카메라 SDK에서 그 둘은 신뢰할 만큼 같지 않습니다.
