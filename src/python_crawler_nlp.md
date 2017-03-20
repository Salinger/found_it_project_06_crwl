
# Python による「スクレイピング & 自然言語処理」入門

## 1. はじめに
　クローラーとはWeb上のデータを自動的に収集するための道具です。クローラーを活用することで、担当者が手動で行っていたWeb情報収集の効率化、また自社だけでは入手できないさまざまなデータを取得し自社データと結合することで新たな示唆を得ることが可能になります。
 
　今回のセミナーでは初心者を対象にクローラーを作成し対象サイトのデータを収集、テキスト解析を行い、分析結果を得るまでの一連の流れについて、Python で使用するライブラリ、解析手法を交えて解説いたします。
 
 ※本発表は所属する組織とは一切関係がありません

　今回は対象ページとして、[日本酒物語 日本酒ランキング（人数）](http://www.sakeno.com/followrank/) とそれに紐づく各銘柄の詳細を収集し、各種分析を行います。本解析の内容として次の項目を含みます。

 * 解析のための下準備
   * 使用するライブラリのインストール
   * 使用するライブラリの読み込み
   * 定数の設定
 * クローラーによるデータの収集
   * ランキング一覧の生の HTML 確認
   * ランキング一覧のテーブル要素の取得
   * 詳細ページのデータ取得
 * テキスト解析
   * TFIDF によるレビュー中の特徴的な形容詞の抽出
   * 単語ベースのクラスタリング

## 2. クローラーとは？

（スライド参照）

## 3. 自然言語処理とは？

（スライド参照）

## 4. 解析のための下準備
### 4. 1 使用するライブラリのインストール

　まずは形態素解析ツール MeCab のインストールを行います。ここでは Mac (OSX Sierra) を仮定して進めています。


```python
!brew install mecab mecab-ipadic
!pip install mecab-python3
```

    [33mWarning:[0m mecab-0.996 already installed
    [33mWarning:[0m mecab-ipadic-2.7.0-20070801 already installed
    Requirement already satisfied: mecab-python3 in /Users/tojima/anaconda3/lib/python3.5/site-packages


　次にクローラー関係のライブラリもインストールします。ここでは以下の3つのライブラリを導入しています。

* html5lib
* requests
* BeautifulSoup


```python
!conda install -y html5lib 
!conda install -y requests
!conda install -y BeautifulSoup4
```

    Fetching package metadata .........
    Solving package specifications: .
    
    # All requested packages already installed.
    # packages in environment at /Users/tojima/anaconda3:
    #
    html5lib                  0.999                    py35_0  
    Fetching package metadata .........
    Solving package specifications: .
    
    # All requested packages already installed.
    # packages in environment at /Users/tojima/anaconda3:
    #
    requests                  2.13.0                   py35_0  
    Fetching package metadata .........
    Solving package specifications: .
    
    # All requested packages already installed.
    # packages in environment at /Users/tojima/anaconda3:
    #
    beautifulsoup4            4.5.3                    py35_0  


### 4.2. 使用するライブラリの読み込み

　ここではこの解析に使用する各種ライブラリを読み込んでいます。


```python
# ファイル操作
import glob
import csv

# データ処理・視覚化
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# クローラー
import time
from datetime import datetime
from bs4 import BeautifulSoup
import requests

# テキスト解析
import MeCab
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA
from sklearn.cluster import AgglomerativeClustering
```

### 4.3. 定数の設定

　ここでは取得対象となっているページのURLやクローラーの待ち時間、各種データの出力先を定義しています。


```python
# 日本酒物語 日本酒ランキング（人数）の URL
FOLLOWRANK_URL = "http://www.sakeno.com/followrank/"

# クロール時の待ち時間
WAIT_TIME = 5

# 銘柄マスタの出力先
MEIGARA_MASTER_PATH = "../data/meigara_maseter.csv"
# 銘柄評価スコアの出力先ディレクトリ
MEIGARA_SCORES_DIR = "../data/meigara_scores/"
# 銘柄コメントの出力先ディレクトリ
MEIGARA_COMMENTS_DIR = "../data/meigara_comments/"

# TFIDF スコア算出後の結果出力先
TFIDF_PATH = "../data/tfidf.csv"
# クラスタリング結果の結果出力先
CLUSTER_PATH = "../data/cluster.csv"
```

## 5. クローラーによるデータの取得

　では、ここからクローラーでデータを取得するための流れを見ていきます。基本的な流れは次のようになります。
 
1. 対象ページのHTMLの取得。
2. 取得対象情報が含まれている部分のタグの特定。
3. 取得対象情報のパース。
4. 結果の保存。

では実際にクローラーのコードを見ていきましょう。

### 5.1 ランキング一覧の生の HTML 確認

　まずはランキングの一覧ページの HTML を取得します。ここでは `response` ライブラリを利用し取得します。


```python
# ページの HTML を取得
response = requests.get(FOLLOWRANK_URL)

# 正しく取得できたかどうか HTTP ステータスコードで確認
if not response.status_code == 200:
    raise ValueError("Invalid response")
else:
    print("OK.")
```

    OK.


正しくランキングページが取得されていれば `OK.` と出力されます。

次は取得してきた HTML の先頭 1000 件を見てみると…


```python
response.text[:1000]
```




    '<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">\r\n<html lang="ja">\r\n<head>\r\n<meta http-equiv="Content-Type" content="text/html; charset=euc-jp">\r\n<meta http-equiv="Content-Style-Type" content="text/css">\r\n<meta http-equiv="Content-Script-Type" content="text/javascript">\r\n<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=4">\r\n<meta name="keywords" content="ÆüËÜ¼ò,¥é¥ó¥\xad¥ó¥°,¸ý¥³¥ß,É¾²Á">\r\n<title>ÆüËÜ¼ò¥é¥ó¥\xad¥ó¥°¡Ê¿Í¿ô¡Ë¡ÝÆüËÜ¼òÊª¸ì</title>\r\n<link rel="stylesheet" href="http://www.sakeno.com/incfiles/sakeno.css">\r\n<link rel="stylesheet" media="screen and (max-width:800px)" href="http://www.sakeno.com/incfiles/sakeno_sp.css">\r\n</head>\r\n\r\n<body><a id="top" name="top"></a>\r\n<div id="bigbox">\r\n\r\n\t<div id="header"><div id="headertitle"><h1><a href="http://www.sakeno.com/"><img border="0" src="http://www.sakeno.com/images/logo_new.gif" width="246" height="54" alt="ÆüËÜ¼òÊª¸ì"></a></h1></div><div id="headerlog'



このように対象ページの HTML が文字列として取得できたが、文字化けしている。文字列の先頭の方を見てみると、`charset=euc-jp`という文字列が見えるので、euc-jpということを考慮して扱ってみる。


```python
response.encoding = 'euc_jp'
print(response.text[:1000])
```

    <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
    <html lang="ja">
    <head>
    <meta http-equiv="Content-Type" content="text/html; charset=euc-jp">
    <meta http-equiv="Content-Style-Type" content="text/css">
    <meta http-equiv="Content-Script-Type" content="text/javascript">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=4">
    <meta name="keywords" content="日本酒,ランキング,口コミ,評価">
    <title>日本酒ランキング（人数）−日本酒物語</title>
    <link rel="stylesheet" href="http://www.sakeno.com/incfiles/sakeno.css">
    <link rel="stylesheet" media="screen and (max-width:800px)" href="http://www.sakeno.com/incfiles/sakeno_sp.css">
    </head>
    
    <body><a id="top" name="top"></a>
    <div id="bigbox">
    
    	<div id="header"><div id="headertitle"><h1><a href="http://www.sakeno.com/"><img border="0" src="http://www.sakeno.com/images/logo_new.gif" width="246" height="54" alt="日本酒物語"></a></h1></div><div id="headerlogin"><a href="http://www.sakeno.com/l


正しく出力された。

### 5.2 ランキング一覧のテーブル要素の取得

　今度はこのページから `find` メソッドを利用し、 `<table> 〜 </table>` の部分を抜き出します。この処理には `BeautifulSoup` を利用します。


```python
soup = BeautifulSoup(response.text, "lxml")
table = soup.body.find("table")
```

　次はテーブルの中の行要素を全て取得します。`<tr> 〜 </tr>` の部分が各行に該当します。`find_all`メソッドを利用すれば各行が1要素となったリスト構造として取得可能です。


```python
trs = table.find_all("tr")[2:] # 先頭のゴミをカット
```

各行から必要な数値、文字列を抜き出します。


```python
# ある行の情報をパースし以下の要素を取得する。
#
#   [ 順位, 銘柄, 銘柄の読み, 
#     蔵元, 蔵元の県, 蔵元の市町村,
#     銘柄詳細ページのURL ]
#
def parse_tr(tr):
    # 順位
    tds = tr.find_all("td")
    rank = int(tds[0].get_text().split("位")[0]) 

    # 銘柄
    a = tds[1].find("a")
    meigara = a.get_text()
    detail_url = a.get("href")
    yomi = tds[1].find("div").string

    # 酒造
    location = tds[2].find_all("a")
    kuramoto = location[0].string
    prefecture = location[1].string
    city = location[2].string
    
    tr_l = [
        rank, meigara, yomi,
        kuramoto, prefecture, city,
        detail_url
    ]
    return tr_l


ranking_list = [parse_tr(tr) for tr in trs]
ranking_list[:5]
```




    [[1,
      '獺祭',
      'だっさい',
      '旭酒造（山口県）',
      '山口県',
      '岩国市',
      'http://www.sakeno.com/meigara/931'],
     [2,
      '醸し人九平次',
      'かもしびとくへいじ',
      '萬乗醸造',
      '愛知県',
      '名古屋市',
      'http://www.sakeno.com/meigara/735'],
     [3,
      '出羽桜',
      'でわざくら',
      '出羽桜酒造',
      '山形県',
      '天童市',
      'http://www.sakeno.com/meigara/219'],
     [4, '田酒', 'でんしゅ', '西田酒造店', '青森県', '青森市', 'http://www.sakeno.com/meigara/11'],
     [5, '黒龍', 'こくりゅう', '黒龍酒造', '福井県', '吉田郡', 'http://www.sakeno.com/meigara/667']]



あとの処理のためにデータフレームに変換します。この際にユニークな連番IDを付加します。


```python
meigara_master_df = pd.DataFrame(
    ranking_list,
    columns=["rank", "meigara", "yomi",
             "kuramoto", "prefecture", "city",
             "detail_url"]
)

# ユニークな連番 ID を追加
meigara_master_df["meigara_id"] = meigara_master_df.index.to_series() + 1
meigara_master_df = meigara_master_df[
    ["meigara_id",
     "rank", "meigara", "yomi",
     "kuramoto", "prefecture", "city",
     "detail_url"]
]

# 銘柄マスタデータの出力
meigara_master_df.to_csv(
    MEIGARA_MASTER_PATH,
    encoding="utf-8",
    sep=",",
    index=False
)

meigara_master_df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>meigara_id</th>
      <th>rank</th>
      <th>meigara</th>
      <th>yomi</th>
      <th>kuramoto</th>
      <th>prefecture</th>
      <th>city</th>
      <th>detail_url</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>1</td>
      <td>獺祭</td>
      <td>だっさい</td>
      <td>旭酒造（山口県）</td>
      <td>山口県</td>
      <td>岩国市</td>
      <td>http://www.sakeno.com/meigara/931</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>2</td>
      <td>醸し人九平次</td>
      <td>かもしびとくへいじ</td>
      <td>萬乗醸造</td>
      <td>愛知県</td>
      <td>名古屋市</td>
      <td>http://www.sakeno.com/meigara/735</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>3</td>
      <td>出羽桜</td>
      <td>でわざくら</td>
      <td>出羽桜酒造</td>
      <td>山形県</td>
      <td>天童市</td>
      <td>http://www.sakeno.com/meigara/219</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>4</td>
      <td>田酒</td>
      <td>でんしゅ</td>
      <td>西田酒造店</td>
      <td>青森県</td>
      <td>青森市</td>
      <td>http://www.sakeno.com/meigara/11</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>5</td>
      <td>黒龍</td>
      <td>こくりゅう</td>
      <td>黒龍酒造</td>
      <td>福井県</td>
      <td>吉田郡</td>
      <td>http://www.sakeno.com/meigara/667</td>
    </tr>
  </tbody>
</table>
</div>



ここまでで、ランキング一覧の結果が無事取得できました。

### 5.3 詳細ページのデータ取得

　次は先程取得したランキング一覧のデータを利用して、詳細ページ（`detail_url`）から以下の情報を取得します。
 * 数値による評価データ (良い／悪い)
   * 味
   * 香り
   * 濃さ
   * 価格
   * デザイン
 * コメント一覧
   * 投稿ID
   * タイトル
   * 投稿日時
   * ユーザ名
   * テキスト

これらのデータを集めるために、以下に次の3つの関数を記述しています。

* 数値データによる評価データ取得用の関数
* コメント一覧取得用の関数
* すべての詳細ページからデータを取得するための関数


```python
# 数値による評価データ取得関数
def parse_scores_table(soup, meigara_id):
    scores_table = soup.body.find_all("form")[1]
    trs = scores_table.find_all("tr")

    # 味
    aji = trs[2].find_all("span")
    aji_good = int(aji[0].string)
    aji_bad = int(aji[1].string)

    # 香り
    kaori = trs[3].find_all("span")
    kaori_good = int(kaori[0].string)
    kaori_bad = int(kaori[1].string)

    # 濃さ
    kosa = trs[4].find_all("span")
    kosa_good = int(kosa[0].string)
    kosa_bad = int(kosa[1].string)

    # 価格
    kakaku = trs[5].find_all("span")
    kakaku_good = int(kakaku[0].string)
    kakaku_bad = int(kakaku[1].string)

    # デザイン
    design = trs[6].find_all("span")
    design_good = int(design[0].string)
    design_bad = int(design[1].string)

    score_li = [
        [meigara_id, "味", aji_good, aji_bad],
        [meigara_id, "香り", kaori_good, kaori_bad],
        [meigara_id, "濃さ", kosa_good, kosa_bad],
        [meigara_id, "価格", kakaku_good, kakaku_bad],
        [meigara_id, "デザイン", design_good, design_bad]
    ]

    score_df = pd.DataFrame(
        score_li,
        columns=["meigara_id", "name", "good_score", "bad_score"]
    )
    
    # (index)  name   good_score bad_score
    #       0  味           1123       260
    #       1  香り         1095       250
    #       2  濃さ          978       304
    #       3  価格          950       344
    #       4  デザイン       975       249
    
    return score_df
```


```python
# コメント一覧取得関数
def parse_comments_table(soup, meigara_id):
    reviews_table = soup.body.find_all("table")[-1]

    # 以下のような構造になっているため、dtのみ、ddのみで処理し、
    #  最後に concat で横方向に単純結合する
    #
    #   <dt>〜</dt><dd>〜</dd>
    #   <dt>〜</dt><dd>〜</dd>
    #   <dt>〜</dt><dd>〜</dd>
    #   ...
    #

    # <dt>〜</dt> の処理
    dts = [
        [meigara_id,
         int(dt.contents[0].get("name").replace("voice", "")),
         dt.contents[3].string]
        for dt
        in reviews_table.find_all("dt")
    ]
    dts_df = pd.DataFrame(
        dts,
        columns=["meigara_id", "toukou_id", "title"]
    )
    
    # <dd>〜</dd> の処理
    dds = [
        [dd.contents[-1].text.split("（")[1].split("）")[0],
         dd.contents[-1].find("a").string,
         dd.contents[-2].replace("\n", " ")]
        for dd
        in reviews_table.find_all("dd")
    ]
    dds_df = pd.DataFrame(
        dds,
        columns=["created_at", "user_name", "text"]
    )
    dds_df["created_at"] = dds_df["created_at"].apply(
        lambda x: datetime.strptime(x, '%Y年%m月%d日 %H時%M分%S秒')
        )
    
    # 結合
    comments_df = pd.concat([dts_df, dds_df], axis=1)
    
    # (index) meigara_id toukou_id title              created_at           user_name  text
    #       0          1      6193 すっきりして飲みやすい 2016-10-20 11:57:54  あいうそん   おいしいです
    #   ...

    return comments_df
```


```python
# すべての詳細ページからデータを取得するための関数
def parse_maigara_detail_page(row):
    print(
        datetime.now().isoformat(sep=" "),
        row["meigara_id"],
        row["meigara"]
    )
    
    # 連続アクセス時の負荷軽減
    time.sleep(WAIT_TIME)
    
    # クローリング
    response = requests.get(row["detail_url"])
    if not response.status_code == 200:
        raise ValueError("Invalid response")
    response.encoding = 'euc_jp'
    # ゴミとなる文字群を除去
    preprocessed_html_string = response.text.replace("<br>", "\n")
    preprocessed_html_string = preprocessed_html_string.replace("\r", "")
    preprocessed_html_string = preprocessed_html_string.replace("　", " ")
    soup = BeautifulSoup(preprocessed_html_string, "lxml")
    
    # 評価スコアの取得 & 出力
    score_df = parse_scores_table(soup, row["meigara_id"])
    scores_path = MEIGARA_SCORES_DIR + str(row["meigara_id"]) + ".csv"
    score_df.to_csv(
        scores_path,
        encoding="utf-8",
        sep=",",
        index=False,
        quoting=csv.QUOTE_NONNUMERIC
    )
    
    # 評価コメントの取得 & 出力
    comments_df = parse_comments_table(soup, row["meigara_id"])
    comments_path = MEIGARA_COMMENTS_DIR + str(row["meigara_id"]) + ".csv"
    comments_df.to_csv(
        comments_path,
        encoding="utf-8",
        sep=",",
        index=False,
        quoting=csv.QUOTE_NONNUMERIC
    )
    return
```

次にこれらのコードを全ての銘柄に対して実行します。


```python
for idx, row in meigara_master_df.iterrows():
    parse_maigara_detail_page(row)
```

    2017-03-12 17:58:02.529907 1 獺祭
    2017-03-12 17:58:09.126493 2 醸し人九平次
    2017-03-12 17:58:15.797209 3 出羽桜
    2017-03-12 17:58:22.873022 4 田酒
    2017-03-12 17:58:29.302859 5 黒龍
    2017-03-12 17:58:35.890476 6 飛露喜
    2017-03-12 17:58:42.338474 7 新政
    2017-03-12 17:58:49.018110 8 雪の茅舎
    2017-03-12 17:58:55.609204 9 鳳凰美田
    2017-03-12 17:59:01.490036 10 風の森
    2017-03-12 17:59:07.875526 11 鍋島
    2017-03-12 17:59:14.489016 12 くどき上手
    2017-03-12 17:59:20.826484 13 十四代
    2017-03-12 17:59:27.388108 14 菊姫
    2017-03-12 17:59:34.022284 15 天狗舞
    2017-03-12 17:59:40.585017 16 神亀
    2017-03-12 17:59:46.673921 17 浦霞
    2017-03-12 17:59:52.898387 18 鶴齢
    2017-03-12 17:59:59.212296 19 楯野川
    2017-03-12 18:00:05.593475 20 八海山
    2017-03-12 18:00:12.830346 21 雁木
    2017-03-12 18:00:19.296840 22 開運
    2017-03-12 18:00:25.956883 23 大七
    2017-03-12 18:00:32.476801 24 〆張鶴
    2017-03-12 18:00:38.731137 25 久保田
    2017-03-12 18:00:45.247792 26 酔鯨
    2017-03-12 18:00:51.643924 27 手取川
    2017-03-12 18:00:58.161813 28 東一
    2017-03-12 18:01:07.472091 29 陸奥八仙
    2017-03-12 18:01:14.158923 30 写楽（寫樂）
    2017-03-12 18:01:20.226436 31 仙禽
    2017-03-12 18:01:26.551723 32 東洋美人
    2017-03-12 18:01:33.177094 33 花陽浴
    2017-03-12 18:01:39.551707 34 磯自慢
    2017-03-12 18:01:46.262988 35 梵
    2017-03-12 18:01:52.706725 36 悦凱陣
    2017-03-12 18:01:58.956448 37 真澄
    2017-03-12 18:02:05.618642 38 秋鹿
    2017-03-12 18:02:11.940649 39 奥播磨
    2017-03-12 18:02:18.542844 40 王祿
    2017-03-12 18:02:24.676691 41 臥龍梅
    2017-03-12 18:02:31.741734 42 豊盃
    2017-03-12 18:02:38.143491 43 黒牛
    2017-03-12 18:02:44.458652 44 伯楽星
    2017-03-12 18:02:50.960992 45 義侠
    2017-03-12 18:02:57.403678 46 上喜元
    2017-03-12 18:03:03.850065 47 日高見
    2017-03-12 18:03:10.586871 48 玉川（京都府）
    2017-03-12 18:03:16.942453 49 小左衛門
    2017-03-12 18:03:23.533112 50 一ノ蔵
    2017-03-12 18:03:29.888834 51 作
    2017-03-12 18:03:36.620805 52 奈良萬
    2017-03-12 18:03:43.160130 53 屋守
    2017-03-12 18:03:49.590359 54 村祐
    2017-03-12 18:03:55.606912 55 白瀑
    2017-03-12 18:04:02.068721 56 南部美人
    2017-03-12 18:04:08.546669 57 三芳菊
    2017-03-12 18:04:16.465088 58 七田
    2017-03-12 18:04:22.600221 59 雨後の月
    2017-03-12 18:04:28.704886 60 紀土 KID
    2017-03-12 18:04:34.915424 61 亀齢（広島県）
    2017-03-12 18:04:41.046368 62 天明
    2017-03-12 18:04:47.481380 63 満寿泉
    2017-03-12 18:04:53.624541 64 勝駒
    2017-03-12 18:04:59.531139 65 蓬莱泉
    2017-03-12 18:05:05.942901 66 東光
    2017-03-12 18:05:12.123451 67 雅山流
    2017-03-12 18:05:18.514607 68 緑川
    2017-03-12 18:05:24.588177 69 正雪
    2017-03-12 18:05:30.825473 70 まんさくの花
    2017-03-12 18:05:37.373877 71 飛良泉
    2017-03-12 18:05:43.456885 72 七本槍
    2017-03-12 18:05:49.441395 73 澤乃井
    2017-03-12 18:05:55.667049 74 遊穂
    2017-03-12 18:06:01.888673 75 あさ開
    2017-03-12 18:06:08.607785 76 姿
    2017-03-12 18:06:15.474695 77 南
    2017-03-12 18:06:21.992489 78 宝剣
    2017-03-12 18:06:28.424933 79 大那
    2017-03-12 18:06:35.182849 80 越乃寒梅
    2017-03-12 18:06:41.723710 81 春鹿
    2017-03-12 18:06:48.032324 82 常きげん
    2017-03-12 18:06:54.589442 83 五橋
    2017-03-12 18:07:00.893189 84 加賀鳶
    2017-03-12 18:07:07.456926 85 国権
    2017-03-12 18:07:14.117180 86 刈穂
    2017-03-12 18:07:20.776054 87 松の司
    2017-03-12 18:07:27.020833 88 ロ万
    2017-03-12 18:07:33.502105 89 剣菱
    2017-03-12 18:07:39.757611 90 綿屋
    2017-03-12 18:07:45.877950 91 山間
    2017-03-12 18:07:52.074591 92 秀鳳
    2017-03-12 18:07:58.190948 93 越乃景虎
    2017-03-12 18:08:04.403371 94 菊水
    2017-03-12 18:08:10.592417 95 亀泉
    2017-03-12 18:08:17.113787 96 銀嶺立山
    2017-03-12 18:08:23.281406 97 三千盛
    2017-03-12 18:08:30.810961 98 尾瀬の雪どけ
    2017-03-12 18:08:37.117417 99 ひこ孫
    2017-03-12 18:08:43.595980 100 庭のうぐいす
    2017-03-12 18:08:49.925061 101 麒麟山
    2017-03-12 18:08:56.533390 102 来福
    2017-03-12 18:09:04.490698 103 乾坤一
    2017-03-12 18:09:10.272546 104 墨廼江
    2017-03-12 18:09:16.541895 105 石鎚
    2017-03-12 18:09:22.834053 106 初亀
    2017-03-12 18:09:29.015368 107 初孫
    2017-03-12 18:09:35.386027 108 早瀬浦
    2017-03-12 18:09:41.710258 109 羽根屋
    2017-03-12 18:09:47.951677 110 天の戸
    2017-03-12 18:09:54.279195 111 佐久乃花
    2017-03-12 18:10:00.824845 112 幻
    2017-03-12 18:10:06.994982 113 郷乃誉
    2017-03-12 18:10:13.265061 114 諏訪泉
    2017-03-12 18:10:19.696445 115 李白
    2017-03-12 18:10:25.903834 116 酔心
    2017-03-12 18:10:32.087111 117 長珍
    2017-03-12 18:10:38.637781 118 呉春
    2017-03-12 18:10:44.877142 119 一白水成
    2017-03-12 18:10:50.838438 120 不動
    2017-03-12 18:10:56.848904 121 繁桝
    2017-03-12 18:11:03.375572 122 蒼空
    2017-03-12 18:11:09.275653 123 雪中梅
    2017-03-12 18:11:16.715891 124 喜久酔
    2017-03-12 18:11:23.209904 125 泉川
    2017-03-12 18:11:28.979427 126 車坂
    2017-03-12 18:11:35.139811 127 梅乃宿
    2017-03-12 18:11:41.624577 128 いづみ橋
    2017-03-12 18:11:47.942934 129 大信州
    2017-03-12 18:11:54.041809 130 龍力
    2017-03-12 18:12:00.506518 131 嘉美心
    2017-03-12 18:12:07.910173 132 残草蓬莱
    2017-03-12 18:12:14.389312 133 栄光冨士
    2017-03-12 18:12:20.607095 134 貴
    2017-03-12 18:12:26.494997 135 水芭蕉
    2017-03-12 18:12:32.842182 136 萩の鶴
    2017-03-12 18:12:39.052504 137 篠峯
    2017-03-12 18:12:45.484481 138 日置桜
    2017-03-12 18:12:51.802755 139 龍神丸
    2017-03-12 18:12:58.025424 140 北雪
    2017-03-12 18:13:04.397964 141 鷹勇
    2017-03-12 18:13:10.444564 142 上善如水
    2017-03-12 18:13:16.541637 143 たかちよ
    2017-03-12 18:13:22.887476 144 宗玄
    2017-03-12 18:13:30.389642 145 小鼓
    2017-03-12 18:13:37.043704 146 鯉川
    2017-03-12 18:13:43.258559 147 不老泉
    2017-03-12 18:13:49.701382 148 磐城壽
    2017-03-12 18:13:55.527370 149 群馬泉
    2017-03-12 18:14:01.875800 150 龍勢
    2017-03-12 18:14:08.031016 151 鏡山
    2017-03-12 18:14:14.378424 152 花垣
    2017-03-12 18:14:20.753708 153 白岳仙
    2017-03-12 18:14:27.017937 154 笑四季
    2017-03-12 18:14:33.638816 155 萩乃露
    2017-03-12 18:14:39.690084 156 醴泉
    2017-03-12 18:14:46.411015 157 花泉
    2017-03-12 18:14:52.805909 158 大山
    2017-03-12 18:14:59.338377 159 鳴海
    2017-03-12 18:15:07.233578 160 帰山
    2017-03-12 18:15:13.895577 161 賀茂金秀
    2017-03-12 18:15:20.145517 162 奥の松
    2017-03-12 18:15:26.563100 163 宮寒梅
    2017-03-12 18:15:32.422080 164 るみ子の酒
    2017-03-12 18:15:38.670114 165 甲子
    2017-03-12 18:15:44.928295 166 誠鏡
    2017-03-12 18:15:51.241363 167 旭日
    2017-03-12 18:15:57.226133 168 あぶくま
    2017-03-12 18:16:03.706618 169 町田酒造
    2017-03-12 18:16:10.099683 170 春霞
    2017-03-12 18:16:16.087101 171 賀儀屋
    2017-03-12 18:16:22.474515 172 川鶴
    2017-03-12 18:16:28.903583 173 明鏡止水
    2017-03-12 18:16:35.099438 174 洗心
    2017-03-12 18:16:41.198932 175 出雲富士
    2017-03-12 18:16:47.213342 176 会津娘
    2017-03-12 18:16:52.938671 177 住吉
    2017-03-12 18:16:59.059578 178 天吹
    2017-03-12 18:17:06.178848 179 会津中将
    2017-03-12 18:17:12.259170 180 豊賀
    2017-03-12 18:17:18.051022 181 美丈夫
    2017-03-12 18:17:24.358635 182 菊正宗
    2017-03-12 18:17:30.542959 183 富久長
    2017-03-12 18:17:37.288783 184 れいざん
    2017-03-12 18:17:43.221305 185 一本義
    2017-03-12 18:17:49.443888 186 竹鶴
    2017-03-12 18:17:55.976943 187 丹沢山
    2017-03-12 18:18:02.464572 188 奥
    2017-03-12 18:18:08.285727 189 十九
    2017-03-12 18:18:14.013907 190 長陽福娘
    2017-03-12 18:18:20.125712 191 白龍（福井県）
    2017-03-12 18:18:26.727557 192 美寿々
    2017-03-12 18:18:32.445372 193 三連星
    2017-03-12 18:18:38.574551 194 天寿
    2017-03-12 18:18:45.343555 195 天覧山
    2017-03-12 18:18:51.417105 196 酔右衛門
    2017-03-12 18:18:57.799544 197 大倉
    2017-03-12 18:19:04.069903 198 高千代
    2017-03-12 18:19:10.391022 199 相模灘
    2017-03-12 18:19:16.763779 200 七賢


ここまでで、銘柄のマスタデータ、詳細ページの評価、詳細ページのコメントの情報が手に入りました。

## 6. テキスト解析

　ここからは先程クローラーで収集したデータを利用して、TFIDFによるレビュー中の特徴的な形容詞の抽出と単語ベースのクラスタリングを行っていきます。

### 6.1 TFIDF によるレビュー中の特徴的な形容詞の抽出

　この解析では、あるドキュメント中における特徴的な単語（特徴語）の抽出を行います。集めたデータを各種統計処理で扱えるようにするためには、行列形式に変換する必要があります。今回は Bag-of-Words モデルを用いて、単語を行列の形に変換します。Bag-of-Words とは、文章に単語が含まれているかどうかのみを考え、単語の並び方などは考慮しないモデルのことです。一番シンプルなモデルは単語があれば 1、なければ 0 となります。また、単語の出現回数をそのまま使う (Term Frequency) という方法もあります。これは文書中にある単語が含まれている回数をそのまま値として用います。

```
すもももももももものうち  (1)
↓
[すもも, も, もも, も, もも, の, うち]  (2)
↓
{すもも: 1, も:2, もも: 2, の: 1, うち:1}  (3)
```

　そして各ドキュメントに含まれる単語を列に、文書を行とすると単語の出現回数を要素とした行列形式に変換できます。例として、 (a) 「すもももももももものうち」、(b) 「料理も景色もすばらしい」、(c) 「私の趣味は写真撮影です」という3つの文書を考えます。列のラベルは単語の出現の早い順に [すもも, も, もも, の, うち, 料理, 景色, 素晴らしい, 私, 趣味, は, 写真撮影, です] とすると、文書行列は下記のようになります。

```
[[1,2,2,1,1,0,0,0,0,0,0,0,0],    #  (a)
 [0,2,0,0,0,1,1,1,0,0,0,0,0],    #  (b)
 [0,0,0,1,0,0,0,0,1,1,1,1,1]]    #  (c)
```
 
　今回は数値に Term Frequency を用います。この変換を行うためには、元のテキストデータを単語単位に分割する必要があります。これを行うためには形態素解析ツールのMeCabを利用します。
 
　次に特徴量の計算には TFIDF を用います。TFIDF は TF と IDF というの2つの値を掛けあわせた指標のことです。TF は文書内における単語の出現頻度を表します。これは「ある文書中である単語が何回出現したか」で定義されます。1つの文書に多く出現する単語ほど重要度が高くなります。IDF は多数の文書に出現する単語ほど重要度が低くなるようなスコアです。「ある単語が含まれている文書数を全ての文書数で割ったものの逆数」で定義されます。つまり、TFIDF が大きな値になるということは、「文書内である特定の単語が多く出現し、かつその単語は他の文書ではほとんど出現しない」ということを表します。例えば、「私」という単語は、各文書内における出現回数は多いですが、多くの文書に出現するので重要度は下がります。逆に「特許」という単語は、「特許」を話題の中心にしている特定の文書には文書内には多く現れ、一般的な文書には現れない単語なので重要度は上がります。

　これらの解析を行うためのコードを見ていきましょう。まずは単語の分割を行うための関数を定義します。ここでは対象の品詞のみに絞り込んで、原形のみを抽出するようにしています。


```python
def split_text(text, target_pos=["形容詞"]):
    tagger = MeCab.Tagger()
    text_str = text
    tagger.parse('')
    node = tagger.parseToNode(text_str)

    words = []
    while node:
        l = node.feature.split(",")
        pos = l[0]
        if pos in target_pos:
            # unicode 型に戻す
            if l[6] == "*":
                word = node.surface # 変化しない語は表層形をそのまま使う
            else:
                word = l[6]         # 動詞や形容詞は原形を使う
            words.append(word)
        node = node.next
    return " ".join(words)          # スペース区切りで単語を結合し返す
```

　では実際に全銘柄のコメントを読み込んで、単語単位に分割し必要な単語のみを取り出してみましょう。


```python
# 全ファイル読み込み
comment_files = glob.glob("../data/meigara_comments/*.csv")

# 縦方向に単純結合
comment_df = pd.concat([pd.read_csv(f) for f in comment_files])
comment_df = comment_df.reset_index()

# 全ての text に対して形容詞の抽出を行う
comment_df["split"] = comment_df["text"].map(split_text)
comment_df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>index</th>
      <th>meigara_id</th>
      <th>toukou_id</th>
      <th>title</th>
      <th>created_at</th>
      <th>user_name</th>
      <th>text</th>
      <th>split</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>1</td>
      <td>6193</td>
      <td>すっきりして飲みやすい</td>
      <td>2016-10-20 11:57:54</td>
      <td>あいうそん</td>
      <td>おいしいです</td>
      <td>おいしい</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>1</td>
      <td>6126</td>
      <td>獺祭 等外２３</td>
      <td>2016-08-08 22:20:28</td>
      <td>富牟谷欠</td>
      <td>獺祭 等外２３ 山田錦２３ 生酒 ２７ＢＹ ライチ様な立ち香、含み香。抜ける香りはやや甘く。...</td>
      <td>甘い 強い 淡い ない 濃い</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>1</td>
      <td>5992</td>
      <td>獺祭50</td>
      <td>2016-04-26 19:24:02</td>
      <td>季がらし</td>
      <td>獺祭と言えば高精米、磨きが強調され、50%はその最低ランクである しかし全国の銘酒蔵もこの5...</td>
      <td>美味い 美味い</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>1</td>
      <td>5946</td>
      <td>獺祭等外</td>
      <td>2016-03-18 20:37:29</td>
      <td>季がらし</td>
      <td>旭酒造蔵本の売店で買った普通酒！ 山田錦は栽培時、5%以上の等外米(規格外)が出てしまい純米...</td>
      <td>美味しい 悪い</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>1</td>
      <td>5940</td>
      <td>スパークリング、うすにごり</td>
      <td>2016-03-13 23:16:16</td>
      <td>nomuyoshi</td>
      <td>抜栓直後、瓶から上る香りは若々しく青いような香り。 上る香りはさっぱりとした果物のよう。 口...</td>
      <td>若々しい 青い 鋭い</td>
    </tr>
  </tbody>
</table>
</div>



銘柄別に全単語を結合します。


```python
meigara_comments_df = comment_df\
    .groupby("meigara_id")["split"]\
    .apply(lambda x: "%s" % ' '.join(x))\
    .reset_index()
meigara_comments_df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>meigara_id</th>
      <th>split</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>おいしい 甘い 強い 淡い ない 濃い 美味い 美味い 美味しい 悪い 若々しい 青い 鋭い...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>ない ない 美味い 美味い ない 早い 早い 若い 美味い 高い たまらない いい 甘い ...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>甘い 柔らかい 無い 美味い イイ やすい 力強い 鋭い 美味い 甘い くどい 旨い 甘い ...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>いい 若い 深い 無い 新しい 甘い 濃い 旨い 美味い 高い 物足りない  甘酸っぱい 良...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>軽い 良い ほしい 強い うまい 不味い 苦い 良い 良い 広い  美味しい ない 美味しい...</td>
    </tr>
  </tbody>
</table>
</div>



　今回は数値に Term Frequency を用います。scikit-learn に [CountVectorizer](http://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.CountVectorizer.html)というものがあり、単語と列番号の対応付けなどの作業をまとめて行うことが出来ます。この次に TFIDF の計算を行う場合、さらに簡易化したライブラリが存在します。
 
　TFIDF の計算は scikit-learn の [TfidfVectorizer](http://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html) を用いれば、前述の CountVectorizer による行列化と TFIDF の計算を同時に行うことが出来るので、今回はこれを用いて TFIDF の計算を行います。では実際に TfidfVectorizer を利用して、TFIDF の計算を行ってみましょう。


```python
vectorizer = TfidfVectorizer()
tfidfs = vectorizer.fit_transform(meigara_comments_df["split"])
tfidfs
```




    <200x292 sparse matrix of type '<class 'numpy.float64'>'
    	with 4099 stored elements in Compressed Sparse Row format>



では計算結果から、各銘柄におけるスコアの上位から5単語ずつ取り出してみましょう。


```python
## TFIDF の結果からi 番目のドキュメントの特徴的な上位 n 語を取り出す
def extract_feature_words(terms, tfidfs, i, n):
    tfidf_array = tfidfs[i]
    top_n_idx = tfidf_array.argsort()[-n:][::-1]
    words = [terms[idx] for idx in top_n_idx]
    return words
```


```python
# index 順の単語のリスト
terms = vectorizer.get_feature_names()
```


```python
top_n = 5  # 上位5件
highscore_words = [
    extract_feature_words(terms, tfidfs.toarray(), i, top_n)
    for i
    in range(len(meigara_comments_df.index))
]
highscore_words_str = [" ".join(l) for l in highscore_words]
meigara_comments_df["highscore_words"] = highscore_words_str
meigara_comments_df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>meigara_id</th>
      <th>split</th>
      <th>highscore_words</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>おいしい 甘い 強い 淡い ない 濃い 美味い 美味い 美味しい 悪い 若々しい 青い 鋭い...</td>
      <td>良い 美味しい 悪い ない うまい</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>ない ない 美味い 美味い ない 早い 早い 若い 美味い 高い たまらない いい 甘い ...</td>
      <td>美味い 不味い ない 甘い 旨い</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>甘い 柔らかい 無い 美味い イイ やすい 力強い 鋭い 美味い 甘い くどい 旨い 甘い ...</td>
      <td>素晴らしい 甘い やすい イイ 美味しい</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>いい 若い 深い 無い 新しい 甘い 濃い 旨い 美味い 高い 物足りない  甘酸っぱい 良...</td>
      <td>良い 旨い 無い うまい ない</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>軽い 良い ほしい 強い うまい 不味い 苦い 良い 良い 広い  美味しい ない 美味しい...</td>
      <td>良い うまい おいしい ない いい</td>
    </tr>
  </tbody>
</table>
</div>



銘柄マスタと結合して、結果を確認してみましょう。


```python
# マスタデータの JOIN
master_df = pd.read_csv(MEIGARA_MASTER_PATH)
result_df = master_df.merge(meigara_comments_df, on="meigara_id", how="inner")
result_df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>meigara_id</th>
      <th>rank</th>
      <th>meigara</th>
      <th>yomi</th>
      <th>kuramoto</th>
      <th>prefecture</th>
      <th>city</th>
      <th>detail_url</th>
      <th>split</th>
      <th>highscore_words</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>1</td>
      <td>獺祭</td>
      <td>だっさい</td>
      <td>旭酒造（山口県）</td>
      <td>山口県</td>
      <td>岩国市</td>
      <td>http://www.sakeno.com/meigara/931</td>
      <td>おいしい 甘い 強い 淡い ない 濃い 美味い 美味い 美味しい 悪い 若々しい 青い 鋭い...</td>
      <td>良い 美味しい 悪い ない うまい</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>2</td>
      <td>醸し人九平次</td>
      <td>かもしびとくへいじ</td>
      <td>萬乗醸造</td>
      <td>愛知県</td>
      <td>名古屋市</td>
      <td>http://www.sakeno.com/meigara/735</td>
      <td>ない ない 美味い 美味い ない 早い 早い 若い 美味い 高い たまらない いい 甘い ...</td>
      <td>美味い 不味い ない 甘い 旨い</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>3</td>
      <td>出羽桜</td>
      <td>でわざくら</td>
      <td>出羽桜酒造</td>
      <td>山形県</td>
      <td>天童市</td>
      <td>http://www.sakeno.com/meigara/219</td>
      <td>甘い 柔らかい 無い 美味い イイ やすい 力強い 鋭い 美味い 甘い くどい 旨い 甘い ...</td>
      <td>素晴らしい 甘い やすい イイ 美味しい</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>4</td>
      <td>田酒</td>
      <td>でんしゅ</td>
      <td>西田酒造店</td>
      <td>青森県</td>
      <td>青森市</td>
      <td>http://www.sakeno.com/meigara/11</td>
      <td>いい 若い 深い 無い 新しい 甘い 濃い 旨い 美味い 高い 物足りない  甘酸っぱい 良...</td>
      <td>良い 旨い 無い うまい ない</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>5</td>
      <td>黒龍</td>
      <td>こくりゅう</td>
      <td>黒龍酒造</td>
      <td>福井県</td>
      <td>吉田郡</td>
      <td>http://www.sakeno.com/meigara/667</td>
      <td>軽い 良い ほしい 強い うまい 不味い 苦い 良い 良い 広い  美味しい ない 美味しい...</td>
      <td>良い うまい おいしい ない いい</td>
    </tr>
  </tbody>
</table>
</div>



最後に結果をCSVファイルとして出力します。


```python
target_cols = ["rank", "meigara", "kuramoto", "detail_url", "highscore_words"]
result_df[target_cols].to_csv(TFIDF_PATH, index=False)
```

ここまでが特徴語抽出を行うまでの一連の流れとなります。

### 6.2 単語ベースのクラスタリング

　次は評価が似た銘柄同士をまとめる方法を見ていきたいと思います。サンプルデータが大量にある場合、似た者同士をまとめることで、新たな知見が得られる可能性があります。ここではその一連の流れを見ていきます。

　まず最初行わなければならないのは、特徴語の抽出の場合と同じく各文書に含まれる単語を行列形式に変換することです。今回も Term Frequency を用いた Bag-of-Words モデルで変換します。
 
　次にクラスタリングを行う前に、行列に対していくつか前処理を行う必要があります。まずレビューの件数が大きく異なるので、数値を標準化してやる必要があります。今回は「Zスコア」という標準化を行います。これは各要素から平均を引いて、標準偏差で割ったものです。この変換を行うと、平均が 0 で標準偏差・分散が 1 になります。この変換を行うためのライブラリとして scikit-learn には StandardScaler があります。また、行列は疎な状態となっています。このような場合は次元圧縮を行うことで、より効率が良く、直感に近いクラスタリング結果を得られます。今回は[主成分分析：PCA](http://scikit-learn.org/stable/modules/generated/sklearn.decomposition.PCA.html)を利用して次元を圧縮しています。
 
　最後にクラスタリングを行います。今回はコサイン類似度を距離の基準とした階層型クラスタリングを行いクラスタを決定しています。
 

　では実際のコードを確認していきましょう。まずは対象の単語の抽出です。今回は名詞、動詞、形容詞を抽出しています。
 


```python
noun_verb_adj_words = comment_df["text"]\
    .apply(lambda x: split_text(x, target_pos=["名詞", "動詞", "形容詞"]))
comment_df["split_noun_verb_adj"] = noun_verb_adj_words
meigara_comments_df = comment_df\
    .groupby("meigara_id")["split_noun_verb_adj"]\
    .apply(lambda x: "%s" % ' '.join(x))\
    .reset_index()

meigara_comments_df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>meigara_id</th>
      <th>split_noun_verb_adj</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>おいしい 獺 祭 等外 ２ ３ 山田 錦 ２ ３ 生酒 ２ ７ ＢＹ ライチ 様 立ち 香 ...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>Wow very oishii Sake ! 米 吟醸 aka 醸す 人 九 平次 彼 地 ...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>上 立つ 仄か 甘い 香り 含み 柔らかい 入る 派手 さ 無い 純大 吟 旨み 特徴 個 ...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>好き 酒 一つ する コク ある いい 感じ 田 酒 特 純生 飲む 開 栓 直後 含む す...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>甘み 感じる する 辛口 ｡ 後味 軽い 酸味 ある ｡ ラベル 飲む 方 ｢ 冷やす ｣ ...</td>
    </tr>
  </tbody>
</table>
</div>



次に行列への変換です。前述の CountVectorizer を利用することで、簡単に変換できます。


```python
vectorizer = CountVectorizer()
word_counts = vectorizer.fit_transform(meigara_comments_df.split_noun_verb_adj)
wca = word_counts.toarray()
wca
```




    array([[0, 0, 0, ..., 0, 0, 1],
           [0, 0, 0, ..., 0, 1, 0],
           [0, 0, 0, ..., 0, 0, 0],
           ..., 
           [0, 0, 0, ..., 0, 0, 0],
           [0, 0, 0, ..., 0, 0, 0],
           [0, 0, 0, ..., 0, 0, 0]], dtype=int64)



StandardScaler を利用することで標準化も簡単に行えます（int 型から float 型への変換は意図通りなので警告内容については問題ない）。


```python
sds = StandardScaler()
X = sds.fit_transform(wca)
```

    /Users/tojima/anaconda3/lib/python3.5/site-packages/sklearn/utils/validation.py:429: DataConversionWarning: Data with input dtype int64 was converted to float64 by StandardScaler.
      warnings.warn(msg, _DataConversionWarning)


次は次元圧縮を行います。今回は30次元に圧縮しています。この圧縮された行列の各行は、それぞれの文書が表す概念を表現しているものとなり、概念ベクトルと呼べるものとなります。


```python
model = PCA(n_components=30)
X_decomp = model.fit_transform(X)
X_decomp
```




    array([[  2.28167681e+02,  -1.16255034e+02,  -4.89383813e+01, ...,
              9.14023707e-02,  -8.60942515e-01,  -6.03637929e-01],
           [  1.12847840e+02,   1.94123781e+02,  -6.26221348e+01, ...,
              3.46364954e-01,  -1.24565097e+00,  -6.24311590e-01],
           [  2.25193159e+01,   6.46982151e+00,   1.69020459e+01, ...,
              1.00232518e+00,   6.03836210e-02,  -5.87677278e-02],
           ..., 
           [ -8.47630942e+00,  -2.13891471e+00,  -4.14508268e+00, ...,
             -8.44277795e-02,  -4.82077079e-01,  -4.45857641e-01],
           [ -4.62522339e+00,  -3.79801541e-01,  -3.06852519e+00, ...,
              4.29486114e-01,  -1.49074427e+00,   2.41093999e-01],
           [ -6.23976436e+00,  -1.28528158e+00,  -2.87711006e+00, ...,
              1.53063625e-01,   9.22628033e-02,  -9.44972731e-01]])



さて、ここまでで下準備が終わったのでクラスタリングを実行しましょう。今回はコサイン類似度を基準とした6つのクラスタに分けています。


```python
model = AgglomerativeClustering(n_clusters=6, linkage="average", affinity="cosine")
y = model.fit_predict(X_decomp)
y
```




    array([2, 2, 2, 2, 2, 5, 0, 2, 0, 0, 0, 2, 2, 2, 2, 2, 0, 0, 0, 1, 3, 0, 0,
           1, 2, 3, 3, 3, 3, 0, 0, 0, 3, 0, 2, 0, 2, 1, 3, 3, 2, 1, 3, 3, 3, 0,
           0, 3, 3, 3, 3, 3, 3, 3, 1, 2, 0, 3, 3, 3, 3, 0, 0, 3, 2, 3, 3, 3, 3,
           3, 3, 0, 0, 3, 2, 3, 3, 0, 3, 2, 3, 3, 3, 3, 3, 0, 4, 3, 2, 3, 3, 3,
           3, 2, 3, 3, 3, 3, 3, 3, 3, 3, 0, 3, 3, 3, 2, 3, 3, 3, 3, 3, 0, 3, 3,
           3, 3, 3, 0, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 0, 3, 3, 3, 0, 3, 3, 3, 3,
           3, 3, 3, 1, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 0, 3, 3, 3, 3, 3, 3, 3,
           2, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 2,
           3, 3, 3, 3, 3, 3, 3, 1, 3, 3, 3, 3, 3, 3, 3, 3])



クラスタ番号が算出できたので、銘柄マスタデータと結合し結果を確認してみましょう。


```python
# マスタデータの JOIN
master_df = pd.read_csv(MEIGARA_MASTER_PATH)
result_df = master_df.merge(meigara_comments_df, on="meigara_id", how="inner")

result_df["cluster"] = y
result_df.sort_values(by=["cluster", "rank"])
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>meigara_id</th>
      <th>rank</th>
      <th>meigara</th>
      <th>yomi</th>
      <th>kuramoto</th>
      <th>prefecture</th>
      <th>city</th>
      <th>detail_url</th>
      <th>split_noun_verb_adj</th>
      <th>cluster</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>6</th>
      <td>7</td>
      <td>7</td>
      <td>新政</td>
      <td>あらまさ</td>
      <td>新政酒造</td>
      <td>秋田県</td>
      <td>秋田市</td>
      <td>http://www.sakeno.com/meigara/55</td>
      <td>冷蔵 する いる 気 つける 開 栓 する プシュー 吹き出す しまう 半年 寝かせる いる...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>9</td>
      <td>7</td>
      <td>鳳凰美田</td>
      <td>ほうおうびでん</td>
      <td>小林酒造（栃木県）</td>
      <td>栃木県</td>
      <td>小山市</td>
      <td>http://www.sakeno.com/meigara/95</td>
      <td>鳳凰 美田 濾過 本 生 ２ ０ １ ７ / ２ 頂く アル 添 フルーティー 香り ラベル...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>10</td>
      <td>10</td>
      <td>風の森</td>
      <td>かぜのもり</td>
      <td>油長酒造</td>
      <td>奈良県</td>
      <td>御所市</td>
      <td>http://www.sakeno.com/meigara/898</td>
      <td>風 森 ALPHA TYPE 3 米 吟醸 八 錦 ５ ０ ２ ７ ＢＹ 軽い 吟醸 香 含...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>11</td>
      <td>10</td>
      <td>鍋島</td>
      <td>なべしま</td>
      <td>富久千代酒造</td>
      <td>佐賀県</td>
      <td>鹿島市</td>
      <td>http://www.sakeno.com/meigara/1482</td>
      <td>春先 購入 する 半年 寝かせる もの タッチ チリ する 白 ワイン 的 ブドウ シャープ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>16</th>
      <td>17</td>
      <td>17</td>
      <td>浦霞</td>
      <td>うらかすみ</td>
      <td>佐浦</td>
      <td>宮城県</td>
      <td>塩竈市</td>
      <td>http://www.sakeno.com/meigara/47</td>
      <td>素っ気 ラベル 純 米 事 原酒 事 わかる メチャクチャ ぶっきらぼう 強面 塩釜 漁師 ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>17</th>
      <td>18</td>
      <td>18</td>
      <td>鶴齢</td>
      <td>かくれい</td>
      <td>青木酒造（新潟県）</td>
      <td>新潟県</td>
      <td>南魚沼市</td>
      <td>http://www.sakeno.com/meigara/583</td>
      <td>鶴 齢 特別 米 越 淡い 麗 ５ ５ ％ 濾過 生 原酒 ２ ８ ＢＹ 綺麗 果実 香 含...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>18</th>
      <td>19</td>
      <td>18</td>
      <td>楯野川</td>
      <td>たてのかわ</td>
      <td>楯の川酒造</td>
      <td>山形県</td>
      <td>酒田市</td>
      <td>http://www.sakeno.com/meigara/229</td>
      <td>楯 野川 純 米 吟醸 山田 ５ ０ 汲む 夏 熟 ２ ７ ＢＹ 好い 熟れる 甘い 果実 ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>21</th>
      <td>22</td>
      <td>21</td>
      <td>開運</td>
      <td>かいうん</td>
      <td>土井酒造場</td>
      <td>静岡県</td>
      <td>掛川市</td>
      <td>http://www.sakeno.com/meigara/729</td>
      <td>酒屋 特 本 思う 特 純 特 純 包み 紙 オレンジ ん 思い返す 飲 購入 する みる ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>22</th>
      <td>23</td>
      <td>21</td>
      <td>大七</td>
      <td>だいしち</td>
      <td>大七酒造</td>
      <td>福島県</td>
      <td>二本松市</td>
      <td>http://www.sakeno.com/meigara/381</td>
      <td>全体 的 濃い 醇 いう もと 生まれる コシ 力強い さ ある いい ん じん わり 感じ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>29</th>
      <td>30</td>
      <td>30</td>
      <td>写楽（寫樂）</td>
      <td>しゃらく</td>
      <td>宮泉銘醸</td>
      <td>福島県</td>
      <td>会津若松市</td>
      <td>http://www.sakeno.com/meigara/1804</td>
      <td>一 回 火入れ 米 酒 バランス いい 言う の 特徴 感じる られる の 甘い さ 旨み ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>30</th>
      <td>31</td>
      <td>30</td>
      <td>仙禽</td>
      <td>せんきん</td>
      <td>株式会社せんきん</td>
      <td>栃木県</td>
      <td>さくら市</td>
      <td>http://www.sakeno.com/meigara/457</td>
      <td>米 栃木 県 さくら 市 山田 錦 仕込 水 地域 地下 水 使用 ドメーヌ 化 する 米 ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>31</th>
      <td>32</td>
      <td>32</td>
      <td>東洋美人</td>
      <td>とうようびじん</td>
      <td>澄川酒造場</td>
      <td>山口県</td>
      <td>萩市</td>
      <td>http://www.sakeno.com/meigara/1094</td>
      <td>シリーズ 使用 米 ラベル 記載 する れる いる 筈 ん コレ 米 品種 名 書く れる ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>33</th>
      <td>34</td>
      <td>32</td>
      <td>磯自慢</td>
      <td>いそじまん</td>
      <td>磯自慢酒造</td>
      <td>静岡県</td>
      <td>焼津市</td>
      <td>http://www.sakeno.com/meigara/725</td>
      <td>磯 自慢 特別 醸造 山田 ５ ５ ６ ０ ２ ７ ＢＹ メロン 様 優しい 清楚 香り 柔...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>35</th>
      <td>36</td>
      <td>36</td>
      <td>悦凱陣</td>
      <td>よろこびがいじん</td>
      <td>丸尾本店</td>
      <td>香川県</td>
      <td>仲多度郡</td>
      <td>http://www.sakeno.com/meigara/325</td>
      <td>悦 凱陣 山 廃 米 ろ過 生 赤磐 雄 町 ６ ８ ２ ６ ＢＹ 微か 酸 感じる させる...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>45</th>
      <td>46</td>
      <td>45</td>
      <td>上喜元</td>
      <td>じょうきげん</td>
      <td>酒田酒造</td>
      <td>山形県</td>
      <td>酒田市</td>
      <td>http://www.sakeno.com/meigara/224</td>
      <td>上 喜 米 吟醸 仕込 五 五 号 濾過 生 原酒 杜氏 佐藤 正一 渾身 山田 白玉 ５ ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>46</th>
      <td>47</td>
      <td>47</td>
      <td>日高見</td>
      <td>ひたかみ</td>
      <td>平孝酒造</td>
      <td>宮城県</td>
      <td>石巻市</td>
      <td>http://www.sakeno.com/meigara/184</td>
      <td>うすい にごる フレッシュ テクスチャー 林檎 的 香味 マッチング 素晴らしい 生原 酒 ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>56</th>
      <td>57</td>
      <td>57</td>
      <td>三芳菊</td>
      <td>みよしきく</td>
      <td>三芳菊酒造</td>
      <td>徳島県</td>
      <td>三好市</td>
      <td>http://www.sakeno.com/meigara/1456</td>
      <td>グラス 注ぐ 瞬間 立ちあがる 華やか 芳香 含む 甘酸っぱい パイナップル 系 甘味 口 ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>61</th>
      <td>62</td>
      <td>61</td>
      <td>天明</td>
      <td>てんめい</td>
      <td>曙酒造</td>
      <td>福島県</td>
      <td>河沼郡</td>
      <td>http://www.sakeno.com/meigara/404</td>
      <td>天明 通常 飲む こと ない 本来 ノーマル 飲む 比較 する タイプ の 情報 なる 思う...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>62</th>
      <td>63</td>
      <td>61</td>
      <td>満寿泉</td>
      <td>ますいずみ</td>
      <td>桝田酒造店</td>
      <td>富山県</td>
      <td>富山市</td>
      <td>http://www.sakeno.com/meigara/621</td>
      <td>米 酒 生酒 生酒 言う フレッシュ 甘い イメージ ある の 違う 印象 若干 香り 高い...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>71</th>
      <td>72</td>
      <td>70</td>
      <td>七本槍</td>
      <td>しちほんやり</td>
      <td>冨田酒造</td>
      <td>滋賀県</td>
      <td>長浜市</td>
      <td>http://www.sakeno.com/meigara/1211</td>
      <td>何 気 飲む スペック 飲む 瞬間 器 大きい さ 感じる 精白 ら する さ イメージ す...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>72</th>
      <td>73</td>
      <td>70</td>
      <td>澤乃井</td>
      <td>さわのい</td>
      <td>小澤酒造</td>
      <td>東京都</td>
      <td>青梅市</td>
      <td>http://www.sakeno.com/meigara/112</td>
      <td>淡い 麗 辛口 飲む 口 いい クセ ない 素直 味 旨み する いる 呑む やすい コスト...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>77</th>
      <td>78</td>
      <td>77</td>
      <td>宝剣</td>
      <td>ほうけん</td>
      <td>宝剣酒造</td>
      <td>広島県</td>
      <td>呉市</td>
      <td>http://www.sakeno.com/meigara/1020</td>
      <td>含む 意外 熟れる 果実 香 ムン 感じる 生 フレッシュ さ イチゴ マスカット 様 甘味...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>85</th>
      <td>86</td>
      <td>77</td>
      <td>刈穂</td>
      <td>かりほ</td>
      <td>刈穂酒造</td>
      <td>秋田県</td>
      <td>大仙市</td>
      <td>http://www.sakeno.com/meigara/198</td>
      <td>含む フレッシュ 直球 的 香味 感じる 日ハム 大谷 投手 オーラ ある 味わい バスンバ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>102</th>
      <td>103</td>
      <td>101</td>
      <td>乾坤一</td>
      <td>けんこんいち</td>
      <td>大沼酒造店</td>
      <td>宮城県</td>
      <td>柴田郡</td>
      <td>http://www.sakeno.com/meigara/1153</td>
      <td>含む ハチミツ よう する 甘味 口 中 フワ ッ 波打つ よう 広がる いく 酸 弱める ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>112</th>
      <td>113</td>
      <td>113</td>
      <td>郷乃誉</td>
      <td>さとのほまれ</td>
      <td>須藤本家（茨城県）</td>
      <td>茨城県</td>
      <td>笠間市</td>
      <td>http://www.sakeno.com/meigara/1081</td>
      <td>香り 控え目 特有 フレッシュ 甘味 ある する 口当たり ドライテイスト 余韻 する 酸 ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>118</th>
      <td>119</td>
      <td>113</td>
      <td>一白水成</td>
      <td>いっぱくすいせい</td>
      <td>福禄寿酒造</td>
      <td>秋田県</td>
      <td>南秋田郡</td>
      <td>http://www.sakeno.com/meigara/1753</td>
      <td>目 とまる 一 白水 成 四 合 瓶 見る ラベル 左上 plus +」 ある 店員 さん ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>129</th>
      <td>130</td>
      <td>128</td>
      <td>龍力</td>
      <td>たつりき</td>
      <td>本田商店</td>
      <td>兵庫県</td>
      <td>姫路市</td>
      <td>http://www.sakeno.com/meigara/871</td>
      <td>ひる 燻蒸 香 太い すぎる よい うまい さ 燗 する 燻蒸 クセ 和らぐ よう メロン ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>133</th>
      <td>134</td>
      <td>128</td>
      <td>貴</td>
      <td>たか</td>
      <td>永山本家酒造場</td>
      <td>山口県</td>
      <td>宇部市</td>
      <td>http://www.sakeno.com/meigara/1085</td>
      <td>オリ 甘い さ 中 バナナ 感じる 甘味 中盤 活性 にごる 来る 炭酸 押し寄せる 強い ...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>153</th>
      <td>154</td>
      <td>144</td>
      <td>笑四季</td>
      <td>えみしき</td>
      <td>笑四季酒造</td>
      <td>滋賀県</td>
      <td>甲賀市</td>
      <td>http://www.sakeno.com/meigara/784</td>
      <td>笑 四季 特別 米 Polynesia 生 アル 原酒 ２ ７ ＢＹ 熟れる 果実 濃厚 甘...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>19</th>
      <td>20</td>
      <td>20</td>
      <td>八海山</td>
      <td>はっかいさん</td>
      <td>八海醸造</td>
      <td>新潟県</td>
      <td>南魚沼市</td>
      <td>http://www.sakeno.com/meigara/593</td>
      <td>八海山 言う こと 味 薄い そう 思う いる 米 原酒 ｡ する 旨み コク ある これ ...</td>
      <td>1</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>170</th>
      <td>171</td>
      <td>168</td>
      <td>賀儀屋</td>
      <td>かぎや</td>
      <td>成龍酒造</td>
      <td>愛媛県</td>
      <td>西条市</td>
      <td>http://www.sakeno.com/meigara/1536</td>
      <td>伊予 賀 儀 屋 純大 しずく 媛 ４ ５ 生原 ２ ６ ＢＹ 控えめ 香り 甘い 穏やか ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>171</th>
      <td>172</td>
      <td>168</td>
      <td>川鶴</td>
      <td>かわつる</td>
      <td>川鶴酒造</td>
      <td>香川県</td>
      <td>観音寺市</td>
      <td>http://www.sakeno.com/meigara/337</td>
      <td>川鶴 特別 米 Wisdom 限定 生 原酒 山田 / オオセト ･ ５ ０ ５ ５ ２ ８...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>172</th>
      <td>173</td>
      <td>168</td>
      <td>明鏡止水</td>
      <td>めいきょうしすい</td>
      <td>大澤酒造</td>
      <td>長野県</td>
      <td>佐久市</td>
      <td>http://www.sakeno.com/meigara/525</td>
      <td>これ 当たり 旨味 濃い フルーティー 色 黄み かう 1 日 目 尖る 3 日 目 マイル...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>173</th>
      <td>174</td>
      <td>168</td>
      <td>洗心</td>
      <td>せんしん</td>
      <td>朝日酒造（新潟県）</td>
      <td>新潟県</td>
      <td>長岡市</td>
      <td>http://www.sakeno.com/meigara/1171</td>
      <td>私 １ ２ 年 前 プロデュース する 下田 富士 米 吟醸 富士 錦 洗心 飲む 比べる ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>174</th>
      <td>175</td>
      <td>168</td>
      <td>出雲富士</td>
      <td>いずもふじ</td>
      <td>富士酒造</td>
      <td>島根県</td>
      <td>出雲市</td>
      <td>http://www.sakeno.com/meigara/971</td>
      <td>出雲 市 契約 農家 さん 特別 栽培 山田 錦 100 % 使用 35 % 贅沢 磨く 上...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>175</th>
      <td>176</td>
      <td>168</td>
      <td>会津娘</td>
      <td>あいづむすめ</td>
      <td>高橋庄作酒造店</td>
      <td>福島県</td>
      <td>会津若松市</td>
      <td>http://www.sakeno.com/meigara/80</td>
      <td>含む リンゴ パイナップル 感じる ドライ フルーツ よう 上品 果実 的 香味 バランス ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>176</th>
      <td>177</td>
      <td>168</td>
      <td>住吉</td>
      <td>すみよし</td>
      <td>樽平酒造</td>
      <td>山形県</td>
      <td>東置賜郡</td>
      <td>http://www.sakeno.com/meigara/247</td>
      <td>かすか 杉 香り ある 酒 自体 冷 辛口 角 トンガッタ よう 感じ 痛い なる よう 味...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>177</th>
      <td>178</td>
      <td>168</td>
      <td>天吹</td>
      <td>あまぶき</td>
      <td>天吹酒造</td>
      <td>佐賀県</td>
      <td>三養基郡</td>
      <td>http://www.sakeno.com/meigara/288</td>
      <td>精米 歩合 60 % 米 酒 酸味 ある 辛口 うまい ｡ 温める うまみ 消える 酸味 残...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>178</th>
      <td>179</td>
      <td>168</td>
      <td>会津中将</td>
      <td>あいづちゅうじょう</td>
      <td>鶴乃江酒造</td>
      <td>福島県</td>
      <td>会津若松市</td>
      <td>http://www.sakeno.com/meigara/396</td>
      <td>うまい すごい きれい 酒 食 中 うまい 美味しい 香り 穏やか 甘味 酸味 旨味 バラン...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>179</th>
      <td>180</td>
      <td>168</td>
      <td>豊賀</td>
      <td>とよか</td>
      <td>高沢酒造</td>
      <td>長野県</td>
      <td>上高井郡</td>
      <td>http://www.sakeno.com/meigara/1601</td>
      <td>豊 賀 米 吟醸 天女 しずく 中 取る 濾過 生 原酒 美山 錦 ５ ９ 長野 酵母 ２ ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>180</th>
      <td>181</td>
      <td>168</td>
      <td>美丈夫</td>
      <td>びじょうぶ</td>
      <td>浜川商店</td>
      <td>高知県</td>
      <td>安芸郡</td>
      <td>http://www.sakeno.com/meigara/375</td>
      <td>間違い ない 上品 酒 質 値段 安い たまらない 辛口 おとなしい 酒 味 辛口 クリア ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>181</th>
      <td>182</td>
      <td>168</td>
      <td>菊正宗</td>
      <td>きくまさむね</td>
      <td>菊正宗酒造</td>
      <td>兵庫県</td>
      <td>神戸市</td>
      <td>http://www.sakeno.com/meigara/842</td>
      <td>何 よい の 菊 正 純 米 目 入る 買う みる 冷やす 書く ある 冷やす 飲む みる ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>182</th>
      <td>183</td>
      <td>168</td>
      <td>富久長</td>
      <td>ふくちょう</td>
      <td>今田酒造本店</td>
      <td>広島県</td>
      <td>東広島市</td>
      <td>http://www.sakeno.com/meigara/1040</td>
      <td>甘み ある 有名 米 吟醸 みたい 女性 勧める 地元 広島 大手 スーパー 配置 する れ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>184</th>
      <td>185</td>
      <td>168</td>
      <td>一本義</td>
      <td>いっぽんぎ</td>
      <td>一本義久保本店</td>
      <td>福井県</td>
      <td>勝山市</td>
      <td>http://www.sakeno.com/meigara/1083</td>
      <td>文句 ない 爽快 感 至高 廉価 (＾∇＾) 淡い 麗 辛口 香り 昔 ある 日本 酒 ( ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>185</th>
      <td>186</td>
      <td>168</td>
      <td>竹鶴</td>
      <td>たけつる</td>
      <td>竹鶴酒造</td>
      <td>広島県</td>
      <td>竹原市</td>
      <td>http://www.sakeno.com/meigara/1036</td>
      <td>竹 鶴 純 米 熱 燗冷まし 新た 飲む 方 教える くれる にごり酒 純 米 にごる 冷 ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>186</th>
      <td>187</td>
      <td>168</td>
      <td>丹沢山</td>
      <td>たんざわさん</td>
      <td>川西屋酒造店</td>
      <td>神奈川県</td>
      <td>足柄上郡</td>
      <td>http://www.sakeno.com/meigara/498</td>
      <td>すう 私 蔵 訪問 する 吟 旨い さ 惚れ込む 販売 上 策 名 する その後 当時 ２ ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>187</th>
      <td>188</td>
      <td>168</td>
      <td>奥</td>
      <td>おく</td>
      <td>山崎合資</td>
      <td>愛知県</td>
      <td>西尾市</td>
      <td>http://www.sakeno.com/meigara/1253</td>
      <td>さわやか ラムネ 香 しつこい 甘味 酸味 する 炭酸 夏 酒 する ドライ 仕上がり ソフ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>188</th>
      <td>189</td>
      <td>168</td>
      <td>十九</td>
      <td>じゅうく</td>
      <td>尾澤酒造場</td>
      <td>長野県</td>
      <td>長野市</td>
      <td>http://www.sakeno.com/meigara/1106</td>
      <td>最初 感じる の かすか 吟醸 香 中 甘い さ 味 濃い 酒 味 強い すぎる 寿司 一緒...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>189</th>
      <td>190</td>
      <td>168</td>
      <td>長陽福娘</td>
      <td>ちょうようふくむすめ</td>
      <td>岩崎酒造</td>
      <td>山口県</td>
      <td>萩市</td>
      <td>http://www.sakeno.com/meigara/1742</td>
      <td>日本 酒 度 甘い ほう 夏みかん よう 香り のどごし いい 甘い 酒 飲む 進む くどい...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>190</th>
      <td>191</td>
      <td>168</td>
      <td>白龍（福井県）</td>
      <td>はくりゅう</td>
      <td>吉田酒造（福井県）</td>
      <td>福井県</td>
      <td>吉田郡</td>
      <td>http://www.sakeno.com/meigara/1288</td>
      <td>米 吟醸 頂く 米 吟醸 飲める 味 辛口 飲める すぎる ちゃう 気 する 値段 する 満足</td>
      <td>3</td>
    </tr>
    <tr>
      <th>192</th>
      <td>193</td>
      <td>168</td>
      <td>三連星</td>
      <td>さんれんせい</td>
      <td>美冨久酒造</td>
      <td>滋賀県</td>
      <td>甲賀市</td>
      <td>http://www.sakeno.com/meigara/1653</td>
      <td>含む 生らす いる フレッシュ 風味 綺麗 質感 中盤 余韻 甘酸っぱい 酸味 印象 的 酸...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>193</th>
      <td>194</td>
      <td>168</td>
      <td>天寿</td>
      <td>てんじゅ</td>
      <td>天寿酒造</td>
      <td>秋田県</td>
      <td>由利本荘市</td>
      <td>http://www.sakeno.com/meigara/197</td>
      <td>原酒 香り 少ない 旨味 アルコール 感 強い 涼 冷え 常温 非常 旨い ここ 酒 近い ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>194</th>
      <td>195</td>
      <td>168</td>
      <td>天覧山</td>
      <td>てんらんざん</td>
      <td>五十嵐酒造</td>
      <td>埼玉県</td>
      <td>飯能市</td>
      <td>http://www.sakeno.com/meigara/1120</td>
      <td>美山 錦 精米 歩合 65 %、 日本 酒 度 - 5 酸度 1 . 8 アルコール 15 ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>195</th>
      <td>196</td>
      <td>168</td>
      <td>酔右衛門</td>
      <td>よえもん</td>
      <td>川村酒造店</td>
      <td>岩手県</td>
      <td>花巻市</td>
      <td>http://www.sakeno.com/meigara/1269</td>
      <td>アルコール 度数 17 18 精米 歩合 50 ％ 発泡 炭酸 これ 華やか フレッシュ 吟...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>196</th>
      <td>197</td>
      <td>168</td>
      <td>大倉</td>
      <td>おおくら</td>
      <td>大倉本家</td>
      <td>奈良県</td>
      <td>香芝市</td>
      <td>http://www.sakeno.com/meigara/1273</td>
      <td>雄 町 ひる ひかり オオセト ３ 種類 米 作る 酒 責め ところ ブレンド 責め 責め ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>197</th>
      <td>198</td>
      <td>168</td>
      <td>高千代</td>
      <td>たかちよ</td>
      <td>高千代酒造</td>
      <td>新潟県</td>
      <td>南魚沼市</td>
      <td>http://www.sakeno.com/meigara/1442</td>
      <td>2 年 前 飲む 感動 する 日本 酒 香り メロン 味 重い 飲める いる かちよ 濃厚 ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>198</th>
      <td>199</td>
      <td>168</td>
      <td>相模灘</td>
      <td>さがみなだ</td>
      <td>久保田酒造（神奈川県）</td>
      <td>神奈川県</td>
      <td>相模原市</td>
      <td>http://www.sakeno.com/meigara/1241</td>
      <td>開 栓 直後 含む ピリリ ガス 感 原酒 濃厚 フレッシュ バター 甘味 口 中 一 杯 ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>199</th>
      <td>200</td>
      <td>168</td>
      <td>七賢</td>
      <td>しちけん</td>
      <td>山梨銘醸</td>
      <td>山梨県</td>
      <td>北杜市</td>
      <td>http://www.sakeno.com/meigara/506</td>
      <td>今 日本 酒 苦手 敬遠 する 毎年 お世話 なる いる 山中湖 民宿 主人 すすめ 試す ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>86</th>
      <td>87</td>
      <td>77</td>
      <td>松の司</td>
      <td>まつのつかさ</td>
      <td>松瀬酒造</td>
      <td>滋賀県</td>
      <td>蒲生郡</td>
      <td>http://www.sakeno.com/meigara/793</td>
      <td>松 司 純 米 吟醸 楽 しぼる たて 山田 錦 吟 吹雪 ６ ０ ２ ８ ＢＹ 僅か セメ...</td>
      <td>4</td>
    </tr>
    <tr>
      <th>5</th>
      <td>6</td>
      <td>6</td>
      <td>飛露喜</td>
      <td>ひろき</td>
      <td>廣木酒造本店</td>
      <td>福島県</td>
      <td>河沼郡</td>
      <td>http://www.sakeno.com/meigara/409</td>
      <td>今宵 息子 家 飲む こと 飛 露 喜 特別 純 米 セレクト 予想 する れる 飲む 飽き...</td>
      <td>5</td>
    </tr>
  </tbody>
</table>
<p>200 rows × 10 columns</p>
</div>



最後に結果を出力します。


```python
target_cols = ["rank", "meigara", "kuramoto", "detail_url", "cluster"]
result_df[target_cols].to_csv(CLUSTER_PATH, index=False)
```

このような流れで単語ベースのクラスタリングが行えます。

## 7. おわりに

　今回の解析はまだまだ不足している点があります。例えば以下のような点を考慮しませんでした。
 
 * 形態素解析用辞書の改善
   * デフォルトのままだと例えば「山田錦」が「山田」＋「錦」に分割されてしまう。
 * 否定語の扱い
   * 美味しくない → 美味しい ＋ ない と分割され、このままでは「美味しい」としてカウントされてしまう。
 * 数値を含んだ単語の取扱
   * 例えばアルコール度数を表すような数値や、精米歩合の数値などがうまく扱えていない。
 * ノイズとなるような単語のカット
   * 「Wow very oishii Sake ! 」などのテキストが含まれているが、このようなものが含まれていてもあまり有益な結果をえられないので、カットすべき。
 
これらの点などを改善していくことにより、より我々の感覚と近い結果を得られるようになります。
