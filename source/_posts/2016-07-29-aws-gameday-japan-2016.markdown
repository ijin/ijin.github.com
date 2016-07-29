---
layout: post
title: "AWS GameDay Japan 2016を開催してきた"
date: 2016-07-29 15:41
comments: true
categories: 
- aws
- events
- study
---

過去に何回も参加・開催両方を経験した事がある[AWS GameDay](http://gameday-japan.connpass.com/event/33531/)に、またしても運営側として関わりました。

お題は基本的に[去年のre:Inventでやった](/blog/2015/10/26/aws-re-invent-2015/)のを若干チューニングしたやつ。
詳細は今後また別のところで開催される可能性があるので、その時の参加者の為に伏せておきます。

### 当日 ###

競技中はそれぞれのチームのスコアがリアルタイムで見れるダッシュボードがあって白熱した様子が伝わってました。

{% tweet https://twitter.com/ijin/status/756717339312164864?lang=ja %}

これはこれで楽しいんですが、何かが足りない感じ。。。

<br >
<br >
<br >
<br >
<br >

そう、**グラフ**です！!

<br >
<br >
<br >
<br >
<br >

参加チームの白熱したバトル（順位の入れ替え等）を時系列で表示し、戦いの軌跡をビジュアライズするアレです。ISUCONでもよく見慣れたあの遷移するグラフがなかったのです！

開始してから気づきました。。

# なので作りました #

## Graphing premiere ##

ダッシュボードの作りを見てみると `jQuery` と `React.js` を使っている模様。ならば、どこかでendpointを叩いてjsonを取得しているはずなので、いろいろ探したところ、それらしいのがありました。

```javascript
  getScores: function() {
    $.ajax({
      url: 'https://xxxxxxxxx.execute-api.us-west-2.amazonaws.com/prod/scores',
      dataType: 'json',
      cache: false,
      success: function(data) {
        this.setState({data: data});
      }.bind(this),
      error: function(xhr, status, err) {
        console.error("scores", status, err.toString());
      }.bind(this)
    });
  },
```

こうなれば話は簡単で後は返却されるjsonを解析して、定期的にcallしてplotしていけば良いだけです。


### データ送信 ###

グラフを描画するサーバを用意するのはしんどいので、カスタムメトリックスが作成できる監視サービスを使いました。
最初は[Mackerel](https://mackerel.io)をと思ったけど、独自グラフを一般公開する設定がなさそうだったので[Datadog](https://www.datadoghq.com)を採用。

Datadogでカスタムメトリックスを送るにはAPIを直接叩くよりは、[StatsD経由で送信](http://docs.datadoghq.com/guides/metrics/)した方が楽なのでrubyでサクっと適当に記述。

``` ruby gameday.rb
statsd.batch do |s|
  s.gauge('gameday2016', scores[0]['Profit']*100, :tags => ["team:" + scores[0]['Team']])
  s.gauge('gameday2016', scores[1]['Profit']*100, :tags => ["team:" + scores[1]['Team']])
  s.gauge('gameday2016', scores[2]['Profit']*100, :tags => ["team:" + scores[2]['Team']])
  # etc
end
```

### グラフ描画 ###

公開ダッシュボードを作成するには `TimeBoard` ではなく `ScreenBoard` を選択し、後は必要そうなグラフを追加していくだけ。

グラフ自体はGUIで作っても良いし、jsonで記述可能なので結構柔軟で素敵です。

``` json profits.json
{
  "viz": "timeseries",
  "requests": [
    {
      "q": "avg:gameday2016{*} by {team}",
      "aggregator": "avg",
      "conditional_formats": [],
      "type": "line"
    }
  ]
}
```

``` json ranking.json
{
  "viz": "toplist",
  "requests": [
    {
      "q": "top(avg:gameday2016{*} by {team}, 20, 'last', 'desc')",
      "style": {
        "palette": "dog_classic"
      },
      "conditional_formats": [
        {
          "palette": "white_on_green",
          "comparator": ">=",
          "value": 0
        },
        {
          "palette": "white_on_red",
          "comparator": "<",
          "value": 0
        }
      ]
    }
  ]
}
```

できあがったグラフはこんな感じ。
高負荷発生等のイベント時の対応との比較もできて見やすいと思います。

{% img https://lh3.googleusercontent.com/XoNcTN3HQIlFDHvaSe8uNlcCP8FYRijZNsPVdrnrsmNvEF3XoW6IUDKnU63JDLL7X6W3Ed3mCLaF=w742-h394-no %}


これで各チームのポジション等を伝えやすくなりました。

{% tweet https://twitter.com/ijin/status/756756052192792576?lang=ja %}
{% tweet https://twitter.com/ijin/status/756766379341025281?lang=ja %}
{% tweet https://twitter.com/ijin/status/756778321749225472?lang=ja %}

というわけで優勝した[チーム初老丸](http://inokara.hateblo.jp/entry/2016/07/24/095228)、おめでとうございます！


### 終わりに ###

今回は競技の途中から実装しちゃったので次回は最初から用意しておきたいと思います。

{% tweet https://twitter.com/sora_h/status/660379055867334656?lang=ja %}
（※）実は去年のISUCONの時も似たような事をやってましたね。。

