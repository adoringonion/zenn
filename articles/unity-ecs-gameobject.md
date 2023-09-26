---
title: "Unity ECSとGameObjectを連携させる"
emoji: "🐰"
type: "tech"
topics: ["unity"]
published: true
---

## はじめに

既存のGameObjectとECSの連携について、プレビュー版の時の解説をされている記事はいくつか見かけたのですが、正式版だと少しやり方が変わってるようなので、勉強がてら記事にしてみました。
今回はUnity公式の[Spawner Example](https://docs.unity3d.com/Packages/com.unity.entities@1.0/manual/ecs-workflow-example.html)をベースとして、SubScene内にスポーンした敵オブジェクトを従来のGameObjectで構成されたプレイヤーオブジェクトを追いかけるものを作っていきます。

## 事前準備
### Player
まず`Player`オブジェクトを用意しておきます。RigidbodyをアタッチしたCubeを作成し、WSADで移動できるようにしておきます。

```csharp
public class PlayerMovement : MonoBehaviour
{
    [SerializeField] private Rigidbody playerRigidbody;
    [SerializeField] private float speed = 10.0f;

    private void FixedUpdate()
    {
        var x = Input.GetAxis("Horizontal");
        var z = Input.GetAxis("Vertical");
        var move = new Vector3(x, 0, z);
        playerRigidbody.MovePosition(playerRigidbody.transform.position + move * (speed * Time.deltaTime));
    }
}
```

### Enemy
`Spawner`が`Enemy`が付いたPrefabをスポーンできるようにしておきます。`Enemy`は`targetPosition`の方に向かって移動するようになっているので、`targetPosition`に`Player`の位置を更新し続けることで`Player`を追跡するようにしておきます。


```csharp
public struct Enemy : IComponentData
{
       public float3 targetPosition;
       public float speed;
}

public class EnemyAuthoring : MonoBehaviour
{
       [SerializeField] public float speed = 1.0f; 
}

public class EnemyBaker : Baker<EnemyAuthoring>
{
    public override void Bake(EnemyAuthoring authoring)
    {
        var enemy = new Enemy
        {
            speed = authoring.speed,
            targetPosition = new float3(0,0,0)
        };
        AddComponent(GetEntity(TransformUsageFlags.None), enemy);
    }
}
```

```csharp
public partial struct EnemySystem : ISystem
{
    public void OnCreate(ref SystemState state) { }

    public void OnDestroy(ref SystemState state) { }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        new EnemyMovementJob
        {
            DeltaTime = Time.deltaTime
        }.ScheduleParallel();
    }
}

[BurstCompile]
[StructLayout(LayoutKind.Auto)]
public partial struct EnemyMovementJob : IJobEntity
{
    public float DeltaTime { get; set; }
    
    private void Execute([ReadOnly] ref Enemy enemy, ref LocalTransform transform)
    {
        var direction = enemy.targetPosition - transform.Position;
        var distance = math.length(direction);
        if (!(distance > 0.1f)) return;
        var move = math.normalize(direction) * enemy.speed * DeltaTime;
        transform.Position += move;
    }
}
```

## GameObject → ECS
GameObjectからECSへのアクセスは比較的楽です。`World.DefaultGameObjectInjectionWorld.EntityManager`を使うとMonoBehaviourから`EntityManager`を取得できます。`EntityManager`からEntity検索用のクエリを組み立てたあと、それを通じてEntityを取得できます。
```csharp
public class PlayerPositionSender : MonoBehaviour 
{
    [SerializeField] Transform playerTransform;

    private void Update()
    {
        var entityManager = World.DefaultGameObjectInjectionWorld.EntityManager;
        var query = entityManager.CreateEntityQuery(typeof(PlayerPositionReceiver));
        var entity = query.GetSingletonRW<PlayerPositionReceiver>();
        entity.ValueRW.targetPosition = playerTransform.position;
    }
}
```
今回はPlayerの位置をECSに送るための`PlayerPositionSender`と、ECS側でそれを受け取る`PlayerPositionReceiver`を作っています。

```csharp
public partial struct PlayerPositionReceiverSystem : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<PlayerPositionReceiver>();
        state.EntityManager.CreateSingleton<PlayerPositionReceiver>(); // シングルトンとして生成しておきます
    }

    public void OnDestroy(ref SystemState state)
    {
        
    }
    
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var targetPosition = SystemAPI.GetSingleton<PlayerPositionReceiver>().targetPosition;
        new PlayerPositionReceiverJob
        {
            TargetPosition = targetPosition
        }.ScheduleParallel();
    }
}

[BurstCompile]
[StructLayout(LayoutKind.Auto)]
public partial struct PlayerPositionReceiverJob : IJobEntity
{
    public float3 TargetPosition;
    
    private void Execute(ref Enemy enemy)
    {
        enemy.targetPosition = TargetPosition;
    }
}
```

もちろん`PlayerPositionSender`からEnemyがついたEntityを検索し、直接値をセットしていくこともできます。ただその場合だと、Entityの数が増えていくのに比例してMonoBehaviour上での計算時間が増えていくため、ECSの恩恵をあまり受けることができません。

以下のように次々とスポーンする`Enemy`をGameObjectの`Player`に追跡させることができました。
![](https://storage.googleapis.com/zenn-user-upload/37244fa6c046-20230926.gif)

## ECS → GameObject
逆の場合は少々面倒です。Managed Componentを使ってECS側からGameObjectにアクセスします。まずは`Player`オブジェクトを保持するだけの`PlayerObjectRef`を作ります。

```csharp
public class PlayerObjectRef : MonoBehaviour
{
    public GameObject playerObject;
}
```
次にこの`PlayerObjectRef`をManagedComponentに変換する処理と、変換後の`PlayerObjectRefManaged`を作ります。

```csharp
public class PlayerObjectRefManaged : IComponentData
{
    public GameObject playerObject;

    public PlayerObjectRefManaged(){} // ManagedComponentは引数なしのコンストラクタが必要
}

public partial struct PlayerObjectRefInitSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state) {}

    public void OnUpdate(ref SystemState state)
    {
        state.Enabled = false; // 一回だけの実行でいいので無効化しておく
        InitPlayerObjectRefManaged(ref state);
    }

    [BurstCompile]
    public void OnDestroy(ref SystemState state) {}
    
    private void InitPlayerObjectRefManaged(ref SystemState state)
    {
            var go = GameObject.Find("Player");
            var playerObjectRef = go.GetComponent<PlayerObjectRef>();
            var playerObjectRefManaged = new PlayerObjectRefManaged
            {
                playerObject = playerObjectRef.playerObject
            };

            var entity = state.EntityManager.CreateEntity();
            state.EntityManager.AddComponentData(entity, playerObjectRefManaged);
    }
}
```

`GameObject.Find`と`GetComponent`を使って`PlayerObjectRef`を取得し、それを`PlayerObjectRefManaged`に詰めて`EntityManager`に登録しています。その後各`Enemy`のEntityに`Player`の位置を送る処理を書きます。

```csharp
public partial struct PlayerObjectRefSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<PlayerObjectRefManaged>(); 
    }

    public void OnUpdate(ref SystemState state)
    {
        var playerObjectRefManaged = SystemAPI.ManagedAPI.GetSingleton<PlayerObjectRefManaged>();
        var targetPosition = playerObjectRefManaged.playerObject.transform.position;
        new SetTargetPositionJob
        {
            TargetPosition = targetPosition
        }.ScheduleParallel();
    }

    [BurstCompile]
    public void OnDestroy(ref SystemState state) {}
}

[BurstCompile]
public partial struct SetTargetPositionJob : IJobEntity
{
    public float3 TargetPosition { get; set; }
    
    private void Execute(ref Enemy enemy)
    {
        enemy.targetPosition = TargetPosition;
    }
}
```
これでGameObject → ECSの時と同様に、`Enemy`に`Player`を追跡させることができました。

### ManagedComponentの注意点
ManagedComponentを使うとECS側からGameObjectの参照を取ることができますが、注意することがあります。
https://docs.unity3d.com/Packages/com.unity.entities@1.0/manual/components-managed.html
- ManagedComponentを使用している関数はBurstCompileの対象にできない
- 通常のComponent(UnmanagedComponent)よりもアクセス効率が悪い

一つ目に関しては、ManagedComponentにアクセスする部分を限定し、それ以外の部分を細く区切ってBurstCompileの対象にしたり、Jobにすることで影響を最小限にできます。

二つ目に関しては、ManagedComponentが通常のComponentと管理の仕方が違うことに起因しています。以下DeepL訳

> Unmanaged componentsとは異なり、Unityはmanaged componentsをチャンクに直接格納しません。その代わりに、UnityはWorld全体で1つの大きな配列に格納します。そしてチャンクには、関連するmanaged componentsの配列インデックスが格納されます。つまり、Entityのmanaged componentにアクセスすると、Unityは余分なインデックス検索を処理します。このため、managed componentsはunmanaged componentsよりも最適化されません。
Managed componentsのパフォーマンスへの影響から、可能であれば代わりにunmanaged componentsを使用するべきです。

そのためManagedComponentを使う機会をできるだけ減らすことが望ましいです。

## 比較
ではGameObject → ECSとECS → GameObjectとではパフォーマンスに差があるのか見てみたいと思います。両者ともに差分がないスポーン処理や`Enemy`の移動処理についてはJobSystemで最適化してあり、`Enemy`を一万体スポーンさせた時を計測してみます。

### GameObject → ECS
![](https://storage.googleapis.com/zenn-user-upload/9674e7f799a8-20230927.png)
![](https://storage.googleapis.com/zenn-user-upload/93c687ab4901-20230927.png)
GameObjectからECSへアクセスする`PlayerPositionSender`の処理時間が0.068ms、各`Enemy`へ値をセットする`PlayerPositionReceiverJob`の処理時間が0.006msだったため、合計0.074msです。

### ECS → GameObject
![](https://storage.googleapis.com/zenn-user-upload/fd3064da678f-20230927.png)
![](https://storage.googleapis.com/zenn-user-upload/4a5ae803d531-20230927.png)
ECSからGameObjectへアクセスする`PlayerObjectRefSystem`の処理時間が0.066ms、各`Enemy`へ値をセットする`SetTargetPositionJob`の処理時間が0.019msだったため、合計0.085msです。

### 結果
`PlayerPositionReceiverJob`と`SetTargetPositionJob`とで0.013msの差がありましたが、両方とも実装が全く同じなため、おそらく誤差だと考えられます。そのため今回のような簡単な処理では、GameObject → ECSとECS → GameObjectのどちらを使ってもパフォーマンスにそこまで差がないと考えられ、パフォーマンスの観点ではどちらを使うほうがいいか判断を下すことができませんでした。

## さいごに
今回のGameObjectとECSの連携にどれくらいの需要があるかは分かりませんが（そもそもECSみんな使ってる？）、部分的にECSを導入する方法として有効かもしれません。質問やご指摘などあればお気軽にコメントください。

今回調べるにあたってたくさんサポートしていただいた[notargsさん](https://twitter.com/notargs)、ありがとうございました。

## 参考
- [ECSの公式サンプルリポジトリ](https://github.com/Unity-Technologies/EntityComponentSystemSamples/tree/master)
  - 今回の実装はこのリポジトリの[HelloCube/GameObjectSync](https://github.com/Unity-Technologies/EntityComponentSystemSamples/tree/master/EntitiesSamples/Assets/HelloCube/8.%20GameObjectSync)をベースにしています
