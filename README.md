- 개요 : 메신저봇R을 이용해서 원하는 날짜의 근무자를 알려주는 카카오톡 자동 응답 봇 제작
- 제작기간 : 02/01 ~ 02/02
- 사용언어 : JavaScript

# 시작
- 지인으로부터 카톡으로 채팅을 치면 근무자를 알려주는 기능을 만들어줄 수 있겠냐고 부탁을 받게 됨
- 그렇게 규모가 크지는 않을 거 같아서 알겠다고 함

# 요구사항 분석
- 하루의 근무는 총 3개의 시간대로 나뉜다. (야간, 석간, 조간)
- 근무자는 3명이다.
- 1주마다 근무시간이 로테이션형식으로 바뀐다.
- 일요일에는 모두 휴무이며, 토요일에는 석간이 휴무이다.
- 근무자와 근무패턴은 계속 고정이다.
- 즉, 아래와 같은 형태로 진행된다.

![image](https://user-images.githubusercontent.com/84609913/216977934-0c6b889a-9fcd-4e42-a04f-c058e1a1a0d4.png)

- 카카오톡 그룹 채팅에서 사용할 수 있으면 좋겠다고 함
- 입력과 출력
- "금일근무"라고 입력하면 오늘 날짜와 근무자들을 출력
- "[날짜정보8자리] 근무"라고 입력하면 해당 날짜의 근무자들 출력
  - EX) `20230204 근무`

# 설계
- 근무자와 근무시간대의 패턴이 가변적이지 않기 때문에 데이터를 따로 갱신하거나 저장할 필요는 없다.
- 기준일로부터 목표 날짜의 시간차이만 알면 어떤 로테이션의 근무인지 알아낼 수 있다.
- 매커니즘
  - 입력은 2가지 케이스로 받는다.
    - 정확히 "금일근무"라고 입력했을 때
    - `\d{8} 근무` 정규식을 통과하는 경우
  - 근무 패턴 구하기
    - 두 날짜의 시간(ms) 차이를 구한다
    - 시간차이를 바탕으로 몇일 차이가 나는지 구한다
    - 일 차이를 바탕으로 몇주 차이가 나는지 구한다
    - 주의 차이를 3으로 나눈 나머지를 통해 해당 날짜가 어떤 로테이션인지 구한다.
- 메신저봇R을 이용
  - 인터넷으로 찾아보니 메신저봇R이라는 앱을 사용하는 것이 가장 적절해보였다.
  - 카카오톡 알림을 캐치해서 스크립트의 로직을 실행하고 응답을 보내주는 앱이다.
  - 스크립트는 자바스크립트를 이용해야 한다.

# 완성
- 테스트 화면

![image](https://user-images.githubusercontent.com/84609913/216982755-3dc70d03-2e7c-4a28-9909-37688b19a60c.png)

# 끝나고 느낀 점
- 위에는 안썼지만 처음에 요구사항을 잘못 이해해서 괜히 쓸데없이 스프링 서버를 이용해서 엑셀데이터를 읽는 기능같은 걸 구현하느라 하루정도 삽질을 했다.
- 다음에는 처음부터 요구사항을 제대로 파악하는데 집중해야겠다.
- 개발(?)이라고하긴 좀 민망하지만 어쨋든 직접 구현한 것을 통해 누군가에게 실제로 도움을 준건 처음이어서 굉장히 뿌듯했다.

# 전체 코드

```javascript
const a = '김철수';
const b = "최영희";
const c = "박연진";

const days = ['일','월','화','수','목','금','토']

//기준일
const startDay = new Date(2023, 0, 30);

//날짜를 지정해서 물어볼 시 적용할 정규식
//ex) "20230204 군무"
const regex = /\d{8} 근무/;

//채팅방으로 답장
function response(room, msg, sender, isGroupChat, replier, imageDB, packageName) {

  //근무자를 알고싶은 날짜
  var targetDay;
  
  //메세지가 정확히 금일근무일 경우
  if(msg == '금일근무'){
    targetDay = new Date(); //날짜를 오늘로 설정
    replier.reply(room, getTargetDayWorkers(targetDay));
  }
  
  //메세지가 정규식을 통과한다면
  else if(regex.test(msg)){
    
    const yyyy = msg.substr(0, 4); //연도
    const mm = parseInt(msg.substr(4, 2)); //월
    const dd = msg.substr(6, 2); //일

    targetDay = new Date(yyyy,mm-1,dd); //날짜 설정
    if(mm-1 != targetDay.getMonth()){ //사용자가 입력한 월과 생성된 날짜의 월이 다르다면
      replier.reply(room, '날짜를 잘못 입력했습니다. 다시 확인해주세요.');
    }
    replier.reply(room, getTargetDayWorkers(targetDay));
  }
}

//목표날짜를 기반으로 해당 날짜와 근무자 출력
function getTargetDayWorkers(td){

  const diffTime = td.getTime() - startDay.getTime(); //목표 날짜와 기준일 사이 시간(ms)차이
  const diffDate = Math.floor(diffTime / (1000*60*60*24)); //목표 날짜와 기준일 사이 일 차이
  
  if(diffTime < 0) return "23년 1월 30일 이전은 안됨"; //기준일 이전은 안됨
  if(td.getDay() == 0) return enterWorker(td, '휴무', '휴무', '휴무'); //목표날짜가 일요일이면 모두 휴무
  if(diffTime == 0) return enterWorker(td, a, b, c); //기준일이면 패턴1
  
  const diffWeek = Math.floor(diffDate/7); //목표날짜와 기준일 사이 주 차이
  
  //근무패턴
  //3가지의 교대패턴이 주별로 로테이션 됨
  const routineType = diffWeek % 3;
  
  //패턴에 따라 다르게 출력
  switch(routineType){
    case(0):
    return enterWorker(td, a, b, c);
    case(1):
    return enterWorker(td, c, a, b);
    case(2):
    return enterWorker(td, b, c, a);
  }
}

//출력 결과값 생성
function enterWorker(td, night, dawn, morning){
  let result = "";
  
  //토요일이면 석간은 휴무
  if(td.getDay() == 6) dawn = '휴무';
  
  //해당 날짜
  result += td.getFullYear() + "/" + (td.getMonth()+1) + "/" + td.getDate() + "/" + days[td.getDay()] + "\n";
  
  result += "야간 : " + night + "\n";
  result += "석간 : " + dawn + "\n";
  result += "조간 : " + morning;
  return result;
}
```
