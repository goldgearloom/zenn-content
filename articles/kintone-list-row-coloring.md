---
title: "kintoneの行色分け、2ページ目で色が崩れる理由 ― 見落としがちな4つの罠"
emoji: "🎨"
type: "tech"
topics: ["kintone", "javascript", "業務改善", "ローコード"]
published: true
---

## この記事のゴール

kintoneの一覧は、件数が増えると「どれが要対応かパッと分からない」状態になりがちです。
この記事では、**一覧画面でレコードの値に応じて行の色を変える**カスタマイズを、コピペで動く形で作ります。

例として、「ステータス」フィールドが

- `遅延` → 薄い赤
- `完了` → 薄い緑

になるようにします。値を見なくても**色で状態が分かる一覧**になります。

## 対象読者・前提

- kintoneのJavaScriptカスタマイズを触ったことがある人
- 対象は**標準の一覧画面**（カスタマイズビューは後述の注意あり）
- コードは ［JavaScript / CSSでカスタマイズ］→ PC用にアップロード で適用

## 実装（コピペOK）

`ステータス` は自分のアプリの**フィールドコード**に置き換えてください。

```javascript
(function () {
  'use strict';

  kintone.events.on('app.record.index.show', function (event) {
    // 「ステータス」列のセル要素を取得（表示中レコードと同じ並び順で返る）
    var cells = kintone.app.getFieldElements('ステータス');
    if (!cells) {
      return event; // カスタマイズビュー等では null になることがある
    }

    event.records.forEach(function (record, i) {
      var status = record['ステータス'].value;
      var row = cells[i].closest('tr'); // その行の要素
      if (!row) return;

      if (status === '遅延') {
        row.style.backgroundColor = '#fde2e2';  // 薄い赤
      } else if (status === '完了') {
        row.style.backgroundColor = '#e2f0d9';  // 薄い緑
      } else {
        row.style.backgroundColor = '';         // ← リセット（重要）
      }
    });

    return event;
  });
})();
```

## 解説

- `app.record.index.show` は**一覧が表示されるたび**に発火します
- `kintone.app.getFieldElements('フィールドコード')` は、その列のセル要素を**`event.records` と同じ順番**で返します
- セルから `closest('tr')` で行を取り、`style.backgroundColor` を設定します

数値フィールドで「金額が10万以上なら赤」のように、`Number(record['金額'].value) >= 100000` と条件を変えれば応用できます。

## ⚠️ 崩れやすいポイント（ここが差になる）

色分けは簡単に見えて、DOMを触るぶん**壊れやすい**カスタマイズです。次を押さえておくと事故りません。

### ① 「一度だけ」と思わない

`app.record.index.show` は**ページ送り・ソート・絞り込みのたびに再発火**します。
「初回だけ塗る」実装にすると、2ページ目で崩れます。**毎回、全行を塗り直す**のが正解です（上のコードはそうしています）。

### ② `else` で必ず色をリセットする

リセットを書かないと、ソートや条件変更後に**前の行の色が残ります**。
「該当しない行は空文字に戻す」を忘れないでください。

### ③ 生のクラス名でDOMを取りにいかない

`document.querySelector('.recordlist-row-gaia ...')` のように**kintone内部のクラス名に依存**すると、kintoneのアップデートで**ある日突然動かなくなります**。
できるだけ `kintone.app.getFieldElements` のような**公式API経由**でセルを取得しましょう。「動く」だけでなく「壊れにくい」書き方を選ぶのがプロの分かれ目です。

### ④ カスタマイズビューでは取得できないことがある

標準一覧を前提にしています。カスタマイズビューや未対応の画面では `getFieldElements` が `null` を返すことがあるので、**ガード（`if (!cells) return;`）**を入れておきます。

## まとめ

- 一覧の色分けは `app.record.index.show` ＋ `kintone.app.getFieldElements`
- **毎回・全行を塗り直す**／**else でリセット**
- 生のクラス名依存を避け、**公式API経由**でセルを取る（壊れにくさ重視）

## 次回予告

このシリーズでは他に、

- **保存前の入力チェック**で「そもそも変な値を入れさせない」編
- **外部APIキーを利用者に見せずに安全連携する**編

も公開予定です。続きが気になる方は**フォロー**しておくと更新が流れてきます。

## 参考

- kintone 開発者向け公式ドキュメント（cybozu developer network）… `getFieldElements` / 一覧イベントの仕様は公式で確認を
