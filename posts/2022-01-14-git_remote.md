Title: git remote auth 실패시
Date: 2022-01-14
Tags: Git

git remote 접근이 token 만료되고 업데이트하고 안되었다. 나는 분명히 auth 설정을 다 했는데도. 문제는 ssh로 기본 통신을 하도록 설정해놨는데 왜인지 remote 설정이 https로 되어있던 것. 그래서 간단하게 origin remote를 삭제하고 다시 만들었다.

```bash
❯ git remote -v
origin  https://github.com/jungwookim/jungwookim.github.io.git (fetch) # ssh로 설정해뒀기 때문에 https를 삭제하고 재설정 필요
origin  https://github.com/jungwookim/jungwookim.github.io.git (push)
❯ git remote remove origin


❯ git remote add origin git@github.com:jungwookim/jungwookim.github.io.git
❯ git fetch
From github.com:jungwookim/jungwookim.github.io
 * [new branch]      main       -> origin/main
```

git 초보일지
