---
title: 인트라넷 서버 고친 이야기
subject: Server
tags: [Server, Repair, 학교]
---

필자는 다니는 학교에서 교내 인트라넷 유지보수를 담당하고 있다. 당연히 새학년이 들어왔을테니 동아리 신청도 만들어 사용해야 한다. 

<!--more-->

기존에 있던 서비스들은 안드로이드마냥 심각한 파편화 문제를 가지고 있어, 이를 해결하기 위해 모든 서비스를 통합하는 큰 작업에 착수했고, 몇몇 사소한 문제점들을 제외하고는 나름 순조롭게 진행되고 있었다. **어제까지는.**

### 하아니 이게 왜터져 

사건은 유닉스 시간으로 `1520649000` 즈음에 발생했다. 마지막 업데이트를 위해 docker 이미지를 빌드한 후 실서버로 배포를 했는데, 거짓말같이 서버가 죽어버린것.

![sunfish]({{ site.resource }}/dimigoinpodcast-repair-server/sunfish.jpeg)<small>아직도 간담이 서늘</small>

나를 포함해 API 서버 팀 전체에 맨붕이 왔다. (그래봤자 2명이지만) 동아리 신청은 정오부터 시작인데, 서비스 개시 30분전에 서버가 원인도 모른채 펑 터져버렸으니 어떤 느낌이였는지는 굳이 표현하지 않아도 될 정도. 

멍때리기를 30초 정도 하다가, 정신을 차린뒤 서버가 터진 원인을 분석하기 시작했다. 남은 시간은 30분이 다기에, 최대한 머리를 써서 어떻게든 복구를 시켜야 했다.

![should]({{ site.resource }}/dimigoinpodcast-repair-server/should.jpg)<small><del>살려야 한다</del></small>

### 무엇이 문제?

가장 먼저 소스코드상에 문제를 예상했다. 레포지토리에 있는 코드를 다시 리뷰하고, 서버를 돌려봤지만 소용이 없었다. 그러던중 팀원이 로컬에서 테스트 했을때는 문제없이 잘 돌아간다는 것을 보고했다.

### 그렇다면?

도커 안에서 문제가 발상한다는 소린데... 일단 도커 이미지를 로컬에서 받고 실행해보았다. 그럼 이 문제를 해결할 실마리가 생기겠지?

```
Traceback (most recent call last):
  File "/root/.local/share/virtualenvs/app-J4eGhdK5/lib/python3.6/site-packages/gunicorn/arbiter.py", line 578, in spawn_worker
    worker.init_process()
  File "/root/.local/share/virtualenvs/app-J4eGhdK5/lib/python3.6/site-packages/gunicorn/workers/base.py", line 126, in init_process
    self.load_wsgi()
  File "/root/.local/share/virtualenvs/app-J4eGhdK5/lib/python3.6/site-packages/gunicorn/workers/base.py", line 135, in load_wsgi
    self.wsgi = self.app.wsgi()
  File "/root/.local/share/virtualenvs/app-J4eGhdK5/lib/python3.6/site-packages/gunicorn/app/base.py", line 67, in wsgi
    self.callable = self.load()
  File "/root/.local/share/virtualenvs/app-J4eGhdK5/lib/python3.6/site-packages/gunicorn/app/wsgiapp.py", line 65, in load
    return self.load_wsgiapp()
  File "/root/.local/share/virtualenvs/app-J4eGhdK5/lib/python3.6/site-packages/gunicorn/app/wsgiapp.py", line 52, in load_wsgiapp
    return util.import_app(self.app_uri)
  File "/root/.local/share/virtualenvs/app-J4eGhdK5/lib/python3.6/site-packages/gunicorn/util.py", line 352, in import_app
    import(module)
  File "/home/dimigoin/app/dimigoin/init.py", line 1, in <module>
    from flask import Flask
  File "/root/.local/share/virtualenvs/app-J4eGhdK5/lib/python3.6/site-packages/flask/init.py", line 17, in <module>
    from werkzeug.exceptions import abort
ModuleNotFoundError: No module named 'werkzeug'
```
<small>로-하! (로그 하이라는 뜻)</small>

그런데 예상과는 다르게 `flask`에서 `werkzeug`를 못 받아오는 현상을 발견했다. 더 머리가 멍해졌다. 

### 아이고 두야

차라리 코드작성에서 실수가 났다고 해주지, 이렇게 하나 뚝 띄워주고 안식을 취하는 `docker container`가 미워지는 순간이였다. 여러 방법을 취했다. 우선 코드에는 문제가 없음을 알았으니 라이브러리들을 체크했다. 먼저 버전부터.

`flask`의 버전부터 확인했다. 버전은 `0.12.2`로, 무려 **작년**에 릴리즈 된 버전이다. 그럼 `flask`에는 문제가 없다. 그럼 모듈을 불러오는 측인 `werkzeug`의 문제일까?

결론부터 말하자면 '아니'다. `werkzeug`역시 작년에 마지막으로 릴리즈된 버전이 업로드 되어있기 때문이다. 진짜 머리가 터지기 일보 직전이였다.

### 이걸 *(asterisk) 가?

절망에 빠져 아무생각 없이 멍하니 도커를 처다보다가, 팀원이 작성한 flask 코드에서 `jinja2` 모듈을 로드하지 못하는 모습을 발견했다. 우리 프로젝트는 `jinja2`를 쓰지도 않는데?

```python
from flask import *
```
<small>팀원이 쏘아올린 작은 코드</small>

위 코드는 flask 모듈 안에 있는 모든 것들을 import 하라는 구문이다. 여기서 해결의 결정적 실마리를 얻을 수 있었다.



### 알고보니 몰카에 몰카에 몰카였던거임

`jinja2`를 로드하지 못했다는 생각이 들면서, 코드 자체의 문제가 아니라는 생각이 들었다. 그리고 순간 뇌리에 스친 생각 하나, **`pipenv`**.

`pipenv`는 최근 나온 `virtualenv`와 `pip`을 섞어낸 패키지 관리 도구다. 생각보다 괜찮아서 이번에 도입한건데, 설마 설마 하며 업데이트 로그를 확인해보니 1시 30분 기준 3시간 전에 따끈따끈한 `pipenv` 가 나왔더랬다.

**설마 이거때문에?**라는 생각이 들어 며칠전 릴리즈된 버전으로 고정을 하고 docker에 올려보았더니 거짓말처럼 작동하기 시작했다.

### 여튼 내잘못 아님 ㅡㅡ

어찌됐건 발견한 버그를 해결하고 힘이 풀린 상태에서 멍하니 있던중 실무에 계시던 선배님 말씀 하나가 떠올랐다.

> 회사에서는 라이브러리 업데이트 잘 안해. 하려면 회의까지 하고 업데이트를 결정하더라고.

이 말을 처음 들었을때는 "정말 그렇게까지 해야하나?" 라는 생각을 했지만, 지금 와서 생각해보면 정말 큰 결정사항으로 취급해도 될 정도다.

소제목이 내잘못 아님 이라고 써놨지만, 버전명시 안한 내 잘못이다. 지금 팀 보드에는 라이브러리 버전명시 작업이 떡하니 올라와있다. 정말 버전 고정의 중요성을 뼈저리게 느낀 사고였다. 끗.