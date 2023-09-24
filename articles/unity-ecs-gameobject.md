---
title: "Unity ECSとGameObjectを連携させる"
emoji: "🕌"
type: "tech"
topics: ["unity"]
published: false
---

## 事前準備
### Player
まずPlayerオブジェクトを用意しておきます。RigidbodyをアタッチしたCubeを作成し、WSADで移動できるようにしておきます。

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
SpawnerがEnemyが付いたPrefabをスポーンできるようにしておきます。EnemyはtargetPositionの方に向かって移動するようになっているので、targetPositionにPlayerの位置を更新し続けることでPlayerを追跡するようにしておきます。


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
GameObjectからECSへのアクセスは比較的楽です。`World.DefaultGameObjectInjectionWorld.EntityManager`を使うとMonoBehaviourからEntityManagerを取得できます。EntityManagerからEntity検索用のクエリを組み立てたあと、それを通じてEntityを取得できます。
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
```

もちろんPlayerPositionSenderからEnemyがついたEntityを検索し、直接値をセットしていくこともできます。ただその場合だと、Entityの数が増えていくのに比例してMonoBehaviour上での計算時間が増えていくため、ECSの恩恵をあまり受けることができません。

