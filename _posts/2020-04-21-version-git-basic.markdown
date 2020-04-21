---
layout: post
title: "Git basic"
date: 2020-04-21 23:58:00
author: Juhyeok Bae
categories: Versioning
---
# 소개
Git은 소스코드의 버전을 관리해주는 툴이다. SVN, Perforce 등과 같은 다른 버저닝 툴과 마찬가지 소스코드를 커밋 하고 그 버전을 관리 한다는 점에서는 동일 하다. 하지만 분산형 구조를 갖는점, 버전을 snapshot id를 통해 관리 하는 점, branch와 fork를 통해 유연한 workflow를 가질 수 있는점 등을 통해 더 강력하고 유연하게 사용이 가능하다.

# File lifecycle in git
![tls-handshake](/assets/img/tls-handshake.png)
- untracked  
  원래 git 저장소에 없던 파일을 새로 추가 할시 파일은 untracked 상태가 된다. 이는 해당 파일이 git에서 관리 하는 파일이 아님을 뜻한다. 파일을 git을 통해 관리 하려고 하면 `git add`를 통해 tracked 상태로 변경 해야 한다.
- unmodified  
  git 에서 관리 되고 있으며 최신 상태라는 뜻이다. git에서 관리 되는 파일 중 수정하지 않은 모든 파일이 이 상태이다.
- modified  
  git에서 관리 되는 파일 중 수정이 되었으나 아직 commit 되지 않은 파일이다. 소스를 commit 하려면 modified 상태에 있는 파일을 먼저 stage로 올려야 한다. stage로 올리기 위해선 `git add` 명령어를 사용 한다.
- staged  
  수정된 파일 중 commit 하기 위하여 준비된 파일이다. `git commit` 명령어 입력시 staged 파일이 local git 저장소에 commit 된다. commit이 완료 되면 파일들은 unmodified 상태가 된다.

# 버전관리
- snapshot
  git에서 버전을 가르키는 commit id는 실제 데이터가 저장 되어 있지 않다. 이 commit id에는 파일들의 메타데이터와 디렉토리정보가 저장 되어 있다. 처음에는 통채로 저장 되고 GC가 일어 나면 delta만 저장 하는식으로 변경 된다. 보통 첫번째 commit이 전체 데이터를 가지고 뒷버전 commit이 증분을 가지는데 반해, git은 마지막 commit이 전체 데이터를 가지고 그 전버전 commit이 delta 데이터를 가진다. branch 또한 이 commit id만 저장하고 데이터는 저장하지 않는다.

- HEAD
  HEAD는 git에 존재하는 포인터이다. 현재 작업하는 로컬 브랜치를 가르킨다.

# Merge method
Git에서는 서로 다른 두 브랜치를 통합 하는 방법이 여러 가지 있다. fastforward, 3 way merge, rebase 등이 있으며 방법에 따라 새로운 commit을 만들기도 하고 포인터만 변경 하여 브랜치를 최신 버전으로 유지 한다.

- fastforward
  선형 관계의 new branch를 master branch에 머지 하는것으로 new branch가 가르키는 commit id를 master branch가 바라보도록 만듬. 다른 두 브랜치가 병합 되는것이 아니라 컨플릭이 생길일이 없다. 부모 브랜치에서 바로 나온 하나의 브랜치의 내용을 머지 하는것 임으로 컨플릭이 발생할 일 이 없다.

- 3-way merge
  공통 조상 v1과 다른 두 브랜치 커밋 2, 3을 사용해 새로운 커밋 생성. 3개의 커밋을 비교해 머지.
  merge 하게 되면 merge 커밋이 남게 되며 branch 생긴 병합 과정을 그대로 기록하게 됨.
  브랜치가 사라지더라도 히스토리 그래프 에서는 다른 가지로 표기되기 때문에 어떤 브랜치에서 커밋 진행 되어 어떻게 머지가 되었다는것을 알 수 있음.
  다만 브랜치 갯수가 많아 지거나 머지 횟수가 잦아 지면 그래프 가독성이 떨어져 지저분해 보임.
  원칙적으로 의미 있는 최소한의 변경 단위로 커밋을 해야 하지만 오타수정 등 자잘한 커밋을 많이 할 수 밖에 없게됨.
  한창 개발진행 중일때는 long-running branch인 master가 뒤로 밀리기도 하여 찾기 힘들어 지는 경우가 발생.

- rebase and merge
  rebase는 다른 브랜치의 commit 이지만 마치 long-running branch에서 나온 commit 처럼 보이도록 merge 하는 방법이다. 따라서 하나의 흐름이었던것 처럼 보이며 merge commit도 남지 않는다.
  커밋을 모두 살려 두기 때문에 누가, 언제 어떤 부분 수정했다는 정보는 전부는 알 수 있지만, 어떤 브랜치가 머지된지는 알지 못한다. master에서 쭉 시작한것 처럼 보이기 때문이다. 이를 보완하기 위한 방법으로는 tag를 달아 주는 방법이 있다.(시멘틱 버저닝)
  rebase 되는 브랜치의 기존 commit 들은 rebase 후 dangling 상태가 되어 사용할 수 없는 상태가 되며 GC 때 정리된다.
  장점으로는 커밋이 일렬로 정렬 되기 때문에 그래프가 이쁘게 그려지는 장점이 있다. 단점으로는 충돌시 모든 커밋에 대해 하나씩 충돌이 발생 한다. 따라서 많은 커밋이 담긴 브랜치를 머지할 경우 충돌이 발생하면 피곤해 지기 때문에 지양 하는것이 좋다.

- squash and merge
  squash는 여러 commit을 하나로 합치는 기능이다. 즉 다른 브랜치에서 생성된 여러 커밋들을 하나로 합쳐 타겟브랜치에 머지 하는것이다. commit을 하나로 합쳐 주는거 이외에 두 브랜치를 머지하는 방법은 똑같다.
  자잘자잘한 커밋들이 많이 안남아 일반 merge 보다 가독성이 좋은 경우가 많다. 버전을 잘 쪼갠다면 특정 버전에 들어가는 커밋을 다 합치고 변경 사항만 적어 관리할 수도 있다.
  하지만 merge 보다 정보력이 떨어진다는 단점 존재함.

# Branch
- long-running branch
  삭제 되는것이 아니라 계속 유지되는 생명주기를 갖는 브랜치이다. 대표적으로 master branch가 이에 해당한다. 다른 브랜치 들은 이 브랜치로 부터 checkout 하여 트리가 시작된다. 특정 기능 구현을 위해 만든 브랜치는 변경한 소스코드를 이 long-running branch에 병합하고 삭제 되기도 한다. 하지만 이 long-running branch는 삭제 되지 않고 다른 브랜치의 코드가 계속 해서 병합되며 유지 된다.

- topic branch
  특정 기능 구현을 위해 사용되는 branch 이다. master나 다른 branch에서 commit 되는 내용은 이 branch에 적용이 되지 않는다. 오직 특정 기능에 대한 변경 사항만 commit 가능 하기 때문에 독립적으로 운영이 가능하다. 개발이 끝나면 master나 develop 같은 long-running branch에 merge 하여 프로그램 전체에 기능을 추가한다. 사용이 끝나면 이 topic branch는 쓸모가 없어 지기 때문에 삭제 한다. 쓰지 않는 branch는 바로 삭제 하는것이 좋다. 그렇지 않으면 추적이 힘든 수백개의 브랜치가 쌓이게 된다.

- hotfix branch
  특정 버전으로 배포 되었으나 급하게 수정하여 다시 배포해야 할 일이 생길때 사용 하는 branch이다. 배포한 버전 혹은 안정적인 버전에서 checkout 하여 브랜치를 생성한다. 변경이 필요한 코드를 수정 하여 branch를 다듬는다. 그리고 해당 branch를 release 코드를 관리하는 branch에 머지 하여 배포를 진행한다. 그리고 해당 코드는 현재 production에 적용 되었기 때문에 개발도 이 코드의 내용이 반영 되어야 한다. 따라서 develop 같은 개발관련 long-running branch에도 merge 한다.

- remote branch
  remote 서버의 저장소 이다. 로컬에서 작업한 내용을 push나 기타 방법으로 서버에 올리지 않은 이상 add와 commit만으로는 local 저장소의 내용이 remote 저장소로 올라가지 않는다.

# git command
- fetch  
  원격 저장소의 내용을 로컬로 가져온다. 하지만 데이터를 merge하거나 rebase 하는등의 병합 작업은 하지 않는다.
  ```
  $ git fetch --all
  ```

- pull
  fetch를 통해 원격 저장소의 최신 코드를 가져오며 동시에 현재 브랜치에 merge 한다. fetch와 merge가 합쳐진 명령어라고 보면 된다.
  ```
  $ git pull origin master
  ```

- stash
  현재 작업 내용을 stack에 저장 하는 명령어이다. 특정 기능을 개발하다가 급하게 버그 수정등을 해야 할때 유용하게 사용 될 수 있다. stash 한 내용은 원래 작업하던 브랜치에서 다시 꺼내 쓸 수 있으며 다른 브랜치에도 해당 내용이 적용 가능하다.
  ```
  (git: new-feature)$ git status
  On branch new-feature
  Changes not staged for commit:
    (use "git add <file>..." to update what will be committed)
    (use "git checkout -- <file>..." to discard changes in working directory)

	  modified:   test.py
  => git status 명령어로 new-feature라는 branch에서 test.py 라는 파일을 수정중임을 알 수 있다.

  (git: new-feature)$ git stash
  Saved working directory and index state WIP on new-feature: 22a2df0 add dir
  => git stash로 현재 수정중인 파일을 stack에 저장 했다.

  (git: new-feature)$ git status
    On branch new-feature
    nothing to commit, working tree clean
  => stash 이후 수정 중인 파일이 stack으로 들어가 현재 branch는 깨끗하다.

  (git: new-feature)$ git stash list
  stash@{0}: WIP on new-feature: 22a2df0 add dir
  stash@{1}: WIP on file: 9404895 a
  stash@{2}: WIP on file: 9404895 a
  => stash list 명령어로 0번에 파일들이 저장되어 있음을 알 수 있다.

  (git: new-feature)$ git checkout -b bug-fix
  (git: new-feature)$ git commit -a -m "Fix endpoint typo"
  => 새로운 브랜치를 생성 하여 버그 수정후 커밋.

  (git: bug-fix)$ git checkout new-feature
  Switched to branch 'new-feature'
  => 다시 기능 구현을 위해 branch로 돌아옴.

  (git: new-feature)$ git status
  On branch new-feature
  nothing to commit, working tree clean
  => 현재 브랜치에 아무 변경 사항 없음을 알 수 있음.

  (git: new-feature)$ git stash apply
  On branch new-feature
  Changes not staged for commit:
    (use "git add <file>..." to update what will be committed)
    (use "git checkout -- <file>..." to discard changes in working directory)

  	modified:   test.py

  no changes added to commit (use "git add" and/or "git commit -a")
  => stash apply 를 통해 가장 최근의 stash를 현 브랜치에 적용.
     특정 index의 stack을 적용 하고 싶다면 git stash apply 2 처럼 apply 뒤에 index 번호를 적어 준다.

  (git: new-feature)$ git stash drop
  Dropped refs/stash@{0} (468c3973b75cd4714bc7f38f3ec1e247b4e9422e)
  => 사용이 끝난 stash stack은 drop 명령어로 삭제 가능하다.
    만약 stack에서 꺼내서 적용과 동시에 삭제 하고 싶다면 git stash pop 을 사용.
  ```

- git clean
  현재 브랜치에서 untracked 파일을 날려 버릴때 사용 된다. tacked인 modified나 staged 파일은 삭제 되지 않는다.
  ```
  (git: new-feature)$ $ git status
  On branch new-feature
  Changes to be committed:
    (use "git reset HEAD <file>..." to unstage)

  	new file:   new

  Changes not staged for commit:
    (use "git add <file>..." to update what will be committed)
    (use "git checkout -- <file>..." to discard changes in working directory)

  	modified:   readme.md

  Untracked files:
    (use "git add <file>..." to include in what will be committed)

  	the_new.txt
  => clean 시 untacked인 해당 파일만 지워질걸로 예상 된다.

  (git: new-feature)$ git clean -f -d
  Removing the_new.txt
  => clean 명령어에 강제 삭제 -f와 하위 dir 정리 -d를 추가해 실행.

  (git: new-feature)$ git status
  On branch new-feature
  Changes to be committed:
    (use "git reset HEAD <file>..." to unstage)

  	new file:   new

  Changes not staged for commit:
    (use "git add <file>..." to update what will be committed)
    (use "git checkout -- <file>..." to discard changes in working directory)

  	modified:   readme.md
  => untracked만 삭제 됨.
  ```
