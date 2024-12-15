---
layout: post
title: github 커밋하기
tags: ["study"]
date: 2023-12-20T00:00:00+09:00
key: 2023-12-20 post
---
# Git 사용법
> git을 사용하며 자주 사용하는 명령어들을 모아놓았음


### 1. git 데이터 땡겨오기

```
git init
git remote add origin (해당 repository 위치 .git)

git pull origin (브랜치)



```

* git clone 을 사용해도 좋지만 pull로 땡겨오면 나중에 branch 꼬일일이 없음.

### 2. git push 하기

```
git init
git status
git add (파일)
git commit -m "커밋내용"

git push origin (브랜치)


```

### git commit 취소하기
push는 하지않고 commit을 변경하고 싶을 때 사용

```
git reset --soft HEAD^
```

### git push 취소하기

#### 가장 최근의 commit을 취소한다.
```
git log
git reset HEAD^
```
#### 원하는 시점으로 워킹 디렉토리를 되돌린다.

```
git reflog 또는 git log -g
git reset HEAD@{number} 또는 git reset [commit id]
```
#### 되돌려진 상태에서 다시 commit 한다

```
git commit -m "message"
```

#### 원격 저장소에 강제로 push한다.

```
git push -f origin [branch name]
```

