# サンプルコード記述時に気になったこと

チュ－トリアルは[こちら](https://qiita.com/leafia78/items/73cc7160d002a4989416)を参考にした。

windowsのmongoDB導入方法は[こちら](https://garafu.blogspot.jp/2016/12/install-mongodb.html)を参考に

* body-paraser て何？
    * 直訳でボディ解析的なこと？
    * [github](https://github.com/expressjs/body-parser)のページで確認したらNode.js用のミドルウェアでレスポンスを解析してくれる機能
    * 以下の4種類の解析をサポート 
        * JSON body parser
        * Raw body parser
        * Text body parser
        * URL-encoded form body parser
    * malthpartは対象外なので別のものを用意する必要がありそう
        * formDataで動画像を送っても解析できない？
        * DataUrlに変換すれば大丈夫そう

## mongoDB について
ドキュメント思考DB で NoSQLの一種らしい。
RDBとはぜんぜん違う仕様だったので新鮮でした。

ざっくりいうとJson形式で情報を保存できる！
ただトランザクションがないのでデータ管理は気をつける必要がある

DB ＞ コネクション > ドキュメント（JSON）の階層になっている。
（上記チュートリアルでは
ExpressAPI(DB).articlemodels(コネクション).**(JSON)という構造になる。
joinは出来ない

### mongoose
Node.js用のmongoDBライブラリ

#### model の定義
```javascript
var mongoose = require('mongoose'); //mongoDBに接続するためのライブラリ
var Schema = mongoose.Schema; //mongoDBのスキーマを作る
var moment = require('moment');
var defaultAuthor = "menom";

// 疑似スキーマを定義する
// ライブラリ経由では定義したスキーマでのやりとりが可能
// ただし、DB自体にはすきなJSONを登録できるので注意
var ArticleSchema = new Schema({
    title: String,
    text: String,
    author: String,
    date: String,
});

ArticleSchema.methods.setDate = function () {
    //作った時間をセット
    this.date = moment().format("YYYY-MM-DD HH:mm:ss");
};

ArticleSchema.methods.setAuthor = function (author) {
    this.author = author || defaultAuthor
}

// スキーマをモデルとしてコンパイルし、それをモジュールとして扱えるようにする
module.exports = mongoose.model('ArticleModel', ArticleSchema);
```
#### 定義したモデルを使ってCURD処理する

```javascript
router.post('/', function (req, res) {

    // モデル作成．
    var Article = new ArticleModel();

    // データを詰め込む
    Article.title = req.body.title;
    Article.text = req.body.text;
    Article.setDate();
    Article.setAuthor(req.body.author);

    // 保存処理
    Article.save(function (err) {
        if (err) {
            // エラーがあった場合エラーメッセージを返す
            res.send(err);
        } else {
            // エラーがなければ「Success!!」
            res.json({
                message: 'Success!!'
            });
        }
    });
});

router.get('/articleAuthor/:author', function (req, res) {
    var author = req.params.author;
    ArticleModel
        // find() の引数にプロパティ名と値を与えることでマッチするデータのみ取得可能
        .findOne({
            author: author
        })
        .then((article) => {
            UserModel.findOne({
                screen_name: article.author
            }).then((user) => {
                res.json({
                    article: article.title,
                    user: user._id
                });
            })
        });
});


```

###どのような場面でmongoDBが用いられるのか

* データ整合性よりも速度が求められる場合
* スキーマを事前に定義できない場合
    * しかし、RDBよりもスキーマ管理に気を配っておかないと、どんなデータが保存されているのかわからなくなってしまいます（調べるのは面倒です）。どんな構造のJSONが保存されているのか、しっかりドキュメント化しておくとよい
    * node.js用ライブラリmongooseはスキーマ定義してやり取りできるのでJsonの構造が分かりやすい（フィールドが抜けたりはしますが）

### RDBを使ったほうがいい場面
* リレーションが多い場合 オブジェクト同士の関連が強いシステムなど
* トランザクション処理が多い場合 厳密なデータ整合性が求められるシステム

