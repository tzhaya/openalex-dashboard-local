# Changelog

**最終更新日:** 2026年2月24日

## 修正内容

### Issue #1: 各種ランキング・統計値が最大200件のみから集計されている

**対応内容:**
取得した200件を対象とした集計から、OpenAlex APIの `group_by` パラメータを使用して、全レコードから集計する方法に変更しました。

#### 国別分布（国際共同研究 - 国別分布）
- **変更前:** `works` の配列内から200件までを対象にクライアント側で集計
- **変更後:** `group_by: 'authorships.institutions.country_code'` でサーバー側で全件集計
- **グラフ表示:** 上位10カ国を表示
- **統計値「共同研究国数」:** 全国数を表示

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 848-851)

#### 機関ランキング（共同研究機関 TOP 15）
- **変更前:** クライアント側で200件までのデータから重複排除・集計
- **変更後:** `group_by: 'authorships.institutions.id'` でサーバー側で全件集計し、自機関を除外
  - フィルタ条件の改善：`:!` 無効なnegation演算子を廃止
  - シンプルなフィルタロジック：JavaScriptで自機関を除外

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 856-869)

#### 著者ランキング（主要共著者 TOP 10）
- **変更前:** クライアント側で200件までのデータから著者名で集計
- **変更後:** `group_by: 'authorships.author.id'` でサーバー側で全件集計、著者詳細情報は `/authors` APIで一度に取得

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 873-906)

### Issue #2: 論文一覧の「被引用数」「引用数」のリンク先が逆

**対応内容:**
OpenAlexのフィルタパラメータを修正し、被引用数と引用数のリンク先を正しく対応させました。

- **被引用数リンク:** `cited_by:${workId}` （このペーパーを引用している論文を検索）
- **引用数リンク:** `cites:${workId}` （このペーパーが引用している論文を検索）

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 1182, 1195)

### Issue #3: 研究トピック集計の全件化

**対応内容:**
研究トピック集計を修正し、primary_topic に基づいて集計するよう変更しました。これにより、複数トピック該当による重複カウントを排除し、より正確な統計値を表示します。

#### グラフ「研究トピック TOP 10」
- **変更前:** `group_by: 'topics.id'` で全トピックを集計（複数該当で重複計上）
- **変更後:** `group_by: 'primary_topic.id'` でprimary_topicのみ集計

**例:** 一つの論文が「Rice Cultivation」と「Agricultural Sustainability」のトピックに該当する場合、primary_topic「Rice Cultivation」のみ集計します。

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 841-843)

#### 論文一覧テーブル「研究トピック」列
- **変更前:** `paper.topics` から最初の3個を表示
- **変更後:** `paper.primary_topic?.display_name` のprimary_topicのみ表示（単一表示）

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 1136)

### Issue #4: 共同研究機関 TOP 15 からの自機関確実除外

**対応内容:**
機関IDの正規化比較を導入し、URL形式と生ID形式の差異による自機関漏れを防止しました。

- **変更前:** 単純な文字列比較（形式違いで漏れる可能性）
- **変更後:** `normalizeKey()` で URL プリフィックスを削除し大文字小文字統一してから比較

```javascript
const normalizeKey = k => (k || '').toString()
  .replace(/^https?:\/\/openalex\.org\//i, '')
  .toLowerCase();
const ownIdNorm = normalizeKey(CONFIG.OPENALEX_ID);
topInstitutions = institutionData.group_by
  .filter(item => normalizeKey(item.key) !== ownIdNorm)
  .slice(0, 15)
```

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 856-869)

### Issue #5: グラフの表示方式改善

#### 年次論文数推移グラフ（直線化）
- **変更前:** 曲線グラフ（tension: 0.4）
- **変更後:** 直線グラフ（tension: 0）

このにより、データポイント間が直線で結ばれ、年次推移の変動がより明確に可視化されます。

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 964, 972)

## ファイル変更一覧

| ファイル | 変更箇所 | 説明 |
|--------|--------|------|
| `dashboard_template.html` | Line 822, 1329 | API select に primary_topic を追加 |
| `dashboard_template.html` | Line 841-843 | トピック集計を primary_topic.id に変更 |
| `dashboard_template.html` | Line 856-869 | 機関ランキングで自機関を正規化キーで除外 |
| `dashboard_template.html` | Line 1136 | 論文一覧トピック表示を primary_topic に変更 |
| `dashboard_template.html` | Line 1182, 1195 | Citation links を修正（被引用/引用） |
| `dashboard_template.html` | Line 964, 972 | 年次推移グラフを直線化（tension: 0） |

## 技術詳細

### group_by パラメータの活用

```javascript
// 例：国別分布の集計
const countryData = await fetchOpenAlexData('/works', {
    filter: filterStr,
    'group_by': 'authorships.institutions.country_code'
});
const totalCountries = countryData.group_by?.length || 0;
const countryCollaboration = countryData.group_by.map(item => ({
    country: item.key_display_name || item.key,
    count: item.count
})).sort((a, b) => b.count - a.count).slice(0, 10);
```

**利点:**
- 200件制限を超える全データから集計
- サーバー側処理のため高速
- メモリ効率が良い
- クライアント側の複雑な集計処理を削減

### 国別分布の表示方針

- **グラフ:** 上位10カ国のみ表示（視認性向上）
- **統計値「共同研究国数」:** 全国数を表示（正確な統計情報）

## 検証方法

1. API_KEY と OPENALEX_ID を [dashboard_template.html](dashboard_template.html) の CONFIG セクションに設定
2. ブラウザで dashboard_template.html を開く
3. 任意の機関を検索し、データが読み込まれることを確認
4. 統計値と集計結果が、年号範囲変更で正しく更新されることを確認
5. コンソール（F12）で以下のログで確認：
   - 「国別分布APIレスポンス件数」- API返却件数
   - 表示チャート - 上位10カ国のみ
   - 統計値「共同研究国数」 - 全国数

## 今後の検討項目

- [ ] 世界地図ヒートマップ表示（技術的課題：CDNライブラリの互換性）
- [ ] 集計期間のUIコンポーネント改善（カレンダーピッカー等）
- [x] パフォーマンス最適化（APIコール並列化）
- [ ] パフォーマンス最適化（キャッシング等）

### Issue #6: 共同研究国数の過小計上（2026-02-16）

**原因:**
国別分布の `group_by` クエリに `per-page` が未指定だったため、APIのデフォルト値（25）が適用されていた。共同研究国が26カ国以上ある場合、超えた分が取得されず、「共同研究国数」が実際より少なく表示されていた。

なお、機関ランキングの同様のクエリには `'per-page': 200` が既に指定されていたが、国別集計には漏れていた。

**変更前:**
```javascript
const countryData = await fetchOpenAlexData('/works', {
    filter: filterStr,
    'group_by': 'authorships.institutions.country_code'
});
```

**変更後:**
```javascript
const countryData = await fetchOpenAlexData('/works', {
    filter: filterStr,
    'group_by': 'authorships.institutions.country_code',
    'per-page': 200
});
```

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 848)

### Issue #7: 平均被引用数が200件サンプルからの計算になっている（2026-02-24）

**原因:**
`per-page: 200` で取得した最大200件の論文から平均被引用数を算出していたため、総論文数が200件を超える機関では不正確な値が表示されていた。また、APIのデフォルト順（関連度順）の先頭200件を対象としており、代表性も低かった。

**対応内容:**
`group_by=cited_by_count` とカーソルページネーションを使った専用関数 `fetchCitationStats()` を新設し、全論文の被引用総数を正確に算出するよう変更した。

- 被引用総数 = `Σ(cited_by_count値 × その値を持つ論文数)`
- 平均被引用数 = 被引用総数 ÷ `meta.count`（総論文数、従来より正確）

**変更前:**
```javascript
// 200件のみから算出（不正確）
const avgCitations = works.length > 0
    ? (works.reduce((sum, w) => sum + (w.cited_by_count || 0), 0) / works.length).toFixed(1)
    : 0;
```

**変更後:**
```javascript
// fetchCitationStats() で全件集計（正確）
async function fetchCitationStats(filterStr) {
    let totalCited = 0;
    let cursor = '*';
    while (cursor) {
        const data = await fetchOpenAlexData('/works', {
            filter: filterStr, 'group_by': 'cited_by_count', 'per-page': 200, cursor
        });
        for (const g of (data.group_by || [])) {
            const c = parseInt(g.key, 10);
            if (!isNaN(c)) totalCited += c * g.count;
        }
        cursor = data.meta?.next_cursor || null;
    }
    return totalCited;
}
// ...
const avgCitations = totalWorks > 0 ? (totalCited / totalWorks).toFixed(1) : '0';
```

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 788-806: 新関数追加, Line 873: avgCitations算出変更)

---

### Issue #8: APIコールが逐次実行でロードが遅い（2026-02-24）

**原因:**
`loadData()` 内の8つのAPIコール（年次推移・OA推移・トピック・OA状況・国別・機関別・著者別・被引用統計）が `await` で順番に実行されており、合計待機時間が各リクエスト時間の合計になっていた。

**対応内容:**
独立した8つのAPIコールを `Promise.all` で並列実行するよう変更した。著者詳細情報の取得（`/authors` エンドポイント）は著者IDの取得結果に依存するため、`Promise.all` の後に続けて実行する従来の構造を維持した。

**変更前:**
```javascript
const worksData = await fetchOpenAlexData('/works', { ... });
const yearlyData = await fetchOpenAlexData('/works', { ... });
const yearlyOaData = await fetchOpenAlexData('/works', { ... });
// ... 以下続く（合計8回の逐次 await）
```

**変更後:**
```javascript
const [worksData, yearlyData, yearlyOaData, topicData, oaData,
       countryData, institutionData, authorData, totalCited]
    = await Promise.all([
    fetchOpenAlexData('/works', { ... }),
    fetchOpenAlexData('/works', { ... }),
    // ... 8件並列
    fetchCitationStats(filterStr)   // Issue #7 の修正も統合
]);
```

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 839-869: Promise.all に統合)

---

### Issue #9: 平均被引用数カードのデータソースとラベル変更（2026-02-24）

**対応内容:**
`fetchCitationStats` によるカーソルページネーション（大規模機関で数秒かかる）を廃止し、`loadInstitutionInfo()` で初回取得済みの `institutionInfo` を流用するよう変更。追加APIコスト0。

- **平均被引用数** = `institutionInfo.cited_by_count / institutionInfo.works_count`（全期間・全論文対象）
- フィルター条件（年・OA・種別）は反映されないため、UIラベルを「平均被引用数（全期間）」に変更

**変更前:**
```javascript
// Promise.all 内で fetchCitationStats(filterStr) を呼び出し（カーソルループで複数APIコール）
const avgCitations = totalWorks > 0 ? (totalCited / totalWorks).toFixed(1) : '0';
```

**変更後:**
```javascript
// 追加APIコストなし、初回ロード済みデータを流用
const avgCitations = (institutionInfo?.works_count > 0)
    ? (institutionInfo.cited_by_count / institutionInfo.works_count).toFixed(1)
    : '0';
```

**影響ファイル:**
- [dashboard_template.html](dashboard_template.html) (Line 584: ラベル変更, Line 820-853: Promise.all から fetchCitationStats 削除・avgCitations 算出変更)

---

## 変更履歴

| 日付 | 内容 |
|------|------|
| 2026-02-24 | **Issue #9 修正完了** - 平均被引用数を institutionInfo から取得・ラベルに「全期間」付加 |
| 2026-02-24 | **Issue #7, #8 修正完了** - 平均被引用数の全件集計対応・APIコール並列化 |
| 2026-02-16 | **Issue #6 修正完了** - 共同研究国数の過小計上修正（`per-page:200` 追加） |
| 2026-02-10 | **Issue #1, #2 修正完了** - group_by集計実装、citation links修正 |
| 2026-02-10 | **Issue #3 修正完了** - primary_topic集計に統一（グラフ・論文一覧） |
| 2026-02-10 | **Issue #4 修正完了** - 機関ランキングで自機関を正規化キーで確実除外 |
| 2026-02-10 | **Issue #5 修正完了** - 年次推移グラフを直線化 |

