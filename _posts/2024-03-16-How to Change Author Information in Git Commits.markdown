---
layout: post
title:  "Git Commit Author(name, email) 수정하는 방법"
excerpt: "Git Commit name, email 수정하는 방법"
date:   2024-03-16 00:00
category: dev
tags: [Git]
---

자꾸 집에서 회사 일을 하다보니 회사 프로젝트에는 개인 author로 commit하고, 개인 프로젝트에는 회사 author로 commit하는 일이 반복됐다.
매번 검색해서 수정하기 귀찮아 방법을 블로그에 정리해두고자 한다.

# 모든 commit에 대해 author 수정하는 방법

1. 사용자 이름, 이메일 설정하기
```bash
git config --global user.name "nocturnum91"
git config --global user.email "nocturnum91@gmail.com"
```
<br>
2. git filter-branch 명령어로 author 수정하기
```bash
git filter-branch --env-filter '
OLD_EMAIL="이전 이메일 주소"
CORRECT_NAME="nocturnum91"
CORRECT_EMAIL="nocturnum91@gmail.com"
if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
```
<br>
3. 변경사항을 원격 저장소에 강제로 푸시하기
```bash
git push --force --tags origin 'refs/heads/*'
```
<br>

# 특정 commit에 대해 author 수정하는 방법

1. 사용자 이름, 이메일 설정하기
```bash
git config --global user.name "nocturnum"
git config --global user.email "nocturnum91@gmail.com"
```
<br>
2. git rebase -i 명령어로 수정할 commit 선택하기
```bash
git rebase -i HEAD~3
```
위 명령어를 입력하면 가장 최근 3개의 커밋을 수정할 수 있는 창이 나타납니다.
특정 커밋을 수정하고 싶다면
```bash
git rebase -i commit_id
```
<br>
3. 수정할 커밋을 pick에서 edit으로 변경하고 저장하기
```bash 
e 1a2b3c4 commit message
...
:wq!
```
<br>
4. git commit --amend 명령어로 author 수정하기
```bash
git commit --amend --author="nocturnum <nocturnum91@gmail.com>"
git rebase --continue 
```
수정할 개수만큼 반복한다.
<br>
5. 변경사항을 원격 저장소에 강제로 푸시하기
```bash
git push --force origin branch_name
```
<br><br>
force push를 할 때에는 반드시 주의해야 한다.
다른 사람이 작업하고 있는 브랜치에 force push를 하면 다른 사람의 작업이 사라질 수 있기 때문...