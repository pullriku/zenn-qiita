---
title: 'Typstでカレンダーを作った'
emoji: 📅
type: tech
topics: ["typst", "calendar"]
published: false
only: "zenn"
---

組版システム[Typst](https://typst.app/)でカレンダーを作ってみました。本記事では、完成品とその仕組みについて紹介します。

本記事で紹介するソースコードは、以下のGitHubリポジトリから閲覧することができます。

https://github.com/pullriku/typst-simple-calendar

:::details ビルド
コマンドランナーとして[`just`](https://github.com/casey/just)を使用しています。コマンドラインで以下を実行することでビルドできます。ビルド後、成果物は`dist/`ディレクトリ以下に生成されます。

```sh
just
```

`examples/`ディレクトリに見本のソースコードを配置しています。ビルドを行うと、成果物が`dist/examples/`ディレクトリに生成されます。
:::

## 完成品

以下ような見た目のPDFが生成されます。写真が上半分にあり、下半分にカレンダーの表があります。日本の祝日が緑色で表示されています。

![8月のカレンダー。ページの上半分に写真がある。写真は海の背景。手前に船、奥に島がある。写真の右下に大きく「8月」と書かれている。ページの下半分にカレンダーの表がある。数字の色は日曜日は赤色、土曜日は青色、祝日は緑色、それ以外は黒色。11日に「山の日」と書かれている。](/images/typst-simple-cal/aug.png)

写真をページの背景全体に表示することもできます。

![5月のカレンダー。ページの背景全体に写真がある。写真は中央に滝が写っている。カレンダーの表は半透明の白背景で表示されている。5月3日（憲法記念日）、5月4日（みどりの日）、5月5日（こどもの日）、5月6日（休日）が祝日として緑色で表示されている。](/images/typst-simple-cal/may.png)

これらのようなカレンダーは、以下のようなコードで生成できます。`year_calendar`関数に年、フォント、画像を渡すと、カレンダーが生成されます。特定の月のカレンダーは`month_calendar`関数を使って生成できます。

```js:examples/calendar2/calendar2.typ
#import "/src/lib.typ" as lib: full_area, half_area, single_image, double_image, quadruple_image

#set page(margin: 0pt)

#let lake = "/examples/calendar2/images/lake.jpeg"
#let sunset = "/examples/calendar2/images/sunset.jpeg"

#lib.year_calendar(
  2025,
  font: "Noto Sans JP",
  images: (
    half_area(single_image(lake)),
    full_area(single_image(sunset)),
    half_area(double_image(lake, sunset)),
    full_area(double_image(lake, sunset)),
    half_area(quadruple_image(lake, sunset, lake, sunset)),
    full_area(quadruple_image(lake, sunset, lake, sunset)),
  ),
)
```

## 仕組み

