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

각각 어떤 설정이 필요한지, 어떻게 바꾸어야 하는지 알아보겠습니다.

---

### apache/conf/httpd.conf 수정
apache httpd.conf는 서버에 접속한 url을 어느 tomcat으로 연결시켜 주는지를 설정하는 파일입니다.

```
<IfModule mod_jk.c>
    JkMount /* tomcat1 --- [1]
    JkMount /something* tomcat2 --- [2]
    JkMountCopy All
```
httpd.conf를 열고 아래로 쭉 내리다 보면 ifmodule mod_jk.c라는 설정을 찾을 수 있습니다.
이 곳에 JkMount {url} {worker}의 형식으로 작성해주시면 됩니다.
위 설정은 /account로 시작하는 모든 요청은 tomcat2가 그 이외의 모든 요청은 tomcat1이 실행됩니다.
만약 JkMount /something/ * 으로 설정하면 모든 /something 오는 요청은 tomcat1이 실행되므로 조심해야 합니다.

---

### apache/conf/workers.properties 수정

```
worker.list=tomcat1

worker.tomcat1.type=ajp13
worker.tomcat1.port={ajp13 port #1}
worker.tomcat1.socket_timeout=10
worker.tomcat1.connection_pool_timeout=10

worker.list=tomcat2
worker.tomcat2.type={ajp13 port #2}
worker.tomcat2.port=8002
worker.tomcat2.socket_timeout=10
worker.tomcat2.connection_pool_timeout=10
```

위에서 설정한 worker를 정의하는 파일입니다.
위에서 저는 tomcat1, tomcat2라고 사용하였으니 workers.properties에도 tomcat1, tomcat2를 만들어 주어야 합니다.
port 번호는 나중에 사용해야 하기 때문에 기억해 두셔야 합니다!

---

### tomcat 2개 복사하기

tomcat instance를 2개를 사용하기 위해 실재 2개의 tomcat이 필요합니다.
설치된 폴더에서 tomcat을 복사하여 하나를 더 만들도록 합니다.

---

### tomcat/conf/server.xml 수정

tomcat server.xml은 각각의 tomcat 서버의 설정을 변경하는 파일로 제일 중요한건 port가 충돌나지 않고 아까 apache에서 맞춰주었던 AJP/1.3 프로토콜 port를 맞추어 주어야 한다는 것입니다.

```
<Service name="Catalina">
    <Connector port="9001" protocol="HTTP/1.1" --- [1]
               connectionTimeout="20000"
                            URIEncoding="UTF-8"
               redirectPort="8443" /> --- [2]

    <Connector port="{ajp13 port #1/#2}" protocol="AJP/1.3"  --- [3]
                enableLookups="false"
                                maxThreads="2048"
                acceptCount="100" debug="0"
                connectionTimeout="10000"
                useBodyEncodingForURI="true"
                maxPostSize="3145728"
                disableUploadTimeout="true"
                redirectPort="8443" />  --- [4]

    <Engine name="Catalina" defaultHost="localhost">
      <Host name="localhost"  appBase="webapps"
            deployOnStartup="false"
            unpackWARs="false" autoDeploy="false"
            xmlValidation="false" xmlNamespaceAware="false">

        <Context path="/" docBase="{war path}" reloadable="false"/>  --- [5]
        <Context path="/managerAgent" debug="0" privileged="true" docBase="managerAgent.war" />
      </Host>
    </Engine>
  </Service>
```
아까 복사한 tomcat 2개 모두 설정을 해줘야 합니다.

각각의 port는 겹치면 안되며 [3]에 AJP/1.3 프로토콜은 apache에서 설정했던 port번호를 입력해야 합니다.
5번에 docbase도 서버에 올릴 war 파일을 설정해줍니다.

---

tomcat1
```
export CATALINA_HOME=~/apps/tomcat
export CATALINA_BASE=~/apps/tomcat
```

tomcat2
```
export CATALINA_HOME=~/apps/tomcat2
export CATALINA_BASE=~/apps/tomcat2
```

마지막으로 catalina의 home과 base를 모두 각각의 tomcat이 깔려잇는 폴더로 설정해주시고 catalina.sh start을 모두 실행시키면 tomcat 2개가 실행된 것을 확인하실수 있습니다.
