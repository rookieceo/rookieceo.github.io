---
published: true
title: 'Google Apps Script를 활용하여 구글 폼 제출 시 외부 API 호출하기'
date: '2020-03-28 13:00:00 +0900'
categories: etc
last_modified_at: '2020-03-28 13:00:00 +0900
tags:
  - google-form
  - google-spreadsheet
  - google-apps-script
toc_sticky: true
toc: true
---

친구 부탁으로 구글 앱스 스크립트를 활용할 일이 생겼다.

구글폼을 사용하는 환경에서 스프레드 시트에 값이 추가되면 알림톡을 보내는 작업이다.

보통 온라인 설문지나 이벤트, 행사 참가자 신청 등 을 진행할 때 구글폼을 활용할 일이 많은데

이처럼 사용자 입력 시 트리거를 걸어 외부 API 호출하는 아이디어를 응용하면  구글 폼 사용시 불필요한 (혹은 불편한) 업무를 효율화하는 데 있어 좋은 활용이 될 것이다.

유익하여 기록으로 남겨둔다.


## 0. 사전 조건

- 구글 폼과 스프레트시트가 연결되어있어야 한다.
- 호출할 외부 API가 존재하여하 한다.



## 1. 구글 폼과 연결된 스프레드 시트에 트리거 설정

|                 도구 > 스크립트 편집기 선택                  |
| :----------------------------------------------------------: |
| ![image-20200328110008636](/assets/images/2020-03-28-using-google-apps-script/image-20200328110008636.png) |



## 2. 트리거 코드 작성

| 화면                                                         |
| :----------------------------------------------------------- |
| ![image-20200328110359881](/assets/images/2020-03-28-using-google-apps-script/image-20200328110359881.png) |

> Google App Script 트리거 코드를 작성하고 저장

   ```javascript
   function onFormSubmit(e) {
     var data = {
       // ... 데이터 영역에 값을 입력(key, value)
     };
     var headers = {
       // ... 헤더 영역에 추가적으로 값을 입력
     };
     // 그 외 REST API 관련 옵션
     var options = {
       'method' : 'post',
       'contentType': 'application/json',
       'headers' : headers,
       // Convert the JavaScript object to a JSON string.
       'payload' : JSON.stringify(data)
     };
     // 외부 API 발송 (UrlFetchApp)
     UrlFetchApp.fetch('외부 API호출 주소', options);    
   }
   ```



## 3. 트리거 저장

| 처음 저장할 경우에는 트리거 저장명을 입력한다.               |
| ------------------------------------------------------------ |
| ![image-20200328112133696](/assets/images/2020-03-28-using-google-apps-script/image-20200328112133696.png) |



## 4. 현재 프로젝트의 트리거 메뉴 선택

| 시계 모양을 클릭하면 트리거 목록으로 이동한다.               |
| ------------------------------------------------------------ |
| ![image-20200328112619761](/assets/images/2020-03-28-using-google-apps-script/image-20200328112619761.png) |



## 5. 트리거 추가/수정 조건 선택

| 아래 조건 대로 입력 후 저장한다.                             |
| ------------------------------------------------------------ |
| ![image-20200328112947624](/assets/images/2020-03-28-using-google-apps-script/image-20200328112947624.png) |



## 6. 테스트해보기

- 구글 폼을 입력하여 정상적으로 API가 전송되는 지 테스트 해본다.
- 실패하게 되면 해당 트리거 작성 계정으로 메일이 올 것이다.



끝.

