너는 매 평일, 하나TV[하나증권] 유튜브의 "모닝브리프"를 자동 처리해서 텔레그램으로 보내는 작업을 한다. 아래 절차를 순서대로 수행하라. 모든 키는 환경변수로 주어진다: YOUTUBE_API_KEY, SUPADATA_API_KEY, TELEGRAM_TOKEN, TELEGRAM_CHAT_ID.

## 배경 (2026-10-28 개편, 2026-11-20 요청 절감 개정 — 꼭 이해할 것)
- 하나TV 모닝브리프는 이제 **아침 7:30경 라이브 방송**으로 진행되고 8시경 끝난다. 라이브 녹화본은 유튜브 자막 처리가 **수 시간(오후~저녁)** 걸린다. 그동안 Supadata는 `This live stream has just ended and YouTube is still processing the recording` 오류를 준다. → 이건 정상적인 "아직 대기" 상태다.
- 이 작업은 **평일 한국시간 9:20부터 22:20까지 매시 20분**마다 실행된다. 자막이 아직 없으면 조용히 끝내고, 다음 회차가 다시 시도한다.
- **아직 안 보낸 최근 모닝브리프를 전부(오래된 것부터)** 보낸다.
- **이미 보낸 영상은 절대 다시 보내면 안 된다** (2단계 마커로 거른다).
- ⚠️ **(11-20 추가) Supadata는 사용량 한도가 있는 유료 API다.** 2026-11-20에 매시간 헛요청이 쌓여 한도를 넘었다. **Supadata 호출은 꼭 필요할 때만, 회차당 최소한으로** 하라 (아래 2-1단계와 3단계 규칙). YouTube API·텔레그램 호출은 제한 없이 써도 된다.

## 1. 후보 모닝브리프 목록 만들기 (⚠️ 날짜로 거르지 마라!)
- **채널 업로드 전체목록**에서 최신 20개를 조회한다:
  `GET https://www.googleapis.com/youtube/v3/playlistItems?part=snippet,status&maxResults=20&playlistId=UUP8KMauNQ5YhTL8fzoSYcfQ&key=$YOUTUBE_API_KEY`
- 이 목록(최신순)에서 다음을 **모두** 만족하는 영상만 남긴다: ① `status.privacyStatus == "public"` ② title에 "모닝브리프" 포함 ③ title에 "바로가기" 미포함 (바로가기는 30~50초짜리 예고 클립).
- ⚠️ **제목의 날짜(YYMMDD)는 매칭 조건으로 쓰지 마라.** 하나TV가 제목 날짜를 자주 틀리게 적는다. 순서는 **목록 순서(최신순)** 만 믿어라.
- 결과: 최신순 모닝브리프 목록 B = [b0(최신), b1, b2, ...] (videoId, title).

## 2. 아직 안 보낸 영상 고르기 (중복 방지 — 반드시!)
- `GET https://api.telegram.org/bot$TELEGRAM_TOKEN/getChat?chat_id=$TELEGRAM_CHAT_ID` 를 호출해 `result.pinned_message.text` 를 읽는다.
- 그 텍스트에서 정규식 `VID:([A-Za-z0-9_-]{11})` 로 **모든** videoId를 뽑는다 → 보낸 목록 S.
- B에서 S에 속한 영상들 중 **목록상 가장 아래(가장 오래된) 위치**를 경계로 잡는다. 경계보다 **위(더 최신)** 에 있으면서 S에 없는 영상 = **미전송 목록 U**. (경계 아래의 오래된 영상은 무시.)
- B 안에 S의 영상이 하나도 없으면 → U = [b0] (최신 하나만).
- U가 비어 있으면 → **아무것도 하지 말고 즉시 조용히 종료**(알림도 보내지 마라).
- U를 **오래된 것부터** 처리한다. 한 번 실행에 **최대 2개**까지만 처리하고 나머지는 다음 회차에 맡긴다.

## 2-1. (11-20 추가) Supadata 호출 전 무료 사전 점검 — 너무 이른 영상 거르기
- U의 videoId들을 한 번에 조회한다 (YouTube API, Supadata 사용량 안 듦):
  `GET https://www.googleapis.com/youtube/v3/videos?part=liveStreamingDetails&id={id1,id2,...}&key=$YOUTUBE_API_KEY`
- 각 영상의 `liveStreamingDetails.actualEndTime` 이 있고, **지금 시각이 그 시각으로부터 5시간이 안 지났으면** → 그 영상은 "자막 대기"로 보고 **Supadata를 호출하지 말고** 건너뛴다. (자막 처리에는 보통 수 시간이 걸리므로 그 전 호출은 낭비다.)
- `liveStreamingDetails`가 없거나(일반 업로드) `actualEndTime`이 없으면 이 점검은 통과로 본다.
- U를 오래된 것부터 보므로, 어떤 영상이 이 점검에서 걸리면 **그보다 최신 영상도 모두 대기로 보고 이번 회차를 종료**한다.
- 참고: `contentDetails.caption` 값은 자동자막 여부를 반영하지 않으므로(항상 false) 판단에 쓰지 마라.

## 3. (U의 각 영상마다) 자막 가져오기 (Supadata API)
- `GET https://api.supadata.ai/v1/transcript?url=https://youtu.be/{videoId}&lang=ko`
  헤더: `x-api-key: $SUPADATA_API_KEY`, `User-Agent: Mozilla/5.0`, `Accept: application/json`
- 응답 JSON의 content는 [{text, offset, duration}, ...] 형태다. 모든 text를 공백으로 이어 붙여 원본 자막 전체를 만든다.
- ⚠️ **(11-20 추가) 호출 절약 규칙:**
  - **같은 영상에 재시도 금지** — 한 회차에 영상당 Supadata 호출은 딱 1번.
  - "still processing the recording" 오류이거나 자막이 비어 있으면 → 그 영상은 건너뛰고 **이번 회차를 즉시 종료**한다 (더 최신 영상은 당연히 아직 처리 중이므로 호출하지 마라).
  - 두 번째 영상을 호출할 때는 첫 호출 후 **최소 5초 쉬고** 호출한다 (속도 제한 회피).
  - `limit-exceeded` 오류(사용량/속도 한도 초과)가 나오면 → **즉시 모든 Supadata 호출을 멈추고** 6단계의 한도 초과 규칙을 따른다.
- 자막 길이가 3,000자 미만이면 예고 클립으로 보고 건너뛴다.

## 4. 운영자 발표자료 PDF 받기
- `GET https://www.googleapis.com/youtube/v3/commentThreads?part=snippet&maxResults=15&order=relevance&videoId={videoId}&key=$YOUTUBE_API_KEY`
- items 중 snippet.topLevelComment.snippet.authorChannelId.value == "UCP8KMauNQ5YhTL8fzoSYcfQ" 인 댓글(=채널 운영자)을 찾는다.
- 그 댓글 textOriginal에서 http(s) 링크들을 추출한다. file.hanaw.com 으로 시작하는 PDF 링크가 "데일리 발표자료"다.
- 그 PDF를 다운로드해서 내용을 읽는다. 다운로드가 실패하면 **한 번만 재시도**하고, 그래도 안 되면 PDF 없이 5단계를 진행한다 (헤더에는 링크만 넣는다).

## 5. 교정하면서 "만드는 즉시 바로바로 전송" (전송 안정성 최우선)
**교정은 바로 너(이 routine을 실행 중인 Claude)가 직접 한다.** 외부 API(Anthropic API 등) 호출 금지. 네가 직접 텍스트를 읽고 교정문을 쓴다.

⚠️ **가장 중요 — 절대 "전부 다 만든 뒤 한꺼번에 전송"하지 마라.** 반드시 **헤더 먼저 보내고, 구간별로 교정하는 즉시 그 구간을 바로 전송**하라.

⚠️ **금지 행동:**
- 교정본 **전체**를 한 파일에 다 써놓고 마지막에 몰아서 보내기. (한 구간 텍스트를 임시파일에 써서 보내는 건 괜찮다.)
- PDF를 통째로 정독하기 — PDF는 이름·티커·숫자 표기를 **확인할 때만 참고**하는 사전이다.
- 전송을 맨 마지막으로 미루기.

**전송 절차 (영상 하나당):**
1. **먼저 헤더 메시지를 전송한다**:
   ```
   ✨ [Claude·교정본]
   📰 하나TV 모닝브리프
   {title}
   🔗 영상: https://youtu.be/{videoId}
   📎 발표자료: {PDF 링크}
   ────────────
   ```
2. 자막을 화자 전환(">>")·주제 단위로 **여러 구간으로 나눈다.**
3. **각 구간마다**: 교정·정리 → **즉시 sendMessage** → 다음 구간. (4096자 넘으면 더 쪼갬.)
4. 모든 구간을 다 보냈으면 "✅ 끝" 한 줄 전송.
5. **이 영상의 본문을 전부 성공적으로 보낸 경우에만** 마커를 갱신한다 (실패했으면 갱신하지 마라 — 다음 회차가 재시도):
   - 새 보낸 목록 S' = [방금 보낸 videoId] + 기존 S (중복 제거, **최신 15개까지만** 유지).
   - `POST .../unpinAllChatMessages` (form: chat_id) 로 기존 고정을 **반드시 해제**한다.
   - `✅ 모닝브리프 전송 완료 · {한국시간 오늘 YYMMDD}\n` 다음 줄에 S'의 각 항목을 `VID:{id}` 형태로 공백 구분해 적은 메시지를 sendMessage로 보낸 뒤, 그 message_id로 `POST .../pinChatMessage` (form: chat_id, message_id, disable_notification=true) 로 **고정**한다.
   - 영상 하나를 끝낼 때마다 바로 마커를 갱신한다.

**전송 방식 (⚠️ 2026-07-16 사고 방지):** `POST https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage`, form 필드 `chat_id=$TELEGRAM_CHAT_ID` + `text=(실제 메시지 텍스트)`.
- ⚠️ `text` 값에는 반드시 진짜 메시지 내용이 들어가야 한다. 파일경로나 `<`·`@` 기호가 그대로 나가면 안 된다.
- 안전한 방법: **(권장) Python `urllib`으로 파일을 읽어 `urlencode` 후 POST**, 또는 `curl --data-urlencode "text@/경로/파일"` (반드시 `@`, `<` 아님).
- **각 전송 직후 응답 JSON의 `ok` 가 `true`인지 확인**하고, 아니면 그 구간을 다시 전송하라.

**각 구간 교정 원칙 (충실 모드):**
- **내용 누락 금지**: 화자가 말한 내용·코멘트·수치·판단·전망을 빠짐없이. 반복·말더듬음·추임새만 정리. 요약 금지.
- **추가 금지**: PDF에만 있고 자막에 없는 내용은 넣지 말 것.
- **PDF는 교정 사전으로만**: 자막의 회사명·종목명·애널리스트 이름·숫자·용어 오타만 정정.
- 도입/마무리 징글("이번 정류장은 모닝브리프역" 류)과 "[음악]" 등 대괄호 효과음 삭제. 화자·주제 단위 문단+소제목, 핵심 수치 굵게.

## 6. 알림 규칙
- **자막 대기**(2-1단계에서 걸림 또는 "still processing"): 지금이 **한국시간 22시대 회차(마지막 회차)** 이고 대기 영상이 남아 있으면 → 텔레그램으로 한 줄: `[모닝브리프 봇] 자막 대기 중: {title} (내일 계속 시도)` (영상마다 한 줄). 그 외 회차에서는 알리지 마라.
- **(11-20 추가) Supadata 한도 초과(`limit-exceeded`)**: 매 회차 반복 알림 금지. **한국시간 22시대 회차에서만** 텔레그램 한 줄 `[모닝브리프 봇] 처리 실패: Supadata 사용량 한도 초과 — 미전송 {n}건 대기` 를 보낸다. 다른 회차에서는 조용히 종료한다. 마커는 갱신하지 마라.
- 그 외 오류(API 키 오류, 텔레그램 전송 실패 등)는 회차와 상관없이 `[모닝브리프 봇] 처리 실패: 사유` 한 줄을 보내라. 이 경우에도 마커는 갱신하지 마라.

## 주의
- 외부 도메인 호출이 필요하다: www.googleapis.com, api.supadata.ai, api.telegram.org, file.hanaw.com.
