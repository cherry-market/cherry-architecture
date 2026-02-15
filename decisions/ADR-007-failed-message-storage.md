# ADR-007: 실패 메시지 영구 저장소 선택

- **상태**: 확정
- **일자**: 2026-02-15
- **참여**: Claude (기술 분석, 선택지 제시) + 사용자 (최종 결정)

---

## 맥락 (Context)

Phase 6.3에서 네트워크 끊김 시 메시지 큐잉 기능을 구현했다. 전송 실패한 메시지를 사용자가 재전송/취소할 수 있도록 `failedMessages` 큐를 Zustand store에서 관리한다.

**문제:** Zustand store는 인메모리이므로, 채팅방을 나갔다 돌아오거나 브라우저를 새로고침하면 실패 메시지가 유실된다. 사용자가 재전송 기회를 잃게 된다.

**추가 고려사항:**
- Cherry 앱의 최종 배포 형태가 미정 (웹 전용 / React Native / Electron 가능성)
- 저장소 선택이 향후 플랫폼 확장에 영향

---

## 선택지 (Options)

| 선택지 | 장점 | 단점 |
|--------|------|------|
| **1. localStorage** | 가장 간단한 구현, 동기 API | 5MB 제한, 네이티브 앱 미지원, 동기 API로 UI 블로킹 가능 |
| **2. IndexedDB (localForage)** | 웹/PWA/Electron 모두 지원, 비동기 API, 용량 제한 거의 없음 | 라이브러리 의존성 추가 (8KB) |
| **3. IndexedDB (idb)** | 경량(3KB), Promise 래퍼 | IndexedDB 전용 (폴백 없음) |
| **4. Native IndexedDB** | 의존성 0 | 콜백 기반 API로 코드 복잡, 보일러플레이트 과다 |
| **5. 추상화 레이어** | Storage 인터페이스 정의, 플랫폼별 교체 가능 | 현 시점에서 over-engineering, 배포 형태 미정 |

---

## 결정 (Decision)

**선택지 2: IndexedDB (localForage)** 채택

- `localforage` 라이브러리 사용 (IndexedDB → WebSQL → localStorage 자동 폴백)
- 저장소 접근을 `chatStorage.ts` 모듈로 분리하여 향후 교체 용이
- 추상화 인터페이스까지는 두지 않되, 단일 파일 교체로 플랫폼 대응 가능

### 구현 구조

```
features/chat/services/chatStorage.ts   <- 저장소 접근 모듈 (교체 지점)

features/chat/store/chatStore.ts        <- chatStorage 사용
  ├── addFailedMessage()     → chatStorage.saveFailedMessages()
  ├── removeFailedMessage()  → chatStorage.saveFailedMessages()
  ├── cancelFailedMessage()  → chatStorage.saveFailedMessages()
  └── restoreFailedMessages() <- chatStorage.loadFailedMessages()
```

### 저장 타이밍

| 이벤트 | 동작 |
|--------|------|
| 메시지 전송 실패 | `failedMessages` 큐에 추가 + IndexedDB 저장 |
| 서버 확인 (sent) | 큐에서 제거 + IndexedDB 갱신 |
| 사용자 취소 | 큐 + UI에서 제거 + IndexedDB 갱신 |
| 채팅방 진입 | IndexedDB에서 해당 roomId 메시지 복원 |
| 채팅방 퇴장 | 인메모리만 정리 (IndexedDB는 유지) |

---

## 선택 근거

1. **플랫폼 호환성**: localStorage는 네이티브 앱에서 사용 불가. IndexedDB는 웹/PWA/Electron에서 모두 동작
2. **비동기 API**: UI 블로킹 없음, 대량 데이터에도 안전
3. **localForage 자동 폴백**: IndexedDB 미지원 환경에서 WebSQL → localStorage로 자동 전환
4. **최소 복잡도**: 추상화 인터페이스 없이 `chatStorage.ts` 단일 파일 교체로 향후 대응 가능
5. **경량**: localForage 자체 8KB gzipped, 실패 메시지 데이터는 소량

---

## 향후 확장

| 시나리오 | 대응 방법 |
|----------|----------|
| React Native 앱 추가 | `chatStorage.ts`를 MMKV/AsyncStorage 기반으로 교체 |
| Electron 앱 | 현재 구현 그대로 사용 가능 (IndexedDB 지원) |
| 실패 메시지 외 오프라인 데이터 | `chatStorage.ts`에 메서드 추가 |
| 저장소 추상화 필요 시 | `StorageAdapter` 인터페이스 도입 + DI |

---

## 관련 문서

- [ADR-005: 채팅 메시지 저장 전략](./ADR-005-chat-message-storage.md)
- [채팅 도메인 아키텍처](../architecture/chat-domain.md) — 4.5절 메시지 전송 실패 및 재전송
