---
layout: post
title: "AWS with Nexus"
tags: [aws, maven, nexus]
---

# Install

java 8 이 설치 된 환경에서

```bash
$ wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
$ gunzip latest-unix.tar.gz && tar -xvf latest-unix.tar
$ cd nexus-3.19.0-01/
```

하면 끝이다.

넥서스 설치 후에

```bash
nexus/bin/nexus.rc
```

파일에서 넥서스가 사용 할 유저를 설정 해 주어야 한다...

[https://devopscube.com/how-to-install-latest-sonatype-nexus-3-on-linux/](https://devopscube.com/how-to-install-latest-sonatype-nexus-3-on-linux/)

    $ {nexus_home}/bin/nexus start // start or stop or restart

# Blob Stores 설정

설치 후 사이트로 들어가서 가장 먼저 해주어야 하는건, Blob Stores 를 지정 하는 일이다.

상단에

![/images/aws-blob-store.png](/images/aws-blob-store.png)

요기 녹색 설정으로 들어가서

Repository > Blob Stores 로 가서 Type 을 설정하고

안내에 따라 써주면 된다.

- 추가로 원래 s3 저장소가 없었는데 생겼다고 한다. 플러그인으로 제공했었는데, 굉장히 편해졌다고 함..

나의 경우, Type 을 s3 로 하고 Name 지정하고,

Region, Bucket , Prefix , Authentication 항목에 Access Key / Secret Key 항목을

차례되로 입력하니 끝났다.

S3 로 지정했으면,

아마,

![/images/nexus-002-sample.png](nexus-002-sample.png)

내가 지정 한 버킷 안에 이런 파일들이 생겼으면 성공 한 것이다.

Type 이 파일이면,,, 그냥 설정 하면 된다.

- 넥서스 NPM 배포

[https://sg-choi.tistory.com/123](https://sg-choi.tistory.com/123)

- Nexus 설치 후 {아이피}:8081/ 로 들어가서 adminuser 의 비밀번호를 바꿔준다.
- 배포를 허용하기 위해 Security > Realms > npm Bearer Token Realm 을 Active 그룹 으로 옮긴다.

# 배포

퍼블리셔 (배포자) 는 다음과 같이 설정 하여, npm 저장소를 추가 할 수 있다.

    $ npm config set registry {http://myhost.npm/npm-private}

그다음 로그인

    // 괄호 안은 위 과정을 생략 했으면, 써주어야 하고, 위 과정을 걸쳤다면, 생략해도 된다.
    $ npm login (--registry=http://myhost.npm/npm-private)

그럼 ~/.npmrc 파일이 생기고, 해당 파일을 프로젝트 루트로 옮겨 주도록 한다.

    $ mv ~/.npmrc .

이제 소스를 작성하고,

    $ npm publish

하면 끗.

# 사용

Nexus repository 를 사용하려면, nexus 의 Type 이 group 인 repository 가 있어야 한 듯 하다.

NPM 의 경우 repository > npm (group) 레포를 만들어주고,

위에서 배포하기 위해 만들었던 host repository 와 기존, npm 공식 공개저장소인 npm proxy 를 group

에 넣어주는 작업이 필요하다.

- NPM 공개 저장소 registry : [https://registry.npmjs.org/](https://registry.npmjs.org/)

그다음, 프로젝트를 생성하고 다시 npm login 을 해야하는데,

npm logout 을 먼저 실행하고,

이번에는 아까 npm-group 레포에 로그인을 해야한다.

    $ npm login (--registry=http://myhost.npm/npm-group)

이제 이전에 만든 프로젝트 에서 package.json 에 설정 된 "name" 을 찾아서 install 해주면 끝

    $ npm install nexus-test

결과적으로, 퍼블리셔와 사용자의 프로젝트루트에는 .npmrc 가 하나씩 들어있다.

# yarn.

npm 과 yarn 은 같은 레지스트리 를 사용하는 줄 알았는데, 아닌 것 같다.

이부분은 좀 더 봐야겠다..

그래도, jitpack 이랑 s3, github 등 여러가지 삽질한 후에 보니 좀 의미가 무엇인지는 아는 것같다...

삽질이 답인가

그리고 난 또 삽질을 하겠지.
