# Blog Project

쓰고 싶은 것을 기록하는 개인 블로그

## 명령어

### 개발 서버 실행 (Preview)
```bash
hugo server -D
```
- 접속 URL: http://localhost:1313
- `-D` 옵션: draft 포스트도 포함

### 빌드
```bash
hugo
```
- 결과물: `public/` 디렉토리에 생성

### 새 포스트 생성
```bash
hugo new posts/포스트명.md
```

## 디렉토리 구조

```
content/posts/   # 블로그 포스트
public/          # 빌드 결과물
docs/            # 프로젝트 문서
```

## 글쓰기 가이드

글 작성 시 [글쓰기 규칙](docs/writing-guide.md)을 참고한다.

## 작업 규칙

- 작업을 진행할 때마다 커밋한다.
- 하나의 작업 단위가 완료되면 즉시 커밋한다.
