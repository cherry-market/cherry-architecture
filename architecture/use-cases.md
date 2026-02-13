# Use Case Diagram (Backend Scope)

```mermaid
flowchart LR
  Guest((Guest))
  Member((Member))
  Seller((Seller))

  Guest --> Browse[상품 목록 조회/필터/정렬]
  Guest --> View[상품 상세 조회]
  Guest --> Trending[트렌딩 조회]
  Guest --> Signup[회원가입]
  Guest --> Login[로그인]

  Member --> Browse
  Member --> View
  Member --> Trending
  Member --> Me[내 정보 조회]
  Member --> Like[찜 추가/취소]
  Member --> LikeStatus[찜 여부 조회]
  Member --> LikesList[내 찜 목록 조회]
  Member --> ChatStart[채팅 시작]
  Member --> ChatList[채팅 목록 조회]
  Member --> ChatSend[메시지 전송]
  Member --> ChatHistory[메시지 히스토리 조회]
  Member --> ChatUnread[안읽음 수 조회]

  Seller --> ProductCreate[상품 등록]
  Seller --> ProductEdit[상품 수정]
  Seller --> ProductDelete[상품 삭제]
  Seller --> StatusChange[거래 상태 변경]
  Seller --> AIGenerate[AI 설명 생성]
  Seller --> MySales[내 판매 목록 조회]
```
