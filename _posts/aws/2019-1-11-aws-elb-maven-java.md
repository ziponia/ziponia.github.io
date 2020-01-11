---
layout: post
title: "Elasticbeanstalk /.ebextensions 폴더 를 maven으로 패키징 하기"
tags: [aws, maven, java]
---

# 라이브러리 번들 파일 준비

spring boot 를 jar 로 패키징 하게되면,

jar/.ebextensions

처럼 번들 파일에 root 경로에 .ebextension 폴더가 존재해야 한다.

일반적인 방법으로는 좀 까다로운 것 같고,

maven 에서 plugin 형태로 쉽게 되는 것 같다.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>1.6</version>
    <executions>
        <execution>
            <id>prepare</id>
            <phase>package</phase>
            <configuration>
                <tasks>
                    <unzip src="${project.build.directory}/${project.build.finalName}.jar" dest="${project.build.directory}/${project.build.finalName}" />
                    <copy todir="${project.build.directory}/${project.build.finalName}/" overwrite="false">
                        <fileset dir="./" includes=".ebextensions/**"/>
                    </copy>
                    <zip compress="false" destfile="${project.build.directory}/${project.build.finalName}.jar" basedir="${project.build.directory}/${project.build.finalName}"/>
                </tasks>
            </configuration>
            <goals>
                <goal>run</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```bash
$ mvn clean package
```

# 2. EFS mount 해보기

aws efs 콘솔로 들어가서 기본값 입력 후 파일시스템을 생성하게 되면,

파일 시스템 액세스 항목에 친절하게 커맨드가 나와있는 것을 볼 수 있다.

[Amazon EC2 탑재 지침(로컬 VPC에서)] 항목 클릭 후

파일 시스템 탑재 항목에 NFS 클라이언트 사용 커맨드를 붙혀넣고 실행하면 문제는 없는 듯 하다.

# 3. 자동화 설정을 위한 ELB 환경의 .ebextensions

.ebextensions

/efs-bootstrap.config

형태로 파일을 만들고,

efs-bootstrap.config

    packages:
      yum:
        nfs-utils: []

    commands:
      01_mount:
        command: "/tmp/mount-efs.sh"

    files:
      "/tmp/mount-efs.sh":
          mode: "000755"
          content : |
            #!/bin/bash

    			if [[ -z `grep "/efs" /proc/mounts` ]]
            then
              echo "EFS Mount Process."
              sudo mkdir /efs
              sudo chown webapp:webapp /efs
              sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport {efs_id}.efs.{region}.amazonaws.com:/ /efs
              if [ $? -ne 0 ] ;
                then
                  echo "Mount Error"
                else
                  echo "Success Mount EFS"
              fi
            else
              echo "aleady mounted EFS"
          fi

형태로 집어넣는다.

그리 어렵지 않는 커맨드이니 생략하고, 마지막 sudo mount -t ~~

부분이 위에 aws 가이드 에서 나와있는 커맨드이다.

이 다음에

패키징 하고 ELB 에 배포 하면 끝난다.

# 4. Error

3번 항목을 진행하기전에 반드시 실제 서버에서 커맨드가 제대로 동작하는지 확인하자.

mount 하기전에 EFS 의 보안그룹이 마운트 할 인스턴스를 허용해야 한다. 주의하자
