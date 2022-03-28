---
title: "Unityでsupabaseのデータベースを使ってみる"
emoji: "🏆" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["Unity", "supabase"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

Firebase の置き換えとして話題の supabase を Unity で使ってみた事例が見つからなかったので、メモ書きとして残していきたいと思います

## 開発環境

- Unity 2020.3.24f1

## NuGet を通して supabase を入れる

supabase の C# クライアントは Nuget で入手できるので、 NuGetForUnity を入れます
https://github.com/supabase-community/supabase-csharp
https://github.com/GlitchEnzo/NuGetForUnity

NuGetForUnity を入れた後、 NuGet > Manage NuGet Packages で supabase で検索すると supabase-cshrap が一番上に出てくるのでインストールします。

!["NuGetでsupabaseを検索する"](https://storage.googleapis.com/zenn-user-upload/ec5a3be36637-20220327.jpg)

ダウンロードが終わると、 Unity に元々入ってる Newtonsoft.Json と supabase-csharp に入っているものがぶつかるので、 Unity のほうを削除します。 Version Cotrol を利用していないなら Package Manager から Version Control ごと削除したほうが楽です。詳しくは以下の Qiita 記事を参考にしてください。

> Multiple precompiled assemblies with the same name Newtonsoft.Json.dll included or the current platform. Only one assembly with the same name is allowed per platform.

https://qiita.com/sakano/items/6fa16af5ceab2617fc0f

## クライアントの初期化

supabase クライアント初期化用のクラスを作ります。今回は URL と Key を SerializeField ごしに渡していますが、 `Client.Initialize()` に渡せるなら何でもいいです。

```csharp
public class SupabaseClient : MonoBehaviour
{
    [SerializeField] private string supabaseUrl;
    [SerializeField] private string supabaseKey;

    private async void Awake()
    {
        await Client.Initialize(supabaseUrl, supabaseKey);
    }
}
```

## Model クラスの作成

今回は簡単なオンラインランキング機能を実装していきます。

`BaseModel` を継承した Model クラスを作ります。クラスには `Table` attribute を付けてパラメーターにテーブル名を入れ、 プロパティにはそれぞれ `Column` attribute （プライマリーキーは `PrimaryKey` )をつけて対応するカラム名をパラメーターに入れます。今回プライマリーキーの `id` は DB 側でオートインクリメントするように設定してあるので、クライアント側で生成しないように `ShouldInsert` を `false` にしています。

```csharp
[Table("scores")]
public class ScoreModel : BaseModel
{
    [PrimaryKey("id", false)] public int Id { get; set; }
    [Column("score")] public int Score { get; set; }
    [Column("user_name")] public string UserName { get; set; }


    public Score ToScore()
    {
        return new Score(UserName, Score);
    }

    public static ScoreModel FromScore(Score score)
    {
        return new ScoreModel {UserName = score.UserName, Score = score.Value};
    }
}
```

先ほど初期化した supabase クライアントから `From<ScoreModel>` を呼び出すことでテーブルに対する操作を行うことができます。

```csharp
public class SupabaseRepository : IScoreRepository
{
    private readonly Client _supabaseClient;

    public SupabaseRepository(Client supabaseClient)
    {
        _supabaseClient = supabaseClient;
    }

    public async UniTask InsertScore(Score score)
    {
        var scoreModel = ScoreModel.FromScore(score);
        await _supabaseClient.From<ScoreModel>().Insert(scoreModel);
    }

    public async UniTask<List<Score>> FetchTopThirty()
    {
        var response = await _supabaseClient.From<ScoreModel>().Order("score", Constants.Ordering.Descending)
            .Limit(30).Get();
        return response.Models.Select(model => model.ToScore()).ToList();
    }
}
```

詳しくは `postgrest-csharp` の公式ドキュメントを見てください。

https://github.com/supabase-community/postgrest-csharp
https://supabase-community.github.io/postgrest-csharp/api/Postgrest.html

## できたもの

![](https://storage.googleapis.com/zenn-user-upload/d28a581a922b-20220329.jpg)
https://github.com/adoringonion/unity-ranking-supabase