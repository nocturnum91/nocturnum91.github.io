---
layout: post
title:  "GitHub Blog에 댓글 기능 추가하기 (feat. utterances)"
excerpt: "utterances를 사용하여 블로그에 댓글 기능을 추가합니다."
date:   2023-02-06 00:00
category: dev
tags: [jekyll]
---

GitHub APP인 [utterances](https://github.com/apps/utterances)를 사용하여 댓글 기능을 추가하기로 했다.

1. [Create a new repository](https://github.com/new)에서 blog-comments repository를 public으로 생성한다.
repository 이름은 자유로운데 blog-comments를 많이 쓴다고 해서 그대로 썼다.  

2. 위 repository에 utterances.json 파일을 만든 후 블로그 도메인을 입력한다.
```json
{
  "origins": ["https://nocturnum91.github.io/"]
}
```  
<br>
3. [utterances](https://github.com/apps/utterances)를 설치하고 생성한 저장소에 연결한다.
![Alt text](/images/GitHub Blog에 댓글 기능 추가하기 (feat. utterances).png){: width="95%"}

4. [utterances configuration](https://utteranc.es/)  
- **Repository - repo**: githubId/생성한repo명을 입력한다 (ex. nocturnum91/blog-comments)  
- **Blog Post ↔️ Issue Mapping**:  
utterances은 github의 Issue를 댓글로 만들어주기 때문에 Post와 Issue 매핑이 필요하다.
  - **Issue title contains page pathname(*)**: 파일명으로 Issue 생성 (카테고리명/날짜/파일명)
  - Issue title contains page URL: URL 값으로 Issue 생성 (http://~~~/파일명.html)
  - Issue title contains page title: title 값으로 Issue 생성
  - Issue title contains page og:title: 메타 데이터의 og:title 값으로 Issue 생성
  - Specific issue number: Post와 Issue 번호가 매핑되도록 구성해야함(수동)
  - Issue title contains specific term: Issue 제목에 특정 용어가 포함되도록 구성해야함 
- **Issue Label**: Issue 라벨 값은 그냥 comments 로 설정했다. (안해도 됨)  
- **Theme**:
<br>
여기까지 설정한 후 **Enable Utterances**에 생긴 script를 복사한다.  
<br>
5. _layouts/post.html의 댓글 위치에 4에서 복사한 스크립트를 붙여넣는다.  
<br>

----
**번외편**

utterances 스크립트를 추가했더니 Refused to load the script 에러가 발생했다.
CSP(Content Security Policy)라고 XSS(Cross Site Scripting) 공격을 막기 위해 만들어진 정책인데
아주 기본적인 기능만 존재하는 테마가 Content-Security-Policy 설정은 해놔서 조금 당황했다.
(utterances 설정한 블로그들에는 대부분 설정이 안되어있어서...)

Content-Security-Policy 설정에 관한 자세한 포스팅은 언젠가 하도록 하고

내가 사용하는 테마 기준으로 _layout/default.html에
```html
<meta http-equiv="Content-Security-Policy" ...>
```

코드가 있었는데 아래처럼 수정했다.
전부 * 주기는 싫었다.

```html
<meta http-equiv="Content-Security-Policy"  
content="default-src 'self'; script-src 'self' utteranc.es 'unsafe-inline' 'unsafe-eval';  
style-src 'self' https://fonts.googleapis.com 'unsafe-inline'; img-src 'self' *;
font-src 'self' https://fonts.gstatic.com; connect-src 'self'; media-src 'self'; object-src 'self';
form-action 'none'; base-uri 'self'; frame-src utteranc.es;" />
```
- script-src에 utteranc.es 'unsafe-inline' 'unsafe-eval';  추가
- frame-src utteranc.es; 추가  
<br>
사실 화면단은 항상 되는대로 개발하기 때문에 잘못된게 있다면 알려주세요.


