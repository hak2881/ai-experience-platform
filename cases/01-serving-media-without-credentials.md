# Serving Generated Media Without Storage Credentials

**Surface** · A public page reached by scanning a QR code, on a stranger's phone
**Contents** · Someone's face, in a generated video
**Constraint** · The page must be able to show exactly one session's output and have no capability beyond that

## Context

A customer uses the kiosk, and a QR code gives them a URL containing a device identifier and a session identifier. They scan it and get their video and sticker.

That URL is printed, photographed, shared, and forwarded. It is the least controlled surface in the system, and what it leads to is personal: a video of a specific person's face.

## Problem

**The obvious implementation is a bucket link, and it is wrong in several ways at once.** Direct object URLs are permanent, guessable if keys are structured, and impossible to revoke. Making the bucket public to serve them makes every other object in it public too.

**Session identifiers in a URL are not secrets.** They appear in browser history, in screenshots, in messaging app previews, and in any analytics the page loads. Anything that treats possession of the URL as full authorization is trusting a value that travels in the clear by design.

**The web tier is the most exposed component.** A public Next.js app on a hosting platform is the thing most likely to be probed, and the thing whose dependencies you control least. It is the worst possible place to put credentials that can read a bucket.

**Provider output URLs are not a serving contract.** The generation provider returns transient URLs to its own storage. Handing those to a customer means the result page depends on a third party's retention policy, and the media disappears when they expire.

## Approach

**The web tier holds no storage credentials at all.** The result page runs server-side code that calls the backend with a scoped client credential — not a storage key. It cannot read the bucket, cannot list it, and cannot construct an object URL. If the entire web application were compromised, the attacker would gain the ability to call one backend API, not the ability to enumerate customer media.

```mermaid
flowchart LR
    QR[QR: device + session id] --> P[Result page]
    P -->|server-side<br/>scoped client credential| API[Backend session API]
    API --> DB[(Postgres:<br/>session → exact output keys)]
    API -->|signs ONLY those keys<br/>short expiry| U[Presigned URLs]
    U --> P
    P --> PH[Customer phone]

    X[Web tier] -. no credentials .-x S3[(S3)]

    style API fill:#e6f4ea,stroke:#1a7f37
```

**The backend signs specific keys, resolved from the database.** It looks up the session, reads the exact output keys recorded for it, and signs those. It never signs a prefix, never signs a pattern, and never signs anything derived from the request. The set of objects a session can reach is a database fact, not a string operation on user input — so there is no path by which a manipulated identifier reaches another session's media.

**Signed URLs are short-lived.** Long enough to load the page and play the video, short enough that a forwarded link stops working. Re-visiting the result page re-signs; sharing a URL from the address bar does not share the media indefinitely.

**Outputs are normalized into our own storage before serving.** The worker downloads the provider's transient output and re-uploads it under keys we control, at fixed known paths. Serving is therefore independent of the provider's retention, and swapping providers doesn't change the result page at all.

**Status files in storage exist only as compatibility mirrors.** Earlier in the system's life, session state was read from JSON objects in the bucket. That is still written for compatibility, but the database is authoritative and the API reads from it. Two sources of truth are tolerable only when one of them is explicitly labelled as not being one.

## What this defends against

| Attack | Why it fails |
|---|---|
| Guess or enumerate object keys | No public bucket access; keys are only reachable via signed URLs |
| Compromise the public web app | It holds no storage credential — only the ability to call one API |
| Manipulate the session identifier | Keys come from the database record for that session, not from the input |
| Reuse a forwarded link later | Signature expires |
| Provider deletes its outputs | Media was copied into our storage before it was ever served |

## What I would revisit

Possession of the URL is the authorization. That fits the product — a customer must be able to scan and view with no account and no friction — but it means a shared link is a shared video for as long as it stays fresh. If the content ever became more sensitive, the next step is a session token issued at the kiosk and bound to the first device that redeems it, accepting the support cost that comes with it.

---

## 한국어 요약

고객이 키오스크를 쓰면 QR 코드가 기기 식별자와 세션 식별자가 담긴 URL을 줍니다. 스캔하면 자기 영상과 스티커를 받습니다. **그 URL은 인쇄되고 촬영되고 공유되고 전달됩니다.** 시스템에서 가장 통제가 안 되는 표면이고, 그 끝에 있는 건 **특정 개인의 얼굴이 담긴 영상**입니다.

**어려웠던 지점**

- **가장 뻔한 구현은 버킷 링크이고, 동시에 여러 방향으로 틀렸습니다.** 객체 직접 URL은 영구적이고, 키가 규칙적이면 추측 가능하며, 폐기할 수 없습니다. 그걸 서빙하려고 버킷을 공개하면 **그 안의 다른 모든 객체도 공개**됩니다.
- **URL 안의 세션 식별자는 비밀이 아닙니다.** 브라우저 히스토리, 스크린샷, 메신저 미리보기, 페이지가 로드하는 모든 분석 도구에 남습니다. URL 보유를 곧 완전한 인가로 취급하는 설계는 **설계상 평문으로 돌아다니는 값**을 신뢰하는 것입니다.
- **웹 계층이 가장 노출된 구성요소입니다.** 호스팅 플랫폼 위의 공개 Next.js 앱은 가장 많이 탐침당하고 의존성 통제력이 가장 약한 것입니다. **버킷을 읽을 수 있는 자격증명을 두기에 최악의 장소**입니다.
- **Provider 출력 URL은 서빙 계약이 아닙니다.** 생성 provider는 자기 스토리지의 임시 URL을 반환합니다. 그걸 고객에게 주면 결과 페이지가 서드파티의 보존 정책에 의존하게 되고, 만료되면 미디어가 사라집니다.

**접근**

- **웹 계층은 스토리지 자격증명을 아예 갖지 않습니다.** 결과 페이지의 서버 사이드 코드가 **스토리지 키가 아니라 범위 제한된 클라이언트 자격증명**으로 백엔드를 호출합니다. 버킷을 읽을 수도, 나열할 수도, 객체 URL을 만들 수도 없습니다. **웹 애플리케이션 전체가 탈취돼도** 공격자가 얻는 건 백엔드 API 하나를 호출할 능력이지 고객 미디어를 열거할 능력이 아닙니다.
- **백엔드는 DB에서 해석한 특정 키만 서명합니다.** 세션을 조회해 **그 세션에 기록된 정확한 출력 키**를 읽고 그것만 서명합니다. prefix를 서명하지 않고, 패턴을 서명하지 않고, 요청에서 파생된 어떤 것도 서명하지 않습니다. 한 세션이 도달할 수 있는 객체 집합은 **사용자 입력에 대한 문자열 연산이 아니라 DB 사실**이므로, 조작된 식별자가 남의 세션 미디어에 도달할 경로가 없습니다.
- **서명 URL은 수명이 짧습니다.** 페이지를 열고 영상을 재생할 만큼 길고, 전달된 링크가 곧 안 먹힐 만큼 짧습니다. 결과 페이지를 다시 방문하면 다시 서명되고, 주소창의 URL을 공유한다고 미디어를 무기한 공유하게 되지는 않습니다.
- **출력물은 서빙 전에 우리 스토리지로 정규화합니다.** 워커가 provider의 임시 출력을 내려받아 **우리가 통제하는 고정된 키 경로**로 다시 올립니다. 그래서 서빙이 provider 보존 정책과 무관해지고, provider를 교체해도 결과 페이지는 전혀 바뀌지 않습니다.
- **스토리지의 상태 파일은 호환용 미러로만 존재합니다.** 초기에는 세션 상태를 버킷의 JSON 객체에서 읽었습니다. 호환을 위해 계속 쓰지만 **권위는 DB**이고 API는 DB를 읽습니다. 진실의 원천이 둘인 상태는, **그중 하나가 원천이 아니라고 명시적으로 표시될 때만** 견딜 만합니다.

**막아내는 것**

| 공격 | 실패하는 이유 |
|---|---|
| 객체 키 추측·열거 | 공개 버킷 접근 없음. 키는 서명 URL로만 도달 |
| 공개 웹 앱 탈취 | 스토리지 자격증명 없음 — API 하나 호출 능력뿐 |
| 세션 식별자 조작 | 키는 입력이 아니라 해당 세션의 DB 레코드에서 옴 |
| 전달받은 링크 재사용 | 서명 만료 |
| Provider가 출력 삭제 | 서빙 전에 이미 우리 스토리지로 복사됨 |

**다시 한다면** — URL 보유가 곧 인가입니다. 제품에는 맞습니다(계정도 마찰도 없이 스캔해서 봐야 하니까). 다만 공유된 링크는 신선한 동안 공유된 영상이라는 뜻입니다. 콘텐츠가 더 민감해진다면 다음 단계는 **키오스크에서 발급하고 최초 사용 기기에 바인딩되는 세션 토큰**이고, 그에 따르는 고객 문의 비용을 감수하는 것입니다.
