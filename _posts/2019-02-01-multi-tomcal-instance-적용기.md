---
layout: post
title: multi tomcal instance 적용기
tags:
  - NHN
  - 교육
  - 서버
---
이번 과제에 account서버와 mail서버를 분리하라는 요구사항이 있었습니다.
account 서버와 mail서버를 한 서버 안에서 분리하기 위해 저희 조는 tomcat instance를 2개 띄우는 방법을 택했습니다.

한 서버에서 tomcat을 띄우기 위해서는 총 5가지의 작업이 필요합니다.

1. apache/conf/httpd.conf 수정
1. apache/conf/workers.properties 수정
1. tomcat 2개 생성
1. tomcat/conf/server.xml 수정
1. tomcat/bin/catalina.sh 수정

각각 어떤 설정이 필요한지 어떻게 바꾸어야 하는지 알아보겠습니다.

---

1. apache/conf/httpd.conf 수정
apache httpd.conf는 서버에 접속한 url을 어느 tomcat으로 연결시켜 주는지를 설정하는 파일입니다.

```
<IfModule mod_jk.c>
    JkMount /* tomcat1 --- [1]
    JkMount /account* tomcat2 --- [2]
    JkMountCopy All
```
httpd.conf를 열고 아래로 쭉 내리다 보면 ifmodule mod_jk.c라는 설정을 찾을 수 있습니다.
이 곳에 JkMount {url} {worker}의 형식으로 작성해주시면 됩니다.
위 설정은 /account로 시작하는 모든 요청은 tomcat2가 그 이외의 모든 요청은 tomcat1이 실행됩니다.
만약 JkMount /account/ * 으로 설정하면 모든 /account로 오는 요청은 tomcat1이 실행되므로 조심해야 합니다.
---

2. apache/conf/workers.properties 수정

```
worker.list=tomcat1

worker.tomcat1.type=ajp13
worker.tomcat1.port=8001
worker.tomcat1.socket_timeout=10
worker.tomcat1.connection_pool_timeout=10

worker.list=tomcat2
worker.tomcat2.type=ajp13
worker.tomcat2.port=8002
worker.tomcat2.socket_timeout=10
worker.tomcat2.connection_pool_timeout=10
```

위에서 설정한 worker를 정의하는 파일입니다.
위에서 저는 tomcat1, tomcat2라고 사용하였으니 workers.properties에도 tomcat1, tomcat2를 만들어 주어야 합니다.
port 번호는 나중에 사용해야 하기 때문에 기억해 두셔야 합니다!

---

3. tomcat 2
