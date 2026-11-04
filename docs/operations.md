# 실행과 배포

## 필요한 것

- **Node** — 문법 검사와 확인용 정적 서버에만 쓴다. 앱 자체는 Node가 필요 없다. 설치할 패키지는 없다.
- 최신 브라우저(확인용), 아이폰 사파리(실사용).

## 확인용 서버 띄우기

localStorage·IndexedDB가 필요해서 반드시 http로 띄운다(`file://`는 저장이 막힌다). `.claude/serve.js`는 git에서 빠져 있으므로 새로 받은 저장소에는 없다. 없으면 아래 내용으로 만든다.

```js
// .claude/serve.js — 저장소 루트를 내주는 정적 서버. 없는 경로는 404 (확인용 데이터 넣을 때 쓴다)
const http = require("http"), fs = require("fs"), path = require("path");
const ROOT = path.resolve(__dirname, ".."), PORT = Number(process.env.PORT || 4175);
const T = { ".html":"text/html; charset=utf-8", ".js":"text/javascript; charset=utf-8", ".json":"application/json; charset=utf-8",
  ".webmanifest":"application/manifest+json; charset=utf-8", ".png":"image/png", ".svg":"image/svg+xml" };
http.createServer((req, res) => {
  let rel = decodeURIComponent(req.url.split("?")[0]); if (rel === "/" || rel === "") rel = "/index.html";
  const file = path.join(ROOT, rel); if (!file.startsWith(ROOT)) return res.writeHead(403).end("forbidden");
  fs.readFile(file, (err, buf) => err ? res.writeHead(404).end("not found")
    : res.writeHead(200, { "Content-Type": T[path.extname(file).toLowerCase()] || "application/octet-stream", "Cache-Control": "no-store" }).end(buf));
}).listen(PORT, () => console.log("http://localhost:" + PORT));
```

```bash
node .claude/serve.js
```

`http://localhost:4175`에 뜬다. 포트를 바꾸려면 `PORT=4176 node .claude/serve.js`. 카카오 키로 확인할 때는 등록된 4175를 쓴다.

## 문법 검사

합치거나 올리기 전에 반드시 통과해야 한다. 저장소 루트에서:

```bash
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=[...h.matchAll(/<script>([\s\S]*?)<\/script>/g)];fs.writeFileSync(process.argv[1],m.map(x=>x[1]).join('\n;\n'));console.log('blocks:',m.length);" ./fd-syntax-check.js && node --check ./fd-syntax-check.js && echo "문법 정상" && rm -f ./fd-syntax-check.js
```

`blocks: 1`과 `문법 정상`이 나와야 한다. 뽑기가 먼저고 검사가 나중이며, 마지막 임시 파일 지우기까지가 한 명령이다.

## 확인용 데이터 넣기

1. `http://localhost:4175/__seed`로 간다(404 페이지라 앱 코드가 돌지 않는다).
2. 거기서 `localStorage.fd_data_v2`(또는 1판을 시험할 때 `fd_data_v1`)에 JSON을 넣고, 사진이 필요하면 IndexedDB `fd_photos`(store `photos`, keyPath `id`)에 `{id, full: Blob, thumb: Blob}`을 넣은 뒤 그 id를 방문의 `photos`에 적는다.
3. `/index.html`로 이동한다.
4. 데이터가 실제로 들어갔는지 먼저 읽어 본다.

로컬 서버에서만 `localStorage.clear()`를 쓴다. 실제 주소(`github.io`)는 가계부 앱과 저장 공간을 함께 쓴다.

## 카카오 키 넣기 (사용자가 한다)

1. 카카오 개발자 사이트(developers.kakao.com)에 로그인하고 애플리케이션을 만든다.
2. 앱 설정의 플랫폼 → Web에 사이트 도메인 `https://yooonji13-commits.github.io`와 `http://localhost:4175`를 등록한다.
3. 카카오맵 사용 설정을 켠다(콘솔에 있으면).
4. 앱 키 중 **JavaScript 키**를 `index.html`의 `var KAKAO_JS_KEY = '';` 따옴표 안에 넣고 커밋·배포한다. REST API 키·어드민 키는 넣지 않는다.

## 배포

- 저장소: `https://github.com/yooonji13-commits/food-diary` (공개).
- GitHub Pages가 `main` 브랜치 루트를 내준다. 주소: `https://yooonji13-commits.github.io/food-diary/`.
- 순서: 문법 검사 통과 → 375px 확인 → `main`에 커밋 → `git push origin main` → 1~2분 뒤 주소에서 새 화면 확인(`curl -s 주소 | grep` 등으로 바뀐 문자열이 보이는지).
- 폰에서는 사파리로 주소를 열고 공유 → 홈 화면에 추가. 홈 화면 앱은 완전히 닫았다 다시 열면 새 판을 읽는다.
