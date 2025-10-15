---
title: "Next.jsのDockerビルドを25分→13分に高速化した話【GitHub Actions】"
emoji: "⚡"
type: "tech"
topics: ["nextjs","docker","githubactions","ci"]
published: true
---

## TL;DR（この記事で分かること）

- GitHub ActionsでNext.jsをDockerビルドすると毎回フルリビルドされて遅い問題の解決策
- BuildKitのキャッシュマウントがGitHub Actionsで効かない理由
- `.next/cache`を明示的に出し入れすることで**ビルド時間を25分→13分に短縮**（48%削減）
- コピペで使えるワークフロー例あり

## はじめに：25分のビルド地獄からの脱出

ある日、PRを出すたびにGitHub Actionsのビルドが**25分**もかかっていることに気づきました。

「またビルド待ち」が口癖になっていた頃です😇

ログを見ると、pushするたびにWebpackがフル再コンパイル、ページも全部再生成。Next.jsのビルドキャッシュが全く効いていませんでした。

不思議なのは、`npm run build`を直接GitHub Actions上で実行すると爆速なのに、**Docker経由だと毎回ゼロからビルド**されること。

「Dockerのレイヤーキャッシュ使ってるのに、なんでだろう」

この疑問から始まった高速化の旅を、この記事でシェアします。

## 問題の本質：Next.jsのキャッシュがDockerで消える

### Next.jsのビルドキャッシュって何？

Next.jsは`.next/cache`というディレクトリにビルド成果物を保存して、次回以降のビルドを爆速化します：

- **Webpackコンパイルキャッシュ** - コンパイル済みモジュールとチャンク
- **TypeScriptキャッシュ** - 型チェック結果（`.tsbuildinfo`）
- **SWCキャッシュ** - 変換済みのJavaScript/TypeScriptファイル
- **画像最適化キャッシュ** - 処理済み画像

ローカルや普通のCI環境で`npm run build`を再実行すると、Next.jsはこのキャッシュをチェックして、変更がないファイルは再コンパイルをスキップしてくれます。**これが数分かかるビルドを数秒に短縮する秘密**です。

### なぜDockerだとキャッシュが消えるのか？

ネイティブ環境では`.next/cache`がビルド間で保持されるため、これが自動的に機能します。

**ところがDockerだと？**

```
ビルド開始 → キャッシュ生成 → ビルド中に使用 → コンテナ終了 → キャッシュ消失
```

毎回ゼロからビルドが始まります。せっかく生成されたキャッシュが、コンテナと一緒に捨てられてしまうんです。

## 試行錯誤：BuildKitのキャッシュマウントが効かない罠

「よし、BuildKitのキャッシュマウント使えばいいじゃん！」

そう思って、最初はこう書きました：

```dockerfile
RUN --mount=type=cache,target=/app/.next/cache \
    npm run build
```

GitHub Actionsのキャッシュも設定：

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

これで完璧のはずが、ビルドは相変わらず遅いまま。**毎回25分のフルビルド**。

ログを見ても、キャッシュヒットの形跡なし。なぜだろう？

### 原因：GitHub Actionsとキャッシュマウントの相性問題

調べてみると、**GitHub Actionsのキャッシュバックエンド（type=gha）は、BuildKitのキャッシュマウントを正しくエクスポートしてくれない**ことが判明。

つまり：
- キャッシュマウントは作られる ✅
- ビルド中は使われる ✅
- でも次回のビルドのために保存されない ❌

完全に罠でした。

## 解決策：キャッシュを明示的に出し入れする

ということで、発想を変えました。

**BuildKitに任せるのではなく、Next.jsのキャッシュをDockerコンテナの内外で明示的に出し入れする**作戦です。

具体的には、以下の3ステップ：

### 1. Dockerビルド前にキャッシュをリストア

```yaml
- name: Cache Next.js build
  uses: actions/cache@v4
  with:
    path: .next/cache
    key: ${{ runner.os }}-nextjs-docker-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-nextjs-docker-
```

これでランナー上に`.next/cache`がリストアされます。Dockerイメージをビルドする前に実行するのがポイント。

### 2. キャッシュをDockerに含める

`.dockerignore`を作成して、重いファイルは除外するけど、`.next/`は**除外しない**ようにします：

```
node_modules
.git
.github
```

こうすれば、`COPY . .`でリストアされたキャッシュがDockerに含まれます。

### 3. ビルド後にキャッシュを抽出

Dockerビルドが終わったら、更新されたキャッシュを取り出します：

```yaml
- name: Extract Next.js cache from container
  run: |
    CONTAINER_ID="$(docker create ${{ env.DOCKER_IMAGE_REPO }}:${{ env.DOCKER_IMAGE_TAG }})"
    mkdir -p .next
    docker cp "${CONTAINER_ID}:/app/.next/cache" .next/cache
    docker rm "${CONTAINER_ID}"
```

GitHub Actionsが、ジョブ終了時に自動でこれを保存してくれます。

## ハマりポイント：キャッシュキーの設定ミス

実装してテストしたとき、最初は全然速くなりませんでした。

原因は、キャッシュキーに`${{ github.sha }}`を使っていたこと：

```yaml
key: ${{ runner.os }}-nextjs-docker-${{ github.sha }} # ❌ NG！
```

これだと、**コミットごとに別のキャッシュキー**が生成されるので、毎回新規キャッシュ扱いになります。全く意味がない。

### 正しいキャッシュキーの設定

キャッシュキーは`package-lock.json`のハッシュをベースにするのが正解：

```yaml
key: ${{ runner.os }}-nextjs-docker-${{ hashFiles('**/package-lock.json') }} # ✅ これが正解！
```

こうすることで：
- 依存関係が変わらない限り、同じキャッシュキーが使われる
- コミットをまたいでキャッシュが再利用される
- `package-lock.json`が更新されたときだけキャッシュが無効化される

これで、ようやくキャッシュが効くようになりました！

## 全体の流れ

```
┌─────────────────────────────────────┐
    1. GitHub Actionsがキャッシュリストア  
    .next/cache → ランナーのファイル    
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
    2. Dockerビルドでキャッシュをコピー     
    COPY . . → キャッシュも含まれる     
    npm run build → キャッシュを再利用  
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
    3. 更新されたキャッシュを抽出           
    docker cp → コンテナからランナーへ   
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
    4. GHAが自動保存（次回ビルド用）         
└─────────────────────────────────────┘
```

## 完全なワークフロー例

```yaml
- name: Cache Next.js build
  uses: actions/cache@v4
  with:
    path: .next/cache
    key: ${{ runner.os }}-nextjs-docker-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-nextjs-docker-

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build image
  uses: docker/build-push-action@v6
  with:
    context: .
    file: dockerfiles/nextjs/Dockerfile
    load: true
    tags: my-app:latest
    outputs: type=docker
    cache-from: type=gha
    cache-to: type=gha,mode=max

- name: Extract Next.js cache from container
  run: |
    CONTAINER_ID="$(docker create my-app:latest)"
    mkdir -p .next
    docker cp "${CONTAINER_ID}:/app/.next/cache" .next/cache
    docker rm "${CONTAINER_ID}"
```

## 結果：ビルド時間が半減した

実装後、劇的な改善が得られました：

| 項目 | Before | After | 改善率 |
|------|--------|-------|--------|
| **ビルド時間** | 25分 | 13分 | **48%削減** |
| **キャッシュサイズ** | 0 GB（毎回ゼロから） | 約2GB（再利用） | - |

![nextjs-docker-build-cache-result.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3617150/b024dfe6-5caa-423b-bc3a-db851c35c027.png)

PRを出してから結果が返ってくるまでの時間が半分になり、開発速度が大幅に向上しました。

もう「ビルド待ち」で時間を無駄にすることはありません！

## まとめ：この記事で学んだこと

### キーポイント

1. **DockerでNext.jsをビルドすると、デフォルトでは`.next/cache`が毎回消える**
   - コンテナのライフサイクルと一緒にキャッシュも消失する

2. **BuildKitのキャッシュマウント（`type=cache`）はGitHub Actionsと相性が悪い**
   - `type=gha`バックエンドはキャッシュマウントを正しくエクスポートしない

3. **解決策：キャッシュを明示的に出し入れする**
   - ビルド前：GitHub Actionsキャッシュから`.next/cache`をリストア
   - ビルド中：`.dockerignore`で除外せず、`COPY . .`でDockerに含める
   - ビルド後：`docker cp`でコンテナから抽出して、次回のために保存

4. **キャッシュキーは`package-lock.json`のハッシュを使う**
   - `github.sha`を使うと毎回別キャッシュになるので注意

### 得られた効果

- ビルド時間：25分 → 13分（**48%削減**）
- 開発体験の向上
- チーム全体の生産性アップ

## おわりに

時には、エレガントな解決策じゃなくても、**動くものが最高の解決策**です。

BuildKitのキャッシュマウントが理想的に見えても、GitHub Actionsとの組み合わせで動かないなら、`docker cp`を使ってでも目的を達成する。それが実務です。

同じ問題で困っている方の参考になれば嬉しいです！

内容について、ご意見やツッコミもお寄せいただけると嬉しいです🥰
