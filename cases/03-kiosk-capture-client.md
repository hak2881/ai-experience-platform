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
**하드웨어** — 제조사 원격 제어 SDK로 USB를 통해 구동하는 풀프레임 미러리스 카메라
**리스크** — 고객이 앞에 서 있고, **촬영 기회는 한 번**

키오스크가 고객 대면 흐름(프리뷰 → 촬영 → 업로드 → 결과)을 돌립니다. 웹캠이 아니라 실제 카메라를 제어하는데, 결과물 자체가 제품이기 때문입니다. 카메라는 제조사 원격 SDK로 제어하고 이건 네이티브 C++ 라이브러리입니다. 반면 데스크톱 애플리케이션은 C#입니다. 현장에는 기술 인력이 없고, 업데이트는 아무도 로그인하지 않을 장비까지 도달해야 합니다.

**어려웠던 지점**

- **SDK는 C++이고 앱은 아닙니다.** 네이티브 원격 제어 SDK는 자체 초기화 생애주기와 자체 콜백 스레딩 모델, 프로세스 상태에 대한 자기 나름의 고집을 가진 C++ 라이브러리입니다. 이걸 데스크톱 UI 애플리케이션에 그대로 바인딩하면 벤더 코드에서 난 결함이 고객 앞에서 화면까지 같이 내립니다.
- **촬영 실패를 사용자가 복구할 수 없습니다.** 카메라를 다시 연결하거나 앱을 재시작해줄 운영자가 없습니다. SDK가 실패했을 때 할 일은 앱이 혼자 해야 합니다.
- **중복 촬영은 실제로 일어나는 실패입니다.** 고객이 버튼을 두 번 누릅니다. 이미 성공한 요청 뒤로 재시도가 나갑니다. 둘 다 두 번째 세션을 만들고, 하류에서 **두 번째 유료 생성 잡**이 됩니다.
- **플랫폼을 옮겨야 했습니다.** 원래 macOS 애플리케이션이던 흐름을 키오스크 하드웨어에 맞춰 Windows 우선으로 바꿨습니다.
- **벤더 SDK 바이너리는 레포에 넣을 수 없습니다.** 자체 라이선스가 붙은 재배포물이고 용량도 큽니다.

**접근**

- **카메라는 별도 프로세스에 삽니다.** 전용 에이전트가 SDK를 소유합니다. 초기화, 연결, 라이브 프로퍼티 조회, 셔터, 파일 전송까지 맡고 정의된 프로토콜로 애플리케이션과 통신합니다. 애플리케이션은 벤더 라이브러리를 링크하지 않습니다.

  이게 핵심 결정입니다. 프로세스 경계와 프로토콜이라는 비용을 치르고 세 가지를 얻습니다. 벤더 SDK가 크래시해도 내려가는 건 고객 대면 UI가 아니라 에이전트이고, 세션을 재시작하지 않고 에이전트만 다시 띄워 연결할 수 있고, C++ 쪽과 C# 쪽을 따로 빌드하고 테스트하고 따져볼 수 있습니다. 무인 하드웨어라면 가장 잘 깨질 구성요소를 격리하는 편이 in-process 호출의 편의보다 남는 장사입니다.
- **앱과 에이전트 사이 프로토콜은 공유된 명시적 계약입니다.** 먼저 작성된 쪽이 암묵적으로 정해버리게 두지 않고 별도 산출물로 관리합니다. 양쪽 툴체인이 달라 타입을 공유할 수 없기 때문입니다.
- **세션 상태는 로컬에 남습니다.** 촬영 세션, 재시도 상태, 업로드 진행이 장비에 저장됩니다. 키오스크는 현장 네트워크 너머의 원격 백엔드를 쓰는 클라이언트이고 그 네트워크는 불안정합니다. 그래서 끊긴 업로드는 이미 찍은 촬영을 버리는 대신 이어서 재개해야 합니다.
- **중복 방지는 클라이언트 쪽에 둡니다.** 세션마다 전송 전에 로컬에서 식별자를 정하므로, 두 번 누르든 애매한 재시도가 나가든 새 세션이 아니라 같은 세션으로 모입니다. 여기서는 이게 보통의 클라이언트보다 더 중요합니다. 키오스크의 중복 세션은 [백엔드 큐](02-job-queue-for-paid-work.md)에서 중복 유료 생성 잡이 되기 때문입니다. 의도가 생긴 지점에서 중복을 걸러내는 편이 나중에 대사하는 것보다 싸고 확실합니다.
- **설치와 업데이트도 제품의 일부로 봤습니다.** 키오스크 소프트웨어는 설치 프로그램으로 배포하고, 업데이트 절차 문서를 엔지니어가 아니라 고객사 운영자를 독자로 두고 썼습니다. 아무도 로그인하지 않을 장비에서 돌아야 하는 소프트웨어라면 업데이트도 로그인 없이 되어야 합니다.
- **벤더 SDK 바이너리는 레포 밖에 두고 기대 경로만 문서화합니다.** 빌드는 무시 처리된 로컬 디렉터리에서 찾고, 셋업 문서에 어디에 두면 되는지 적어뒀습니다. 카메라 접근 자격증명도 코드나 셸 히스토리가 아니라 무시 처리된 환경 파일에 있습니다.

**검증** — 애플리케이션을 그 위에 올리기 전에 카메라 연동부터 종단간으로 확인했습니다. 연결하고, 바디에서 라이브 프로퍼티를 읽고, 원격으로 셔터를 누르고, 찍힌 이미지를 호스트로 전송하는 것까지. 실제 하드웨어에서 전 경로를 먼저 증명해둔 덕분에, 애플리케이션은 SDK 문서가 설명하는 동작이 아니라 확인된 동작을 상대로 작성됐습니다. 네이티브 카메라 SDK에서 그 둘은 믿을 만큼 같지 않습니다.
