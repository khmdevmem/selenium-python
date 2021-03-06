# 1. 개발 계기
업무에 적용하기 위하여(이력서 제출 파트) 서버시간을 가져와 동시에 여러 개의 submit을 하는 프로그램을 제작하였다.

# 2. 개발 과정

## 2.1 패키지 선정

많이 알려진 웹 크롤링 패키지로 beutifulsoup 이나 selenium이 있다.

Javascript link를 사용하는 웹사이트의 경우 높은 복잡도를 가지고 있고, 이 때 beautifulsoup를 이용하기에 적합하지 않다. Selenium은 드라이버를 통해 웹 브라우저 위에서 동작하기 때문에 동적 상호작용을 할 수 있고 JavaScript link 또한 편리하게 접근할 수 있다. 속도는 selenium이 좀 느리다.

확장성을 생각해 selenium을 선택하였다.

## 2.2 개발 흐름

대략 생각해둔 개발 흐름은 다음과 같다.

1. 자동 로그인 (reqeust)
2. 폼 자동 작성 (selenium)
3. 서버 시간에 맞추어 제출 (time)

(+)
issue에 추가적으로 고려해야할 사항

- 캡차 들의 자동화 로그인 방지 코드.
- 한 번에 여러 개 프로세스 실행.
- 폼에 따라 제약이 있는 경우가 있다. (아이디 별 1폼, ip별 1폼)
- web element를 동적으로 생성할 수 있을까? (해당 폼의 web element를 감지해 자동으로 생성할 수 있나)

## 2.3 개발 진행

비교적 쉬운 작업부터 시작해 점차 심화하는 방식으로 개발하였다. 쉽다고 생각했던 부분도 잘 안되는 경우가 많아 시간을 많이 뺏겼다.

각 과정에서 발생한 문제와 원인, 해결방법에 대해서 기록하겠다.

### 2.3.1 로그인

먼저, 많이 알려져 있는 beautifulsoup 이나 selenium으로 xpath로 web element 가져와 입력하는 방법이 통하지 않았다.
delay를 주어도 캡차로 막히는걸 보면 아마 헤더에 python이 있어서 서버쪽에서 걷어내는 것 같다.

그래서 찾아낸 방법은 javascript로 실행하는 것이다.

```python
driver.execute_script("document.getElementsByName('id')[0].value=\'" + id + "\'")
driver.execute_script("document.getElementsByName('pw')[0].value=\'" + pw + "\'")
driver.find_element_by_xpath('//*[@id="frmNIDLogin"]/fieldset/input').click()
```

그리고 아래와 같이 로그인 후 바로 블로그에 이동하는 url을 이용하면 이동 후 어떤 동작도 먹히지 않는다.

`https://nid.naver.com/nidlogin.login?mode=form&url=https://blog.naver.com/example`

해결 방법은 간단했다.. 따로따로 페이지를 불러오면 된다.

`driver.get(https://nid.naver.com/nidlogin.login?mode=form)`

`driver.get(https://blog.naver.com/example)`

첫 번째 url로 이동한 블로그 페이지는 사용자 입장에서는 똑같이 동작하기 때문에 url이 문제일거라고는 생각도 못했었다. (xpath를 잘못 딴줄 알고 계속 고치고..)

### 2.3.2 지정된 시간에 submit 버튼 누르기

가장 핵심으로 생각한 부분은 로컬 컴퓨터와 서버 시간의 동기화였다.

검색해보니 여러 방법이 있었다.

1. 도메인 서버와 로컬 컴퓨터의 시간 동기화

도메인의 ip 주소를 알아내어 net 명령어를 이용해 windows time을 동기화 시켜주는 것이다.

시스템에서 설정 해줘야 하기 때문에 코드상 구현이 어렵다.

2. 일정 시간 마다 서버 시간 받아오기

```python
    serverDate = urllib.request.urlopen(url).headers['Date']
    datetime_server = dt.strptime(serverDate, '%a, %d %b %Y %H:%M:%S %Z')
```

생각할 수 있는 매우 간단한 방법, 통신 cost가 크다는 것이 단점이다.

3. 결론

1의 방법에는 오류가 없었으나 제대로 동작하는지 의심이 되고 확인 할 방법이 따로없었다.

2의 방법은 두 가지로 생각해봤는데
(1) 통신 코스트를 줄이기 위해 서버 시간은 한 번만 가져오고 로컬에서 count down을 함
(2) 일정 시간 마다 지속적으로 통신을 하여 서버 시간을 가져옴


(1)은 실행 시간이 오래 지속될수록 1초 이상의 오차가 있었다.(clock rate 에 따른 오차가 아닐까 싶음) 서버시간 사이트로 유명한 ㄴ이ㅂ즘도 장시간 틀어놓을 경우 오차가 있는 것으로 판명이 났는데 아마 이와 같은 방법을 쓰지 않았나 싶다. 최소 n분 전부터 실행 하도록 하고싶기 때문에 (2)의 방법을 사용하는 것이 오차를 가장 줄일 수 있었다. 

### 2.3.3 한 번에 여러 개의 프로세스 돌리기

멀티 프로세싱 vs 멀티 스레딩

이론상 멀티 프로세싱의 경우 각각의 프로세스가 request-response를 하여야하기 때문에 cost가 크다고 판단, 통신을 최소화 하기 위해 리소스를 공유할 수 있는 멀티 스레딩을 선택하였다.

두 가지 모두 해보았는데, 일단 목표가 최대 5개의 계정에서 동시 사용이기 때문에 두 개의 방법에 차이는 크지 않았다. 


### 2.3.4 아이디/ip 변경

만일 폼 제출에 아이디 당 하나, 혹은 ip당 하나 라는 제한이 걸려있어 다계정 제출을 위해서는 우회가 필요했다^^...

아이디 변경은 간단하게 프로세스 별로 다른 값을 주면 되지만 ip 변경이 꽤 까다로운 문제였다.

1. 익명 브라우저 사용

익명 브라우저를 사용하는 방법인데 많은 네트워크를 거치기 때문에 매우 느리다.

[참고1](https://wkdtjsgur100.github.io/selenium-change-ip/)

2. fake user agent를 이용한 ip 변경

    `pip install fake-useragent`

[참고1](https://brunch.co.kr/@ueber/28)
[참고2](https://pymon.tistory.com/11)


3. 한 번에 여러 개의 폼을 제출

이미 제출한 ip의 유저가 접속했을 경우 위에 레이어를 하나 더 씌워 값을 작성하지 못하는 식으로 차단한 것 같다. 그래서 같은 ip를 가진 여러 유저가 동시에 제출하면 ip 를 검사하지 않고 제출이 되는 것을 확인하였다. ip를 checking에 정확히 검증되지 않았지만 약 1~3초 정도의 간격으로는 제출이 가능하였다.


## 앞으로...

폼 마다 내용이 다 다르기 떄문에 동적으로 web element를 가져와 생성하면 좋을 것 같았다.
각 기능의 xpath나 네이밍 특징을 파악해서 생성이 가능한데 이 부분은 아직 미구현되었다.

## Issue
서버 시간을 초단위로 체크하는데 네트워크 오류가 발생해서 원하는 시간에 제출하지 못한 케이스가 있었다.
예를 들어 58초에 제출하길 원했는데, 이 때 오류가 발생해 response가 오지 않는다면 제출이 안된다.
이 부분은 초기에 생각했던 로컬에서 시간을 count down하면 해결 될 것 같은데 아직 구현 되지 않았고 local time과 server time을 비교해서 오차를 줄이는 로직이 따로 필요할 것 같다.
