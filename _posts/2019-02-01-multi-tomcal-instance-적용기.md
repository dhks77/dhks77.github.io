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


---


---
