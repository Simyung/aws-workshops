# Accessing AWS CodeCommit

AWS CodeCommit 저장소가 실습 환경에서 생성되었지만, IDE가 이에 연결하기 전에 몇 가지 단계를 완료해야 합니다.

나중에 발생할 수 있는 경고를 방지하기 위해 CodeCommit용 SSH 키를 known hosts 파일에 추가할 수 있습니다:

```
~$ ssh-keyscan -H git-codecommit.${AWS_REGION}.amazonaws.com &> ~/.ssh/known_hosts
```

그리고 Git이 커밋에 사용할 ID를 설정할 수 있습니다:

```
~$ git config --global user.email "you@eksworkshop.com"
~$ git config --global user.name "Your Name"
```

