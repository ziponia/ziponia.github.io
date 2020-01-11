---
layout: post
title: "(ubuntu ) Jenkins + Bitbucket + (AWS) Elb 연동하기"
tags: [aws, bitbucket, jenkins, elasticbeanstalk, elb]
---

## 1. 젠킨스 설치

```bash
$ wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
$ sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
$ sudo apt-get update
$ sudo apt-get install jenkins
```

## 2. java 설치 후 심볼릭 링크 추가

```bash
# java 설치 후

$ ln -s {java가 설치 된 경로} /usr/bin/java
```

## 3. jenkins start

```bash
$ service jenkins start
```

## 4. Jenkins 설정

아이피:8080 로 가서

웹사이트에 나와있는 경로에 password 를 가져와서 화면에 등록

- 왼쪽 버튼 클릭 후 플러그인 설치
- 플러그인 다 설치 되면 기본 admin 정보 입력

Manage Jenkins → Manage Plugins → Bitbucket 설치

젠킨스 재부팅

```bash
$ service jenkins restart
```

## Maven 설정

- Jekins 관리 → Global Tool Configuration → JDK → Add JDK → Install automatically 해제 → JAVA_HOME 항목입력 → scroll → Maven 항목 → installation maven → install automatically 체크 → save

재접속 후

New Item → Item name 입력 → Freestyle Project → [OK] → 프로젝트의 settings → Build 항목 → Add build step → [Invoke top-level Maven targets] → 위에서 설정 한 maven name 설정 → Gloals 설정 후 저장

![/images/jenkins-001.png](/images/jenkins-001.png)

## 5. bitbucket repo 설정

- [Source Code Management] 항목 Git 선택
- Repository URL 항목에 bitbucket repoURL 입력
- 옆에 add 클릭 [jenkins]
- bitbucket 이메일이랑 패스워드 클릭

![/images/jenkins-002.png](/images/jenkins-002.png)

- ADD 후 키설정
- 키선택

![/images/jenkins-003.png](/images/jenkins-003.png)

- [Build Triggers] 항목에 Build when a change is pushed to BitBucket 체크
- push 후 테스트

## 6. Elasticbeanstalk 설정

- AWS Elastic Beanstalk Deployment Plugin 설치 후 재부팅
- project → 구성 (settings) → [Build] 항목 → [Add build step] → AWS Elastic Beanstalk →

Crendentails → [Add] → jenkins → Kind → AWS Credentials → Access Key Id, Secrret Access Key 항목 입력

![/images/jenkins-004.png](/images/jenkins-004.png)

→ Add → Region 항목 입력 → Root Object 항목에 빌드 상대경로 입력 [./target/]

![/images/jenkins-005.png](/images/jenkins-005.png)

완료

이제 BuildNow 를 클릭하면 Build History 에 빌드 과정이 나온다.

![/images/jenkins-006.png](/images/jenkins-006.png)
