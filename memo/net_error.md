# net error

## Electron `net`

　Electronの[`net.request`][] APIによりGitHub APIの[createRepo][]を叩いたらエラーになった（`Error: net:ERR_CONNECTION_REFUSED`）

[net]:https://www.electronjs.org/ja/docs/latest/api/net
[`net.request`]:https://www.electronjs.org/ja/docs/latest/api/net#netrequestoptions
[createRepo]:https://docs.github.com/ja/rest/repos/repos#create-a-repository-for-the-authenticated-user

```javascript
const { app, BrowserWindow, ipcMain, dialog, net } = require('electron')
let request = net.request({...
```

　エラー内容は以下。

```
A JavaScript error occurred in the main process

Uncaught Exception:
Error: net:ERR_CONNECTION_REFUSED
  at SimpleURLLoaderWrapper.<anonymous> (node:electron/js2c/browser_init:101:7169)
  at SimpleURLLoaderWrapper.emit (node:events:527:28)
```

　画面キャプチャしたが一応テキストもここに残す。コピーできなかったので手書きした。

　原因不明。対処不明。

## Node.js https

　次はNode.jsの[`https.request`][]を使ってリクエストしてみた。

[動作変更: IPC で非 JS オブジェクトを送信すると、例外が送出されるように]:https://www.electronjs.org/ja/docs/latest/breaking-changes#%E5%8B%95%E4%BD%9C%E5%A4%89%E6%9B%B4-ipc-%E3%81%A7%E9%9D%9E-js-%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%82%92%E9%80%81%E4%BF%A1%E3%81%99%E3%82%8B%E3%81%A8%E4%BE%8B%E5%A4%96%E3%81%8C%E9%80%81%E5%87%BA%E3%81%95%E3%82%8C%E3%82%8B%E3%82%88%E3%81%86%E3%81%AB
[https]:https://nodejs.org/api/https.html
[`https.request`]:https://nodejs.org/api/https.html#httpsrequestoptions-callback

```javascript
const https = require('https');
ipcMain.handle('httpsRequest', async(event, url, options, onData=null, onError=null)=>{
    https.get(params.url, options, (res) => {
        console.log('statusCode:', res.statusCode);
        console.log('headers:', res.headers);
        res.on('data', (d) => {
            console.log(d)
            //process.stdout.write(d);
            if (onData) { onData(JSON.parse(d), res) }
        });
    }).on('error', (e) => {
        console.error(e);
        if (onError) { onError(e) }
    });
})
```

　すると以下エラーが出た。

```
Uncaught (in promise) Error: An object could not be cloned.
```

　ググってみると以下がヒットした。

* [動作変更: IPC で非 JS オブジェクトを送信すると、例外が送出されるように][]

> IPC を介して (ipcRenderer.send、ipcRenderer.sendSync、WebContents.send 及び関連メソッドから) オブジェクトを送信できます。このオブジェクトのシリアライズに使用されるアルゴリズムが、カスタムアルゴリズムから V8 組み込みの 構造化複製アルゴリズム に切り替わります。これは postMessage のメッセージのシリアライズに使用されるものと同じアルゴリズムです。 これにより、大きなメッセージに対するパフォーマンスが 2 倍向上しますが、動作に重大な変更が加えられます。

> 関数、Promise、WeakMap、WeakSet、これらの値を含むオブジェクトを IPC 経由で送信すると、関数らを暗黙的に undefined に変換していましたが、代わりに例外が送出されるようになります。

```javascript
// 以前:
ipcRenderer.send('channel', { value: 3, someFunction: () => {} })
// => メインプロセスに { value: 3 } が着く

// Electron 8 から:
ipcRenderer.send('channel', { value: 3, someFunction: () => {} })
// => Error("() => {} could not be cloned.") を投げる
```

* NaN、Infinity、-Infinity は、null に変換するのではなく、正しくシリアライズします。
* 循環参照を含むオブジェクトは、null に変換するのではなく、正しくシリアライズします。
* Set、Map、Error、RegExp の値は、{} に変換するのではなく、正しくシリアライズします。
* BigInt の値は、null に変換するのではなく、正しくシリアライズします。
* 疎配列は、null の密配列に変換するのではなく、そのままシリアライズします。
* Date オブジェクトは、ISO 文字列表現に変換するのではなく、Date オブジェクトとして転送します。
* 型付き配列 (Uint8Array、Uint16Array、Uint32Array など) は、Node.js の Buffer に変換するのではなく、そのまま転送します。
* Node.js の Buffer オブジェクトは、Uint8Array として転送します。 基底となる ArrayBuffer をラップすることで、Uint8Array を Node.js の Buffer に変換できます。
* Buffer.from(value.buffer, value.byteOffset, value.byteLength)

> ネイティブな JS 型ではないオブジェクト、すなわち DOM オブジェクト (Element、Location、DOMMatrix など)、Node.js オブジェクト (process.env、Stream のいくつかのメンバーなど)、Electron オブジェクト (WebContents、BrowserWindow、WebFrame など) のようなものは非推奨です。 Electron 8 では、これらのオブジェクトは DeprecationWarning メッセージで以前と同様にシリアライズされます。しかし、Electron 9 以降でこういった類のオブジェクトを送信すると "could not be cloned" エラーが送出されます。

　私はJSオブジェクトをセットしたつもり。非JSネイティブなオブジェクトではないはず。コードは以下。
　

renderer.js
```javascript
await hub.createRepo({
    'name':setting.github.repo,
    'description':"リポジトリの説明",
    //homepage:"",// URL
    //private:false,// プライベートリポジトリ
    //auto_init:false,
    //gitignore_template:"",
    //license_template:"mit",
})
```
github.js
```javascript
async createRepo(params) { // https://docs.github.com/ja/rest/repos/repos#create-a-repository-for-the-authenticated-user
    console.log(params)
    if (!params.hasOwnProperty('name')) {
        console.error(`引数paramsのプロパティにnameは必須です。`)
        return false
    }
    if (!this.#validRepoName(params.name)) {
        console.error(`引数params.nameの値が不正値です。英数字._-100字以内にしてください。`)
        return false
    }
    // Uncaught (in promise) Error: An object could not be cloned.
    await window.myApi.httpsRequest(
        'https://api.github.com/user/repos',
        {
            'method': 'POST',
            'headers': {
                'Authorization': `token ${this.token}`,
                'Content-Type': 'application/json',
            },
            'body': params,
        },
        (json, res)=>{
            console.debug(res)
            console.debug(json)
            console.debug('GitHub リモートリポジトリ作成完了！')
        },
        (e)=>{
            console.error(e)
            console.debug('GitHub リモートリポジトリ作成失敗……')
        }
    )
    console.log(`リモートリポジトリを作成しました。`)
}
```
main.js
```javascript
ipcMain.handle('httpsRequest', async(event, url, options, onData=null, onError=null)=>{
    const request = https.request(url, options, (res)=>{
        console.log('statusCode:', res.statusCode);
        console.log('headers:', res.headers);
        res.setEncoding('utf8');
        res.on('data', (d) => {
            console.log(d)
            if (onData) { onData(JSON.parse(d), res) }
        });
    }).on('error', (e) => {
        console.error(e);
        if (onError) { onError(e) }
    });
    if (options.hasOwnProperty('body')) {
        request.write(JSON.stringify(params.body));
    }
    request.end();
})
```

### 引数からコールバック関数をとってみる

　さらにオブジェクトもなくしてみる。

renderer.js
```javascript
await window.myApi.createRepo(
    setting.github.token, 
    setting.github.repo, 
    'リポジトリの説明',
    (json, res)=>{
        console.debug(res)
        console.debug(json)
        console.debug('GitHub リモートリポジトリ作成完了！')
    },
    (e)=>{
        console.error(e)
        console.debug('GitHub リモートリポジトリ作成失敗……')
    }
)
```

　`An object could not be cloned.`エラーは消えたが、以下のような403エラーが返ってきた。

```
statusCode: 403
headers: {
  'cache-control': 'no-cache',
  'content-type': 'text/html; charset=utf-8',
  'strict-transport-security': 'max-age=31536000',
  'x-content-type-options': 'nosniff',
  'x-frame-options': 'deny',
  'x-xss-protection': '0',
  'content-security-policy': "default-src 'none'; style-src 'unsafe-inline'",
  connection: 'close'
}

Request forbidden by administrative rules. Please make sure your request has a User-Agent header (https://docs.github.com/en/rest/overview/resources-in-the-rest-api#user-agent-required). Check https://developer.github.com for other possible causes.
```

　ググったらGitHubのページ[User agent の必要性][]に書いてあった。

[User agent の必要性]:https://docs.github.com/ja/rest/overview/resources-in-the-rest-api#user-agent-required

> すべての API リクエストには、有効な User-Agent ヘッダを含める必要があります。 User-Agent ヘッダのないリクエストは拒否されます。 User-Agent ヘッダの値には、GitHub のユーザ名またはアプリケーション名を使用してください。 そうすることで、問題がある場合にご連絡することができます。

# 成功コード



　たしかにWebAPIのリクエストには成功した。だが、致命的な問題がある。それはリクエスト後の取得JSON値を返せないことだ。コールバック関数を受け付ければ対処できるが、IPC通信APIはコールバック関数を引数として受け取るとエラーになりやがる。ようするにリクエスト後の処理ができない……。成否判定すらできない。

# 問題

* 下記２点によりリクエスト後の処理が実装できない……
    * IPCはコールバック関数を引数にとれない
    * [`https.request`][]は結果を`await`で`return`できない（コールバック関数呼び出しの形でしかリクエスト後の処理を実装できない）

　そんなバカな……。

　残念ながらコールバック関数をIPCの引数に渡すと以下エラーになる。

```
Uncaught (in promise) Error: An object could not be cloned.
```

　なら一体どうすればHTTPSリクエスト後の処理を実装できるの？　

　思いつかない。

　私としてはGitHub APIを叩いた後のJSON文字列を返したいのだが。まさか、できないの？　そんなわけないよね？　けどググっても見つけられなかったんだが。

* https://teratail.com/questions/334361

　もしかして標準ライブラリが非推奨ってこと？　Node.jsってヒドイ環境なのね。

