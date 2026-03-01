---
description: microCMSのニュースセクションをimputのHTMLに埋め込む（Vercel Serverless Function経由）
---

以下の手順で実装してください。

### Step 1: ユーザーへの確認（ask_user_input スタイル）

以下を質問してください:

- microCMS のサービスID（例: `your-service`）
- APIエンドポイント名（デフォルト: `news`）
- ニュースセクションを挿入するセクション（どのセクションの直前に置くか。`input/index.html` のid属性一覧を提示して選ばせること）
- 1ページあたりの表示件数（デフォルト: `5`）
- ページング方式（以下の3択から選ばせること）:
  - **なし** — 最新N件のみ表示
  - **もっと見るボタン** — クリックで追加ロード
  - **ページネーションUI** — 前へ／次へ＋ページ番号ボタン

### Step 2: 構造把握

`imput/index.html` を読み込み、サイトのセクション構成（id属性一覧）を把握してください。

### Step 3: `imput/index.html` の変更

以下3点を適用してください:

**① `<head>` に追加:**

```html
<link rel="stylesheet" href="css/news.css" />
```

**② 指定セクションの直前にニュースセクションを挿入:**

ページング方式に応じてHTMLを切り替えてください。

**【なし】または【もっと見るボタン】の場合:**

```html
<section id="news" class="section sd appear">
  <div class="section-inner sd appear">
    <h2 class="text sd appear">News</h2>
    <p class="text sd appear">お知らせ</p>
    <div id="news-list" class="news-list">
      <p class="news-loading">読み込み中...</p>
    </div>
    <!-- もっと見るボタン方式のみ追加 -->
    <button id="news-more" class="news-more-btn" hidden>もっと見る</button>
  </div>
</section>
```

※「なし」の場合は `<button id="news-more">` 行は不要。

**【ページネーションUI】の場合:**

```html
<section id="news" class="section sd appear">
  <div class="section-inner sd appear">
    <h2 class="text sd appear">News</h2>
    <p class="text sd appear">お知らせ</p>
    <div id="news-list" class="news-list">
      <p class="news-loading">読み込み中...</p>
    </div>
    <div id="news-pagination" class="news-pagination"></div>
  </div>
</section>
```

**③ `</body>` 直前にJSを追加（ページング方式に応じて切り替え）:**

**【なし】の場合:**

```html
<script>
  (async () => {
    const container = document.getElementById('news-list');
    const LIMIT = 5; // Step 1 で確認した件数に変更
    try {
      const res = await fetch(`/api/news?limit=${LIMIT}&offset=0`);
      if (!res.ok) throw new Error(res.statusText);
      const { contents } = await res.json();
      if (contents.length === 0) {
        container.innerHTML = '<p>現在お知らせはありません。</p>';
        return;
      }
      container.innerHTML = contents.map(item => {
        const date = new Date(item.publishedAt).toLocaleDateString('ja-JP');
        return `
          <article class="news-item">
            <time class="news-date" datetime="${item.publishedAt}">${date}</time>
            <h3 class="news-title">${item.title}</h3>
            <div class="news-body">${item.content}</div>
          </article>`;
      }).join('');
    } catch (e) {
      container.innerHTML = '<p>お知らせの読み込みに失敗しました。</p>';
    }
  })();
</script>
```

**【もっと見るボタン】の場合:**

```html
<script>
  (async () => {
    const container = document.getElementById('news-list');
    const btn = document.getElementById('news-more');
    const LIMIT = 5; // Step 1 で確認した件数に変更
    let offset = 0;
    let total = Infinity;

    async function loadMore() {
      if (offset >= total) return;
      btn.disabled = true;
      try {
        const res = await fetch(`/api/news?limit=${LIMIT}&offset=${offset}`);
        if (!res.ok) throw new Error(res.statusText);
        const { contents, totalCount } = await res.json();
        total = totalCount;

        const loading = container.querySelector('.news-loading');
        if (loading) loading.remove();

        if (offset === 0 && contents.length === 0) {
          container.innerHTML = '<p>現在お知らせはありません。</p>';
          btn.hidden = true;
          return;
        }

        container.insertAdjacentHTML('beforeend', contents.map(item => {
          const date = new Date(item.publishedAt).toLocaleDateString('ja-JP');
          return `
            <article class="news-item">
              <time class="news-date" datetime="${item.publishedAt}">${date}</time>
              <h3 class="news-title">${item.title}</h3>
              <div class="news-body">${item.content}</div>
            </article>`;
        }).join(''));

        offset += contents.length;

        if (offset >= total) {
          btn.hidden = true;
        } else {
          btn.hidden = false;
          btn.disabled = false;
        }
      } catch (e) {
        container.innerHTML = '<p>お知らせの読み込みに失敗しました。</p>';
      }
    }

    btn.addEventListener('click', loadMore);
    loadMore();
  })();
</script>
```

**【ページネーションUI】の場合:**

```html
<script>
  (async () => {
    const container = document.getElementById('news-list');
    const pagination = document.getElementById('news-pagination');
    const LIMIT = 5; // Step 1 で確認した件数に変更
    let currentPage = 1;
    let totalCount = 0;

    async function loadPage(page) {
      const offset = (page - 1) * LIMIT;
      container.innerHTML = '<p class="news-loading">読み込み中...</p>';
      try {
        const res = await fetch(`/api/news?limit=${LIMIT}&offset=${offset}`);
        if (!res.ok) throw new Error(res.statusText);
        const data = await res.json();
        totalCount = data.totalCount;
        currentPage = page;

        if (data.contents.length === 0) {
          container.innerHTML = '<p>現在お知らせはありません。</p>';
          pagination.innerHTML = '';
          return;
        }

        container.innerHTML = data.contents.map(item => {
          const date = new Date(item.publishedAt).toLocaleDateString('ja-JP');
          return `
            <article class="news-item">
              <time class="news-date" datetime="${item.publishedAt}">${date}</time>
              <h3 class="news-title">${item.title}</h3>
              <div class="news-body">${item.content}</div>
            </article>`;
        }).join('');

        renderPagination();
      } catch (e) {
        container.innerHTML = '<p>お知らせの読み込みに失敗しました。</p>';
      }
    }

    function renderPagination() {
      const totalPages = Math.ceil(totalCount / LIMIT);
      if (totalPages <= 1) { pagination.innerHTML = ''; return; }

      let html = '';
      if (currentPage > 1) {
        html += `<button class="news-page-btn" data-page="${currentPage - 1}">前へ</button>`;
      }
      for (let i = 1; i <= totalPages; i++) {
        html += `<button class="news-page-btn${i === currentPage ? ' active' : ''}" data-page="${i}">${i}</button>`;
      }
      if (currentPage < totalPages) {
        html += `<button class="news-page-btn" data-page="${currentPage + 1}">次へ</button>`;
      }

      pagination.innerHTML = html;
      pagination.querySelectorAll('.news-page-btn').forEach(btn => {
        btn.addEventListener('click', () => loadPage(Number(btn.dataset.page)));
      });
    }

    loadPage(1);
  })();
</script>
```

### Step 4: `output/css/news.css` を新規作成

既存のCSSファイル（`imput/css/about.css` 等）のデザイントーン・CSS変数を参照し、以下のクラスを定義してください:

- `.news-list` — 記事一覧ラッパー
- `.news-item` — 1記事分のカード（日付・タイトル・本文を含む）
- `.news-date` — 日付スタイル
- `.news-title` — 記事タイトル
- `.news-body` — 本文エリア
- `.news-loading` — ローディング表示
- `.news-more-btn` — 「もっと見る」ボタン（もっと見るボタン方式のみ）
- `.news-pagination` — ページネーションボタン群のラッパー（ページネーションUI方式のみ）
- `.news-page-btn` — ページ番号・前へ・次へボタン（ページネーションUI方式のみ）
- `.news-page-btn.active` — 現在ページのボタン（ページネーションUI方式のみ）

選択されたページング方式に不要なクラスは定義しなくて構いません。

### Step 5: `output/api/news.js` を新規作成（Vercel Serverless Function）

`output/api/` ディレクトリを作成し、以下のファイルを追加してください:
（※ `output/` の中身をまるごと GitHub へ push する構成のため、Serverless Function も `output/api/` 配下に置きます）

`limit` と `offset` はフロントエンドからクエリパラメータで受け取ります。

```javascript
export default async function handler(req, res) {
  const serviceId = process.env.MICROCMS_SERVICE_ID;
  const apiKey    = process.env.MICROCMS_API_KEY;

  if (!serviceId || !apiKey) {
    return res.status(500).json({
      error: 'Missing environment variables',
      missing: { serviceId: !serviceId, apiKey: !apiKey }
    });
  }

  const limit  = parseInt(req.query.limit  ?? '5',  10);
  const offset = parseInt(req.query.offset ?? '0', 10);
  const endpoint = `https://${serviceId}.microcms.io/api/v1/news?limit=${limit}&offset=${offset}&orders=-publishedAt`;

  try {
    const response = await fetch(endpoint, {
      headers: { 'X-MICROCMS-API-KEY': apiKey }
    });
    const data = await response.json();
    res.status(response.status).json(data);
  } catch (e) {
    res.status(500).json({ error: e.message });
  }
}
```

### Step 6: 完了後に以下を出力してください

1. **Vercel環境変数の設定値:**
   - `MICROCMS_SERVICE_ID` = （Step 1で確認したサービスID）
   - `MICROCMS_API_KEY` = （microCMS管理画面のAPIキー）

2. **microCMS管理画面で必要なAPIスキーマ定義:**
   - API形式: リスト形式
   - エンドポイント: （Step 1で確認したエンドポイント名）
   - フィールド:
     - `title`（テキストフィールド）
     - `body`（リッチテキスト）
     - ※ `publishedAt` はmicroCMSが自動付与するため定義不要

3. **CORSについて:** ブラウザは同一オリジンの `/api/news` のみ呼ぶため、microCMS側のCORS設定は不要です。
