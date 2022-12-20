---
title: "Behavior Treeで実装する敵AI"
emoji: "🤖" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["Unity"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

この記事は[Unity Advent Calendar 2022](https://qiita.com/advent-calendar/2022/unity)の12日目の記事です

NPCや敵のAIを実装しようと思った時にBehavior Treeがいいよと教えてもらったので、資料読んだりBehavior Tree自体を実装してみたりして学んでました。ただこういうAIを実装しようとしたら実際はこんなツリーになるよって解説してるところが少ないと思ったので、敵AIを段階的に実装して、ツリーがどういう構造に変わっていくのか解説する記事を書こうと思いました。

今回はBehavior TreeでAIを実装するにあたって、UnityのアセットのBehavior Designerを使っていますが、ツリー自体は他のアセットや自前実装でも参考になると思います。
https://assetstore.unity.com/packages/tools/visual-scripting/behavior-designer-behavior-trees-for-everyone-15277

## 実装例

### 1. プレイヤーを見つけたら追いかけて攻撃する

まず最初に、プレイヤーが視界内に入ったら追いかけて攻撃する単純なAIを作ってみます。

![敵に追いかけられる ツリー](https://storage.googleapis.com/zenn-user-upload/415ddcdc237b-20221220.jpg)
![敵に追いかけられる](https://storage.googleapis.com/zenn-user-upload/69140eb9f7f6-20221220.gif)

### 2. プレイヤーが一定距離離れたら追いかけるのをやめる

このままでは一度見つかると永久に追いかけられるので、ある程度距離が離れたら追いかけるのをやめるようにしてみます。

![離れたら追いかけるのをやめる ツリー](https://storage.googleapis.com/zenn-user-upload/6019b7270a44-20221220.jpg)
![離れたら追いかけるのをやめる](https://storage.googleapis.com/zenn-user-upload/4104a10a35f0-20221220.gif)

行動を分岐させる場合、Selectorの下に優先度が高い行動を左から並べていく感じです。

### 3. HPが低いと逃げる

さらにAIっぽくするために、状況に応じて行動を変化させるようにしてみます。HPが少ない場合はプレイヤーから逃げるようにしてみます。

![HP低いと逃げる ツリー](https://storage.googleapis.com/zenn-user-upload/1ff9de5937d8-20221220.jpg)
![HP低いと逃げる](https://storage.googleapis.com/zenn-user-upload/42cf882a4142-20221220.gif)

そこそこツリーが大きくなってきましたね。2で実装したツリーにSelectorを追加して、左に逃げるロジックを追加したので、行動の優先順位は逃げる > 見失う > 攻撃する になっています。

### 4. プレイヤーを発見したら仲間に知らせる

次はプレイヤーを発見したら周囲の敵に知らせる機能を追加してみます。

![仲間に知らせる ツリー](https://storage.googleapis.com/zenn-user-upload/a28af6fc7b68-20221220.jpg)
![仲間に知らせる](https://storage.googleapis.com/zenn-user-upload/076ce42f6867-20221220.gif)

「プレイヤーを発見したか」のConditionalの右に置いたので、プレイヤーを発見している場合は常に周囲の敵に知らせるようになっています。

### 5. 見失ったら索敵モードに移行する

今のままだとプレイヤー未発見時は棒立ちのままなので、周囲をランダムに歩き回ってプレイヤーを索敵するようにしてみます。

![索敵に戻る ツリー](https://storage.googleapis.com/zenn-user-upload/d7533d89719b-20221220.jpg)
![索敵に戻る](https://storage.googleapis.com/zenn-user-upload/151583959567-20221220.gif)

これぐらいの大きさのツリーで索敵・追跡・攻撃・逃走・仲間を呼ぶが行える敵AIを実装することができました。

## Behavior Designerを使ってみて

ここからは今回使用したBehavior Designerを使ってみた感想を書きます

### 便利

かなり便利です。直感的に操作できますし、Task同士を繋げて左から順番に並べたら実行順もよしなにしてくれます。またBehavior Designerを開いたままPlayすると、ツリーがどのように実行されたかリアルタイムで可視化してくれますし、Taskにブレークポイントも貼れます。
![実行中のツリー](https://storage.googleapis.com/zenn-user-upload/dfccf78c0c43-20221220.jpg)
Behavior Tree自体はシンプルなデザインパターンですし、わざわざアセットを買わなくても自分で実装することもできますが、やはりGUI上でツリーの構造と実行結果を確認できるのはとても楽ですし、買う価値はあると思います（それでも通常価格は高すぎるのでセールで買いましたが…）

### 全部Behavior Designerでやるのはキツイ

Behavior Designerには元々ある程度Taskが用意されていて、UnityのAPIも使えるので、Unityビジュアルスクリプティングのようにノーコードでロジックを作っていくこともできます。（追加パックを買えばさらにTaskが増えます）
![ノーコードもできる](https://storage.googleapis.com/zenn-user-upload/bc4a3e1727e0-20221220.jpg)
*GameObject.Findした結果をNavMeshAgentのDestinationに設定したい*
ただあるTaskの結果を他のTaskで使う場合、Behavior Designer上で変数を定義してその変数越しに渡していくことになります。
![](https://storage.googleapis.com/zenn-user-upload/741040129232-20221220.jpg)
<https://opsive.com/support/documentation/behavior-designer/variables/>

C#で書いてるコードとは別に変数を管理しなきゃいけないので大変です。思い切って閉じたBehavior Designerの中で全て実装してもらうという手もありますが、正直プログラマー視点だと使いづらいなという気持ちがあり、また非プログラマーに使ってもらうとしても、UnityのAPIを直接触らせてロジック組んでもらいたくないなと思いました。

今回紹介するにあたり、できるだけBehavior Designer上で完結する実装例を見せようかなと思ったんですが、あんまり実用的ではないと思ったので、独自Taskを実装してそれをSelectorやSequenceで繋げていくという形にしました。もっと良い使い方があれば教えてください。

### 敵専用Taskを作ってみる

公式通りに作るとどこでも使えるTaskになってしまうので、敵専用のTaskを作ってみます。

独自Taskを作る際、TaskからはBehavior TreeコンポーネントをアタッチしたGameObjectを参照することができるのですが、ロジックが実装されているMonobehaviorクラスにアクセスするためには、都度`GetComponent()`しなければいけません。

```csharp
    public class TestAction : Action
    {
        private Enemy _enemy;

        public override void OnAwake()
        {
            _enemy = GetComponent<Enemy>();
        }
    }
```

今回は`Enemy`クラス経由で敵を操作する作りにしていますが、敵専用Taskを作るたびに`GetComponent()`で`Enemy`クラスの参照を取ってくるのは面倒なので、`EnemyAction`という基底クラスを作って、そこで`GetComponent()`させます。

```csharp
    [RequiredComponent(typeof(Enemy))]
    public class EnemyAction : Action
    {
        protected Enemy EnemySelf { get; private set; }

        public override void OnAwake()
        {
            EnemySelf = gameObject.GetComponent<Enemy>();
        }
    }
```

Taskにも`RequiredComponent`を付けることができ、このTaskをGUI上で配置すると同時にGameObjectにコンポーネントを自動的に付けてくれるようになるので、付け忘れもないです。

```csharp
    [TaskName("プレイヤーを攻撃する")]
    [TaskCategory("Enemy")]
    public class Attack : EnemyAction
    {
        public override TaskStatus OnUpdate()
        {
            return EnemySelf.Attack() ? TaskStatus.Success : TaskStatus.Failure;
        }
    }
```

`TaskName`でTask名を設定、`TaskCategory`でカテゴリーに振り分けることができます。カテゴリー分けておくだけでだいぶ見やすくなるので、都度設定したほうが良いです。
また今回は敵専用のActionを作成しましたが、Conditionalでも手順は同様です。

こんな感じで敵専用のTaskを作ってみましたが、もっとうまいやり方があれば教えてください。

## 参考にさせてもらったもの

https://light11.hatenadiary.com/entry/2019/06/06/223659
https://www.slideshare.net/torisoup/unity-behavior-treeai
https://learn.unity.com/project/behaviour-trees
実際にBehavior Treeを実装してみるUnity公式チュートリアルです。英語ですがかなり分かりやすいのでおすすめです。
