---
title: "Earth, Wind & FireっぽいエフェクトをRenderGraphで実装する"
emoji: "🌈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "shader"]
published: true
---

この記事は [Anthrotech Advent Calendar 2025](https://adventar.org/calendars/11972) の1日目の記事です。Anthrotechについては [こちら](https://anthrotech.dev/) をご覧ください。

## はじめに

RenderGraphはUnity6からURPに導入され、6.5からはデフォルトになって従来の書き方ができなくなるので今のうちに慣れておこうと思い、エフェクトを作ってみます。

今回作るのはEarth Wind and FireのLet's Grooveっぽいエフェクトです。
https://youtu.be/Lrle0x_DHBM?si=gDZG7Sx1TySWDIH4&t=60
大体1分あたりに出てくる、人物にカラフルな残像がつくエフェクトをRenderGraphで実装してきます。

:::message
- Unityバージョン: 6000.2.11f1
- URPバージョン: 17.2.0
:::

## 方針

そもそも残像エフェクトってどう作るねんって話ですが、今回のエフェクトの要件は

- プレイヤーの動きに合わせて残像がつく
- 残像は時間経過でフェードアウトする
- 残像は色相が時間で変化する

という感じなので、以下の画像のような流れで実装します。

![](https://storage.googleapis.com/zenn-user-upload/9c1112ceab23-20251201.png)

## 1. 残像を作る

まずは残像を作っていきましょう。初めにコード全体を載せておきます。

:::details AfterimageFeature.cs

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Experimental.Rendering;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;
using UnityEngine.Rendering.RenderGraphModule;
using UnityEngine.Rendering.RendererUtils;

public class AfterimageRenderFeature : ScriptableRendererFeature
{
    private static readonly int Persistence = Shader.PropertyToID("_Persistence");
    private static readonly int Mix = Shader.PropertyToID("_Mix");
    private static readonly int HistoryTex = Shader.PropertyToID("_HistoryTex");
    private static readonly int BlitTextureId = Shader.PropertyToID("_BlitTexture");

    [System.Serializable]
    public class Settings
    {
        [Range(0f, 1f)] public float trailPersistence = 0.9f; // 1 に近いほど履歴が長く残る
        [Range(0f, 1f)] public float mix = 1f;                 // どれだけ履歴を最終画像に戻すか
        [Range(0, 10)] public int framesBetweenTrails = 0;     // 何フレームおきに蓄積するか（0なら毎フレーム）
        public Shader shader;
        public LayerMask playerRenderingLayer;
        public LayerMask fallbackLayerMask = ~0;
    }

    [SerializeField] private Settings settings = new();

    private AfterimagePass _pass;

    public override void Create()
    {
        if (settings.shader == null)
            settings.shader = Shader.Find("Hidden/Afterimage/Accumulation");

        _pass = new AfterimagePass(settings);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (!renderingData.cameraData.postProcessEnabled || settings.shader == null)
            return;

        renderer.EnqueuePass(_pass);
    }

    protected override void Dispose(bool disposing)
    {
        _pass?.Dispose();
        base.Dispose(disposing);
    }

    private class AfterimagePass : ScriptableRenderPass
    {
        private readonly Settings _settings;
        private Material _material;
        private RTHandle _historyA, _historyB;
        private bool _toggle;
        private bool _historyValid;
        private RTHandle _playerColor;
        private RTHandle _playerDepth;
        private int _frameCounter; // 蓄積を間引くためのカウンタ
        private readonly List<ShaderTagId> _shaderTags = new()
        {
            new ShaderTagId("UniversalForwardOnly"),
            new ShaderTagId("UniversalForward"),
            new ShaderTagId("SRPDefaultUnlit"),
            new ShaderTagId("LightweightForward")
        };

        public AfterimagePass(Settings settings)
        {
            _settings = settings;
            renderPassEvent = RenderPassEvent.AfterRenderingPostProcessing;
        }

        public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData)
        {
            if (_settings.shader == null)
                return;

            if (_material == null)
                _material = CoreUtils.CreateEngineMaterial(_settings.shader);

            UniversalCameraData cameraData = frameData.Get<UniversalCameraData>();
            UniversalResourceData resources = frameData.Get<UniversalResourceData>();

            var desc = cameraData.cameraTargetDescriptor;
            desc.msaaSamples = 1;
            desc.depthBufferBits = 0;
            EnsureHistory(desc);       // 履歴用の A/B バッファをカメラ解像度で確保
            EnsurePlayerTargets(desc); // プレイヤーだけを描くためのカラー/深度バッファを確保

            TextureHandle activeColor = resources.activeColorTexture;
            if (!activeColor.IsValid())
                return;

            TextureHandle historyRead = renderGraph.ImportTexture(_toggle ? _historyA : _historyB);
            TextureHandle historyWrite = renderGraph.ImportTexture(_toggle ? _historyB : _historyA);
            TextureHandle playerColor = renderGraph.ImportTexture(_playerColor);
            TextureHandle playerDepth = renderGraph.ImportTexture(_playerDepth);

            // 蓄積を行うフレームかどうか（0なら毎フレーム行う）
            bool accumulateThisFrame = _frameCounter <= 0;
            if (accumulateThisFrame)
                _frameCounter = _settings.framesBetweenTrails;
            else
                _frameCounter--;

            if (!_historyValid)
            {
                AddClearPass(renderGraph, _historyA, "Clear Afterimage History A");
                AddClearPass(renderGraph, _historyB, "Clear Afterimage History B");
                _historyValid = true;
            }

            var playerRendererList = CreatePlayerRendererList(frameData, renderGraph); // 残像対象だけを描くリスト
            using (var builder = renderGraph.AddRasterRenderPass<PlayerCapturePassData>("Afterimage Player Capture", out var passData, profilingSampler))
            {
                passData.RendererList = playerRendererList;
                passData.ColorTarget = playerColor;
                passData.DepthTarget = playeraDepth;

                builder.UseRendererList(passData.RendererList);
                builder.SetRenderAttachment(passData.ColorTarget, 0);
                builder.SetRenderAttachmentDepth(passData.DepthTarget);

                builder.SetRenderFunc((PlayerCapturePassData data, RasterGraphContext ctx) =>
                {
                    ctx.cmd.ClearRenderTarget(RTClearFlags.All, Color.clear, 1f, 0u);
                    ctx.cmd.DrawRendererList(data.RendererList);
                });
            }

            var cameraCopyDesc = activeColor.GetDescriptor(renderGraph);
            cameraCopyDesc.name = "CameraColorCopy";
            TextureHandle cameraCopy = renderGraph.CreateTexture(cameraCopyDesc);
            using (var builder = renderGraph.AddRasterRenderPass<CopyPassData>("Afterimage Camera Copy", out var passData, profilingSampler))
            {
                passData.Source = activeColor;
                passData.Destination = cameraCopy;
                builder.UseTexture(passData.Source);
                builder.SetRenderAttachment(passData.Destination, 0);

                builder.SetRenderFunc((CopyPassData data, RasterGraphContext ctx) =>
                {
                    Blitter.BlitTexture(ctx.cmd, data.Source, Vector2.one, 0f, false);
                });
            }

            using (var builder = renderGraph.AddRasterRenderPass<AccumPassData>("Afterimage Accumulate", out var passData, profilingSampler))
            {
                if (accumulateThisFrame)
                {
                    passData.Source = playerColor;
                    passData.HistoryIn = historyRead;
                    passData.HistoryOut = historyWrite;
                    passData.Material = _material;
                    passData.Persistence = _settings.trailPersistence;
                    passData.Mix = _settings.mix;

                    builder.UseTexture(passData.Source);
                    builder.UseTexture(passData.HistoryIn);
                    builder.SetRenderAttachment(passData.HistoryOut, 0);

                    builder.SetRenderFunc((AccumPassData data, RasterGraphContext ctx) =>
                    {
                        data.Material.SetFloat(Persistence, data.Persistence);
                        data.Material.SetFloat(Mix, data.Mix);
                        data.Material.SetTexture(HistoryTex, data.HistoryIn);
                        Blitter.BlitTexture(ctx.cmd, data.Source, Vector2.one, data.Material, 0);
                    });
                }
                else
                {
                    // スキップ時は何もしない（RenderGraph のお作法として空関数をセット）
                    builder.SetRenderFunc((AccumPassData data, RasterGraphContext ctx) => { });
                }
            }

            using (var builder = renderGraph.AddRasterRenderPass<CopyPassData>("Afterimage Composite", out var passData, profilingSampler))
            {
                // 蓄積をスキップしたフレームは前の履歴（historyRead）をそのまま合成に使う
                passData.Source = accumulateThisFrame ? historyWrite : historyRead;
                passData.CameraColor = cameraCopy;
                passData.Destination = activeColor;
                passData.Material = _material;
                passData.Mix = _settings.mix;
                builder.UseTexture(passData.Source);
                builder.UseTexture(passData.CameraColor);
                builder.SetRenderAttachment(passData.Destination, 0);

                builder.SetRenderFunc((CopyPassData data, RasterGraphContext ctx) =>
                {
                    data.Material.SetTexture(HistoryTex, data.Source);
                    data.Material.SetFloat(Mix, data.Mix);
                    data.Material.SetTexture(BlitTextureId, data.CameraColor);
                    Blitter.BlitTexture(ctx.cmd, data.CameraColor, Vector2.one, data.Material, 1);
                });
            }

            // 蓄積を行ったフレームだけ ping-pong を進める
            if (accumulateThisFrame)
                _toggle = !_toggle;
        }

        public void Dispose()
        {
            RTHandles.Release(_historyA);
            RTHandles.Release(_historyB);
            CoreUtils.Destroy(_material);
            RTHandles.Release(_playerColor);
            RTHandles.Release(_playerDepth);
        }

        private void EnsureHistory(RenderTextureDescriptor desc)
        {
            desc.msaaSamples = 1;
            desc.depthBufferBits = 0;

            if (_historyA != null && _historyA.rt.width == desc.width && _historyA.rt.height == desc.height)
                return;

            RenderingUtils.ReAllocateHandleIfNeeded(ref _historyA, desc, name: "_AfterimageHistoryA");
            RenderingUtils.ReAllocateHandleIfNeeded(ref _historyB, desc, name: "_AfterimageHistoryB");
            _historyValid = false;
        }

        private void EnsurePlayerTargets(RenderTextureDescriptor desc)
        {
            var colorDesc = desc;
            colorDesc.msaaSamples = 1;
            colorDesc.depthBufferBits = 0;

            var depthDesc = desc;
            depthDesc.msaaSamples = 1;
            depthDesc.graphicsFormat = GraphicsFormat.None;
            depthDesc.depthStencilFormat = GraphicsFormat.D32_SFloat;
            depthDesc.depthBufferBits = 32;

            RenderingUtils.ReAllocateHandleIfNeeded(ref _playerColor, colorDesc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_AfterimagePlayerColor");
            RenderingUtils.ReAllocateHandleIfNeeded(ref _playerDepth, depthDesc, FilterMode.Point, TextureWrapMode.Clamp, name: "_AfterimagePlayerDepth");
        }

        private void AddClearPass(RenderGraph renderGraph, RTHandle target, string name)
        {
            TextureHandle imported = renderGraph.ImportTexture(target);
            using var builder = renderGraph.AddRasterRenderPass<ClearPassData>(name, out var passData, profilingSampler);
            passData.Target = imported;
            builder.SetRenderAttachment(passData.Target, 0);
            builder.SetRenderFunc((ClearPassData data, RasterGraphContext ctx) =>
            {
                ctx.cmd.ClearRenderTarget(false, true, Color.clear);
            });
        }

        private class AccumPassData
        {
            internal TextureHandle Source;
            internal TextureHandle HistoryIn;
            internal TextureHandle HistoryOut;
            internal Material Material;
            internal float Persistence;
            internal float Mix;
        }

        private class CopyPassData
        {
            internal TextureHandle Source;
            internal TextureHandle CameraColor;
            internal TextureHandle Destination;
            internal Material Material;
            internal float Mix;
        }

        private class ClearPassData
        {
            internal TextureHandle Target;
        }

        private class PlayerCapturePassData
        {
            internal RendererListHandle RendererList;
            internal TextureHandle ColorTarget;
            internal TextureHandle DepthTarget;
        }

        private RendererListHandle CreatePlayerRendererList(ContextContainer frameData, RenderGraph renderGraph)
        {
            var renderingData = frameData.Get<UniversalRenderingData>();
            var cameraData = frameData.Get<UniversalCameraData>();
            var lightData = frameData.Get<UniversalLightData>();

            var sortFlags = cameraData.defaultOpaqueSortFlags;
            var filterSettings = new FilteringSettings(RenderQueueRange.all, _settings.fallbackLayerMask)
            {
                layerMask = _settings.playerRenderingLayer
            };

            var drawingSettings = RenderingUtils.CreateDrawingSettings(_shaderTags, renderingData, cameraData, lightData, sortFlags);

            var rendererListParams = new RendererListParams(renderingData.cullResults, drawingSettings, filterSettings);
            return renderGraph.CreateRendererList(rendererListParams);
        }
    }
}

:::

:::details AfterimageAccumulation.shader

```hlsl
Shader "Hidden/Afterimage/Accumulation"
{
    Properties { }
    SubShader
    {
        Tags { "RenderPipeline"="UniversalPipeline" }
        ZWrite Off ZTest Always Cull Off Blend One Zero

        Pass
        {
            Name "Afterimage"
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment frag
            #pragma target 4.5

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"

            TEXTURE2D_X(_BlitTexture);    SAMPLER(sampler_BlitTexture);
            TEXTURE2D_X(_HistoryTex);     SAMPLER(sampler_HistoryTex);
            float _Persistence;
            float _Mix;
            float4 _BlitScaleBias;

            struct attributes
            {
                uint vertex_id : SV_VertexID;
            };

            struct varyings
            {
                float4 position_cs : SV_POSITION;
                float2 tex_coord   : TEXCOORD0;
            };
            
            varyings Vert(attributes input)
            {
                varyings output;
                output.position_cs = GetFullScreenTriangleVertexPosition(input.vertex_id);
                output.tex_coord = GetFullScreenTriangleTexCoord(input.vertex_id);
                return output;
            }

            float4 frag(varyings input) : SV_Target
            {
                float2 uv = input.tex_coord * _BlitScaleBias.xy + _BlitScaleBias.zw;
                float4 current = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv);
                float4 history = SAMPLE_TEXTURE2D_X(_HistoryTex, sampler_HistoryTex, uv);

                // Keep brighter pixels from current frame and fade the previous history.
                float4 accumulated = max(current, history * _Persistence);
                return lerp(current, accumulated, _Mix);
            }
            ENDHLSL
        }

        Pass
        {
            Name "AfterimageComposite"
            Blend One Zero
            ZWrite Off ZTest Always Cull Off
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment frag
            #pragma target 4.5

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"

            TEXTURE2D_X(_BlitTexture);    SAMPLER(sampler_BlitTexture);
            TEXTURE2D_X(_HistoryTex);     SAMPLER(sampler_HistoryTex);
            float _Mix;
            float4 _BlitScaleBias;

            struct attributes
            {
                uint vertex_id : SV_VertexID;
            };

            struct varyings
            {
                float4 position_cs : SV_POSITION;
                float2 tex_coord   : TEXCOORD0;
            };

            varyings Vert(attributes input)
            {
                varyings output;
                output.position_cs = GetFullScreenTriangleVertexPosition(input.vertex_id);
                output.tex_coord = GetFullScreenTriangleTexCoord(input.vertex_id);
                return output;
            }

            float4 frag(varyings input) : SV_Target
            {
                float2 uv = input.tex_coord * _BlitScaleBias.xy + _BlitScaleBias.zw;
                float4 baseColor = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv);
                float4 history = SAMPLE_TEXTURE2D_X(_HistoryTex, sampler_HistoryTex, uv);
                return baseColor + history * _Mix;
            }
            ENDHLSL
        }
    }
}
```
:::

### テクスチャの初期化

まず初めに残像を蓄積するための履歴用のテクスチャを用意してます。今回はAとBの2つの履歴テクスチャを用意し、フレームごとに読み込み用と書き込み用を切り替えていく形にします。初回と解像度が変わったときに初期化するようにしています。

```csharp
        private void EnsureHistory(RenderTextureDescriptor desc)
        {
            desc.msaaSamples = 1;
            desc.depthBufferBits = 0;

            if (_historyA != null && _historyA.rt.width == desc.width && _historyA.rt.height == desc.height)
                return;

            RenderingUtils.ReAllocateHandleIfNeeded(ref _historyA, desc, name: "_AfterimageHistoryA");
            RenderingUtils.ReAllocateHandleIfNeeded(ref _historyB, desc, name: "_AfterimageHistoryB");
            _historyValid = false;
        }
```

次にプレイヤーオブジェクトだけを描画するテクスチャバッファと深度バッファも用意します

```csharp
        private void EnsurePlayerTargets(RenderTextureDescriptor desc)
        {
            var colorDesc = desc;
            colorDesc.msaaSamples = 1;
            colorDesc.depthBufferBits = 0;

            var depthDesc = desc;
            depthDesc.msaaSamples = 1;
            depthDesc.graphicsFormat = GraphicsFormat.None;
            depthDesc.depthStencilFormat = GraphicsFormat.D32_SFloat;
            depthDesc.depthBufferBits = 32;

            RenderingUtils.ReAllocateHandleIfNeeded(ref _playerColor, colorDesc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_AfterimagePlayerColor");
            RenderingUtils.ReAllocateHandleIfNeeded(ref _playerDepth, depthDesc, FilterMode.Point, TextureWrapMode.Clamp, name: "_AfterimagePlayerDepth");
        }
```

次に現在のカメラテクスチャと先程初期化した各テクスチャをRenderGraphで使用する `TextureHandle` に変換します。`TextureHandle` についての解説は[こちら](https://blog.sge-coretech.com/entry/2024/09/11/105032#TextureHandle-%E3%81%AF%E4%BB%A5%E4%B8%8B%E3%81%AE%E7%89%B9%E5%BE%B4%E3%81%8C%E3%81%82%E3%82%8A%E3%81%BE%E3%81%99:~:text=frameData.Get%3CUniversalLightData%3E()%3B-,%E8%A7%A3%E8%AA%AC%20*2,-UniversalResourceData%E3%81%AE%E4%B8%AD)が詳しいですが、従来のRTHandleをさらに扱いやすくしたものだと考えてください。

ではなぜ最初に`EnsureHistory`や`EnsurePlayerTargets`でRTHandleを初期化していたかというと、`TextureHadle`はRenderGraphがよしなに管理してくれる分、フレームごとに解放されてしまいます。今回はフレームを跨いでテクスチャを扱いたいので、RTHandleとして確保しておき、RenderGraphで使用する際に都度変換する形になっています。

```csharp
            TextureHandle activeColor = resources.activeColorTexture;
            if (!activeColor.IsValid())
                return;

            TextureHandle historyRead = renderGraph.ImportTexture(_toggle ? _historyA : _historyB);
            TextureHandle historyWrite = renderGraph.ImportTexture(_toggle ? _historyB : _historyA);
            TextureHandle playerColor = renderGraph.ImportTexture(_playerColor);
            TextureHandle playerDepth = renderGraph.ImportTexture(_playerDepth);
```

また何フレームごとに残像を出すのか指定するためにフレームカウンタも用意します。

```csharp
            // 蓄積を行うフレームかどうか（0なら毎フレーム行う）
            bool accumulateThisFrame = _frameCounter <= 0;
            if (accumulateThisFrame)
                _frameCounter = _settings.framesBetweenTrails;
            else
                _frameCounter--;
```

### テクスチャのクリア

履歴テクスチャが初期化されていなければクリアします。

```csharp
            if (!_historyValid)
            {
                AddClearPass(renderGraph, _historyA, "Clear Afterimage History A");
                AddClearPass(renderGraph, _historyB, "Clear Afterimage History B");
                _historyValid = true;
            }

---
---

        private void AddClearPass(RenderGraph renderGraph, RTHandle target, string name)
        {
            TextureHandle imported = renderGraph.ImportTexture(target);
            using var builder = renderGraph.AddRasterRenderPass<ClearPassData>(name, out var passData, profilingSampler);
            passData.Target = imported;
            builder.SetRenderAttachment(passData.Target, 0);
            builder.SetRenderFunc((ClearPassData data, RasterGraphContext ctx) =>
            {
                ctx.cmd.ClearRenderTarget(false, true, Color.clear);
            });
        }

```


### プレイヤーオブジェクトのレンダリング

次にプレイヤーオブジェクトだけを描画するためのレンダラーリストを作成します。`FilteringSettings`でレイヤーマスクを指定して、指定されたタグがついたオブジェクトだけを描画するように設定します。

```csharp
        private RendererListHandle CreatePlayerRendererList(ContextContainer frameData, RenderGraph renderGraph)
        {
            var renderingData = frameData.Get<UniversalRenderingData>();
            var cameraData = frameData.Get<UniversalCameraData>();
            var lightData = frameData.Get<UniversalLightData>();

            var sortFlags = cameraData.defaultOpaqueSortFlags;
            var filterSettings = new FilteringSettings(RenderQueueRange.all, _settings.fallbackLayerMask)
            {
                layerMask = _settings.playerRenderingLayer
            };

            var drawingSettings = RenderingUtils.CreateDrawingSettings(_shaderTags, renderingData, cameraData, lightData, sortFlags);

            var rendererListParams = new RendererListParams(renderingData.cullResults, drawingSettings, filterSettings);
            return renderGraph.CreateRendererList(rendererListParams);
        }
```

RendererListを作成したら描画を実行します。これで現在フレームのプレイヤーオブジェクトだけが描画されたテクスチャが得られます。

```csharp
            var playerRendererList = CreatePlayerRendererList(frameData, renderGraph);
            using (var builder = renderGraph.AddRasterRenderPass<PlayerCapturePassData>("Afterimage Player Capture", out var passData, profilingSampler))
            {
                passData.RendererList = playerRendererList;
                passData.ColorTarget = playerColor;
                passData.DepthTarget = playerDepth;

                builder.UseRendererList(passData.RendererList);
                builder.SetRenderAttachment(passData.ColorTarget, 0);
                builder.SetRenderAttachmentDepth(passData.DepthTarget);

                builder.SetRenderFunc((PlayerCapturePassData data, RasterGraphContext ctx) =>
                {
                    ctx.cmd.ClearRenderTarget(RTClearFlags.All, Color.clear, 1f, 0u);
                    ctx.cmd.DrawRendererList(data.RendererList);
                });
            }
```

### カメラテクスチャのコピー

後ほど合成に使うために、現在のカメラテクスチャをコピーしておきます。

```csharp
            var cameraCopyDesc = activeColor.GetDescriptor(renderGraph);
            cameraCopyDesc.name = "CameraColorCopy";
            TextureHandle cameraCopy = renderGraph.CreateTexture(cameraCopyDesc);
            using (var builder = renderGraph.AddRasterRenderPass<CopyPassData>("Afterimage Camera Copy", out var passData, profilingSampler))
            {
                passData.Source = activeColor;
                passData.Destination = cameraCopy;
                builder.UseTexture(passData.Source);
                builder.SetRenderAttachment(passData.Destination, 0);

                builder.SetRenderFunc((CopyPassData data, RasterGraphContext ctx) =>
                {
                    Blitter.BlitTexture(ctx.cmd, data.Source, Vector2.one, 0f, false);
                });
            }
```

### 残像の蓄積

現在フレームのプレイヤーオブジェクトだけ描画したテクスチャと、前フレームの履歴テクスチャを使って残像を蓄積します。ソースに現在フレームのプレイヤーオブジェクト、`SetTexture`で前フレームの履歴テクスチャをセットして、出力先に次のフレームで使う履歴テクスチャを指定しておきます。Persistenceでどれだけ前の履歴を残すか、Mixで最終的にどれだけ履歴を合成するかを調整しています。

この段階でプレイヤーオブジェクトの残像が蓄積されたテクスチャが得られます。

```csharp
            using (var builder = renderGraph.AddRasterRenderPass<AccumPassData>("Afterimage Accumulate", out var passData, profilingSampler))
            {
                if (accumulateThisFrame)
                {
                    passData.Source = playerColor;
                    passData.HistoryIn = historyRead;
                    passData.HistoryOut = historyWrite;
                    passData.Material = _material;
                    passData.Persistence = _settings.trailPersistence;
                    passData.Mix = _settings.mix;

                    builder.UseTexture(passData.Source);
                    builder.UseTexture(passData.HistoryIn);
                    builder.SetRenderAttachment(passData.HistoryOut, 0);

                    builder.SetRenderFunc((AccumPassData data, RasterGraphContext ctx) =>
                    {
                        data.Material.SetFloat(Persistence, data.Persistence);
                        data.Material.SetFloat(Mix, data.Mix);
                        data.Material.SetTexture(HistoryTex, data.HistoryIn);
                        Blitter.BlitTexture(ctx.cmd, data.Source, Vector2.one, data.Material, 0);
                    });
                }
                else
                {
                    // スキップ時は何もしない
                    builder.SetRenderFunc((AccumPassData data, RasterGraphContext ctx) => { });
                }
            }
```

```hlsl
 Pass
        {
            Name "Afterimage"
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment frag
            #pragma target 4.5

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"

            TEXTURE2D_X(_BlitTexture);    SAMPLER(sampler_BlitTexture);
            TEXTURE2D_X(_HistoryTex);     SAMPLER(sampler_HistoryTex);
            float _Persistence;
            float _Mix;
            float4 _BlitScaleBias;

            struct attributes
            {
                uint vertex_id : SV_VertexID;
            };

            struct varyings
            {
                float4 position_cs : SV_POSITION;
                float2 tex_coord   : TEXCOORD0;
            };
            
            varyings Vert(attributes input)
            {
                varyings output;
                output.position_cs = GetFullScreenTriangleVertexPosition(input.vertex_id);
                output.tex_coord = GetFullScreenTriangleTexCoord(input.vertex_id);
                return output;
            }

            float4 frag(varyings input) : SV_Target
            {
                float2 uv = input.tex_coord * _BlitScaleBias.xy + _BlitScaleBias.zw;
                float4 current = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv);
                float4 history = SAMPLE_TEXTURE2D_X(_HistoryTex, sampler_HistoryTex, uv);

                // 現在フレームと前フレームを比較して明るい方を保持し、をフェードアウトさせる
                float4 accumulated = max(current, history * _Persistence);
                return lerp(current, accumulated, _Mix);
            }
            ENDHLSL
        }
```

### 最終合成

最後に蓄積した残像テクスチャと前もってコピーしておいた現在フレームのカメラテクスチャを合成して最終的な出力を得ます。シェーダーはシンプルにMixで調整した分だけ履歴テクスチャを加算するようにしておきます。また合成が終わったら蓄積に使う履歴テクスチャを切り替えます。

```csharp
            using (var builder = renderGraph.AddRasterRenderPass<CopyPassData>("Afterimage Composite", out var passData, profilingSampler))
            {
                // 蓄積をスキップしたフレームは前の履歴（historyRead）をそのまま合成に使う
                passData.Source = accumulateThisFrame ? historyWrite : historyRead;
                passData.CameraColor = cameraCopy; 
                passData.Destination = activeColor;
                passData.Material = _material;
                passData.Mix = _settings.mix;
                builder.UseTexture(passData.Source);
                builder.UseTexture(passData.CameraColor);
                builder.SetRenderAttachment(passData.Destination, 0);

                builder.SetRenderFunc((CopyPassData data, RasterGraphContext ctx) =>
                {
                    data.Material.SetTexture(HistoryTex, data.Source); // 蓄積した残像テクスチャをセット
                    data.Material.SetFloat(Mix, data.Mix);
                    data.Material.SetTexture(BlitTextureId, data.CameraColor); // 先ほどコピーした現在フレームのカメラテクスチャをセット
                    Blitter.BlitTexture(ctx.cmd, data.CameraColor, Vector2.one, data.Material, 1);
                });
            }

            if (accumulateThisFrame) _toggle = !_toggle; // 次フレームのために読み込みと書き込みテクスチャを切り替え
```

```hlsl
        Pass
        {
            Name "AfterimageComposite"
            Blend One Zero
            ZWrite Off ZTest Always Cull Off
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment frag
            #pragma target 4.5

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"

            TEXTURE2D_X(_BlitTexture);    SAMPLER(sampler_BlitTexture);
            TEXTURE2D_X(_HistoryTex);     SAMPLER(sampler_HistoryTex);
            float _Mix;
            float4 _BlitScaleBias;

            struct attributes
            {
                uint vertex_id : SV_VertexID;
            };

            struct varyings
            {
                float4 position_cs : SV_POSITION;
                float2 tex_coord   : TEXCOORD0;
            };

            varyings Vert(attributes input)
            {
                varyings output;
                output.position_cs = GetFullScreenTriangleVertexPosition(input.vertex_id);
                output.tex_coord = GetFullScreenTriangleTexCoord(input.vertex_id);
                return output;
            }

            float4 frag(varyings input) : SV_Target
            {
                float2 uv = input.tex_coord * _BlitScaleBias.xy + _BlitScaleBias.zw;
                float4 baseColor = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv);
                float4 history = SAMPLE_TEXTURE2D_X(_HistoryTex, sampler_HistoryTex, uv);
                return baseColor + history * _Mix;
            }
            ENDHLSL
        }
```

これで残像エフェクトは動くようになります。

![](https://storage.googleapis.com/zenn-user-upload/faff0f2800e7-20251128.gif)

## 2. 色相をシフトする

さて次は残像の色相がシフトしていくようにしていきます。

色相がどれぐらいの速度で変化するかを指定するパラメータを追加します。

```csharp

    private static readonly int HueShiftSpeed = Shader.PropertyToID("_HueShiftSpeed");

    [System.Serializable]
    public class Settings
    {
        --省略--
        [Range(-5f, 5f)] public float hueShiftPerSecond = 0.5f;                  // 残像の色相を毎秒どれだけ回すか
    }
```

前フレームの履歴テクスチャとプレイヤーオブジェクトの描画テクスチャを合成する際に色相をシフトするようにします。色相変換はHSL空間で行います。

```csharp
                    passData.HueShiftPerSecond = _settings.hueShiftPerSecond;

                    builder.UseTexture(passData.Source);
                    builder.UseTexture(passData.HistoryIn);
                    builder.SetRenderAttachment(passData.HistoryOut, 0);

                    builder.SetRenderFunc((AccumPassData data, RasterGraphContext ctx) =>
                    {
                        data.Material.SetFloat(Persistence, data.Persistence);
                        data.Material.SetFloat(Mix, data.Mix);
                        data.Material.SetFloat(HueShiftSpeed, data.HueShiftPerSecond); 
                        data.Material.SetTexture(HistoryTex, data.HistoryIn);
                        Blitter.BlitTexture(ctx.cmd, data.Source, Vector2.one, data.Material, 0);
                    });
```

```hlsl
            float3 rgb_to_hsv(float3 rgb)
            {
                float4 K = float4(0.0, -1.0 / 3.0, 2.0 / 3.0, -1.0);
                float4 p = lerp(float4(rgb.bg, K.wz), float4(rgb.gb, K.xy), step(rgb.b, rgb.g));
                float4 q = lerp(float4(p.xyw, rgb.r), float4(rgb.r, p.yzx), step(p.x, rgb.r));

                float d = q.x - min(q.w, q.y);
                float e = 1.0e-10;
                float3 hsv;
                hsv.x = abs(q.z + (q.w - q.y) / (6.0 * d + e));
                hsv.y = d / (q.x + e);
                hsv.z = q.x;
                return hsv;
            }

            float3 hsv_to_rgb(float3 hsv)
            {
                float4 K = float4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
                float3 p = abs(frac(hsv.x + K.xyz) * 6.0 - K.www);
                return hsv.z * lerp(K.xxx, saturate(p - K.xxx), hsv.y);
            }

            varyings Vert(attributes input)
            {
                varyings output;
                output.position_cs = GetFullScreenTriangleVertexPosition(input.vertex_id);
                output.tex_coord = GetFullScreenTriangleTexCoord(input.vertex_id);
                return output;
            }

            float4 frag(varyings input) : SV_Target
            {
                float2 uv = input.tex_coord * _BlitScaleBias.xy + _BlitScaleBias.zw;
                float4 current = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv);
                float4 history = SAMPLE_TEXTURE2D_X(_HistoryTex, sampler_HistoryTex, uv);

                float hueShift = _Time.y * _HueShiftSpeed;
                float3 historyHsv = rgb_to_hsv(history.rgb);
                historyHsv.x = frac(historyHsv.x + hueShift);
                float3 historyShifted = hsv_to_rgb(historyHsv);

                float4 accumulated = max(current, float4(historyShifted, history.a) * _Persistence);
                return lerp(current, accumulated, _Mix);
            }
            ENDHLSL
        }
```

これで色相がシフトしていくようになります。

![](https://storage.googleapis.com/zenn-user-upload/2bfa9139f996-20251128.gif)

## 3. ステンシルバッファを使って本体と重ならないように合成

これで完成でもいいんですが、今回はRenderGraphでのステンシルバッファの使い方を学びたかったので、ステンシルバッファを使って本体と残像が重ならないようにしてみます。

### ステンシルバッファへの書き込み

まずPassのコンストラクタで`RenderStateBlock`を設定します。
```csharp
    private class AfterimagePass : ScriptableRenderPass
    {
        ~~省略~~
        private readonly RenderStateBlock _stencilStateBlock;

        ~~省略~~
        
        public AfterimagePass(Settings settings)
        {
            _settings = settings;
            renderPassEvent = RenderPassEvent.AfterRenderingPostProcessing;

            // プレイヤー描画時にステンシルを書き込む設定（Ref=1, Always, Replace）
            var stencilState = new StencilState(compareFunction: CompareFunction.Always, passOperation: StencilOp.Replace);
            _stencilStateBlock = new RenderStateBlock(RenderStateMask.Stencil)
            {
                stencilReference = 1,
                stencilState = stencilState
            };
        }
```

次にプレイヤーオブジェクト描画用のテクスチャを初期化する際にステンシルバッファの設定も行います。


```csharp
        private void EnsurePlayerTargets(RenderTextureDescriptor desc)
        {
            var colorDesc = desc;
            colorDesc.depthBufferBits = 0;

            var depthDesc = desc;
            depthDesc.graphicsFormat = GraphicsFormat.None;
            depthDesc.depthStencilFormat = GraphicsFormat.D24_UNorm_S8_UInt;
            depthDesc.depthBufferBits = 24;

            RenderingUtils.ReAllocateHandleIfNeeded(ref _playerColor, colorDesc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_AfterimagePlayerColor");
            RenderingUtils.ReAllocateHandleIfNeeded(ref _playerDepth, depthDesc, FilterMode.Point, TextureWrapMode.Clamp, name: "_AfterimagePlayerDepth");
        }
```

プレイヤーオブジェクトのRendererListを描画する際にコンストラクタで用意した `_stencilStateBlock`を適用します。

```csharp
        private RendererListHandle CreatePlayerRendererList(ContextContainer frameData, RenderGraph renderGraph)
        {
            var renderingData = frameData.Get<UniversalRenderingData>();
            var cameraData = frameData.Get<UniversalCameraData>();
            var lightData = frameData.Get<UniversalLightData>();

            var sortFlags = cameraData.defaultOpaqueSortFlags;

            RenderingUtils.CreateDrawingSettings(_shaderTags, renderingData, cameraData, lightData, sortFlags);

            var rendererListParams = new RendererListDesc(_shaderTags.ToArray(), renderingData.cullResults, cameraData.camera)
            {
                layerMask =  _settings.playerRenderingLayer,
                stateBlock =  _stencilStateBlock,
                renderQueueRange = RenderQueueRange.opaque,
                sortingCriteria = sortFlags
            };
            return renderGraph.CreateRendererList(rendererListParams);
        }
```

### ステンシルバッファの読み込み

残像テクスチャとカメラテクスチャを合成する際にステンシルバッファを参照して、プレイヤーオブジェクトが描かれている部分はスキップするようにします。

```csharp
            using (var builder = renderGraph.AddRasterRenderPass<CopyPassData>("Afterimage Composite", out var passData, profilingSampler))
            {
                passData.Source = accumulateThisFrame ? historyWrite : historyRead;
                passData.CameraColor = cameraCopy;
                passData.Destination = activeColor;
                passData.Material = _material;
                passData.Mix = _settings.mix;
                passData.StencilDepth = playerDepth;
                builder.UseTexture(passData.Source);
                builder.UseTexture(passData.CameraColor);
                builder.SetRenderAttachment(passData.Destination, 0);
                builder.SetRenderAttachmentDepth(passData.StencilDepth, AccessFlags.Read);

                builder.SetRenderFunc((CopyPassData data, RasterGraphContext ctx) =>
                {
                    data.Material.SetTexture(HistoryTex, data.Source);
                    data.Material.SetFloat(Mix, data.Mix);
                    data.Material.SetTexture(BlitTextureId, data.CameraColor);
                    Blitter.BlitTexture(ctx.cmd, data.CameraColor, Vector2.one, data.Material, 1);
                });
            }
```

```hlsl
   Pass
        {
            Name "AfterimageComposite"
            Blend One Zero
            ZWrite Off ZTest Always Cull Off

            Stencil
            {
                Ref 1
                Comp NotEqual // プレイヤーが描かれている場所（Ref=1）はスキップ
                Pass Keep
            }

            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment frag
            #pragma target 4.5
```

これで本体と重ならないように残像が合成されるようになります。

![](https://storage.googleapis.com/zenn-user-upload/26c5301fcb5b-20251128.gif)


最終的なコードは以下の通りです

:::details AfterimageRenderFeature.cs
```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Experimental.Rendering;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;
using UnityEngine.Rendering.RenderGraphModule;
using UnityEngine.Rendering.RendererUtils;

public class AfterimageRenderFeature : ScriptableRendererFeature
{
    private static readonly int Persistence = Shader.PropertyToID("_Persistence");
    private static readonly int Mix = Shader.PropertyToID("_Mix");
    private static readonly int HistoryTex = Shader.PropertyToID("_HistoryTex");
    private static readonly int BlitTextureId = Shader.PropertyToID("_BlitTexture");
    private static readonly int HueShiftSpeed = Shader.PropertyToID("_HueShiftSpeed");

    [System.Serializable]
    public class Settings
    {
        [Range(0f, 1f)] public float trailPersistence = 0.9f;
        [Range(0f, 1f)] public float mix = 1f;
        [Range(0, 10)] public int framesBetweenTrails;
        public Shader shader;
        public LayerMask playerRenderingLayer;
        [Range(-5f, 5f)] public float hueShiftPerSecond = 0.5f;
    }

    [SerializeField] private Settings settings = new();

    private AfterimagePass _pass;

    public override void Create()
    {
        if (settings.shader == null)
            settings.shader = Shader.Find("Hidden/Afterimage/Accumulation");

        _pass = new AfterimagePass(settings);
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (!renderingData.cameraData.postProcessEnabled || settings.shader == null)
            return;

        renderer.EnqueuePass(_pass);
    }

    protected override void Dispose(bool disposing)
    {
        _pass?.Dispose();
        base.Dispose(disposing);
    }

    private class AfterimagePass : ScriptableRenderPass
    {
        private readonly Settings _settings;
        private Material _material;
        private RTHandle _historyA, _historyB;
        private bool _toggle;
        private bool _historyValid;
        private RTHandle _playerColor;
        private RTHandle _playerDepth;
        private int _frameCounter; 
        private readonly RenderStateBlock _stencilStateBlock;
        private readonly List<ShaderTagId> _shaderTags = new()
        {
            new ShaderTagId("UniversalForwardOnly"),
            new ShaderTagId("UniversalForward"),
            new ShaderTagId("SRPDefaultUnlit"),
            new ShaderTagId("LightweightForward")
        };

        public AfterimagePass(Settings settings)
        {
            _settings = settings;
            renderPassEvent = RenderPassEvent.AfterRenderingPostProcessing;

            // プレイヤー描画時にステンシルを書き込む設定（Ref=1, Always, Replace）
            var stencilState = new StencilState(compareFunction: CompareFunction.Always, passOperation: StencilOp.Replace);
            _stencilStateBlock = new RenderStateBlock(RenderStateMask.Stencil)
            {
                stencilReference = 1,
                stencilState = stencilState
            };
        }

        public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData)
        {
            if (_settings.shader == null)
                return;

            if (_material == null)
                _material = CoreUtils.CreateEngineMaterial(_settings.shader);

            UniversalCameraData cameraData = frameData.Get<UniversalCameraData>();
            UniversalResourceData resources = frameData.Get<UniversalResourceData>();

            var desc = cameraData.cameraTargetDescriptor;
            EnsureHistory(desc);       // 履歴用の A/B バッファをカメラ解像度で確保
            EnsurePlayerTargets(desc); // プレイヤーだけを描くためのカラー/深度バッファを確保

            TextureHandle activeColor = resources.activeColorTexture;
            if (!activeColor.IsValid())
                return;

            TextureHandle historyRead = renderGraph.ImportTexture(_toggle ? _historyA : _historyB);
            TextureHandle historyWrite = renderGraph.ImportTexture(_toggle ? _historyB : _historyA);
            TextureHandle playerColor = renderGraph.ImportTexture(_playerColor);
            TextureHandle playerDepth = renderGraph.ImportTexture(_playerDepth);

            bool accumulateThisFrame = _frameCounter <= 0;
            if (accumulateThisFrame)
                _frameCounter = _settings.framesBetweenTrails;
            else
                _frameCounter--;

            if (!_historyValid)
            {
                AddClearPass(renderGraph, _historyA, "Clear Afterimage History A");
                AddClearPass(renderGraph, _historyB, "Clear Afterimage History B");
                _historyValid = true;
            }

            var playerRendererList = CreatePlayerRendererList(frameData, renderGraph); // 残像対象だけを描くリスト
            using (var builder = renderGraph.AddRasterRenderPass<PlayerCapturePassData>("Afterimage Player Capture", out var passData, profilingSampler))
            {
                passData.RendererList = playerRendererList;
                passData.ColorTarget = playerColor;
                passData.DepthTarget = playerDepth;

                builder.UseRendererList(passData.RendererList);
                builder.SetRenderAttachment(passData.ColorTarget, 0);
                builder.SetRenderAttachmentDepth(passData.DepthTarget, AccessFlags.Write);

                builder.SetRenderFunc((PlayerCapturePassData data, RasterGraphContext ctx) =>
                {
                    ctx.cmd.ClearRenderTarget(RTClearFlags.All, Color.clear, 1f, 0u);
                    ctx.cmd.DrawRendererList(data.RendererList);
                });
            }

            var cameraCopyDesc = activeColor.GetDescriptor(renderGraph);
            cameraCopyDesc.name = "CameraColorCopy";
            TextureHandle cameraCopy = renderGraph.CreateTexture(cameraCopyDesc);
            using (var builder = renderGraph.AddRasterRenderPass<CopyPassData>("Afterimage Camera Copy", out var passData, profilingSampler))
            {
                passData.Source = activeColor;
                passData.Destination = cameraCopy;
                builder.UseTexture(passData.Source);
                builder.SetRenderAttachment(passData.Destination, 0);

                builder.SetRenderFunc((CopyPassData data, RasterGraphContext ctx) =>
                {
                    Blitter.BlitTexture(ctx.cmd, data.Source, Vector2.one, 0f, false);
                });
            }

            using (var builder = renderGraph.AddRasterRenderPass<AccumPassData>("Afterimage Accumulate", out var passData, profilingSampler))
            {
                if (accumulateThisFrame)
                {
                    passData.Source = playerColor;
                    passData.HistoryIn = historyRead;
                    passData.HistoryOut = historyWrite;
                    passData.Material = _material;
                    passData.Persistence = _settings.trailPersistence;
                    passData.Mix = _settings.mix;
                    passData.HueShiftPerSecond = _settings.hueShiftPerSecond;

                    builder.UseTexture(passData.Source);
                    builder.UseTexture(passData.HistoryIn);
                    builder.SetRenderAttachment(passData.HistoryOut, 0);

                    builder.SetRenderFunc((AccumPassData data, RasterGraphContext ctx) =>
                    {
                        data.Material.SetFloat(Persistence, data.Persistence);
                        data.Material.SetFloat(Mix, data.Mix);
                        data.Material.SetFloat(HueShiftSpeed, data.HueShiftPerSecond);
                        data.Material.SetTexture(HistoryTex, data.HistoryIn);
                        Blitter.BlitTexture(ctx.cmd, data.Source, Vector2.one, data.Material, 0);
                    });
                }
                else
                {
                    // スキップ時は何もしない
                    builder.SetRenderFunc((AccumPassData _, RasterGraphContext _) => { });
                }
            }

            using (var builder = renderGraph.AddRasterRenderPass<CopyPassData>("Afterimage Composite", out var passData, profilingSampler))
            {
                passData.Source = accumulateThisFrame ? historyWrite : historyRead;
                passData.CameraColor = cameraCopy;
                passData.Destination = activeColor;
                passData.Material = _material;
                passData.Mix = _settings.mix;
                passData.StencilDepth = playerDepth;
                builder.UseTexture(passData.Source);
                builder.UseTexture(passData.CameraColor);
                builder.SetRenderAttachment(passData.Destination, 0);
                builder.SetRenderAttachmentDepth(passData.StencilDepth, AccessFlags.Read);

                builder.SetRenderFunc((CopyPassData data, RasterGraphContext ctx) =>
                {
                    data.Material.SetTexture(HistoryTex, data.Source);
                    data.Material.SetFloat(Mix, data.Mix);
                    data.Material.SetTexture(BlitTextureId, data.CameraColor);
                    Blitter.BlitTexture(ctx.cmd, data.CameraColor, Vector2.one, data.Material, 1);
                });
            }


            if (accumulateThisFrame)
                _toggle = !_toggle;
        }

        public void Dispose()
        {
            RTHandles.Release(_historyA);
            RTHandles.Release(_historyB);
            CoreUtils.Destroy(_material);
            RTHandles.Release(_playerColor);
            RTHandles.Release(_playerDepth);
        }

        private void EnsureHistory(RenderTextureDescriptor desc)
        {
            desc.msaaSamples = 1;
            desc.depthBufferBits = 0;

            if (_historyA != null && _historyA.rt.width == desc.width && _historyA.rt.height == desc.height)
                return;

            RenderingUtils.ReAllocateHandleIfNeeded(ref _historyA, desc, name: "_AfterimageHistoryA");
            RenderingUtils.ReAllocateHandleIfNeeded(ref _historyB, desc, name: "_AfterimageHistoryB");
            _historyValid = false;
        }

        private void EnsurePlayerTargets(RenderTextureDescriptor desc)
        {
            var colorDesc = desc;
            colorDesc.depthBufferBits = 0;

            var depthDesc = desc;
            depthDesc.graphicsFormat = GraphicsFormat.None;
            depthDesc.depthStencilFormat = GraphicsFormat.D24_UNorm_S8_UInt;
            depthDesc.depthBufferBits = 24;

            RenderingUtils.ReAllocateHandleIfNeeded(ref _playerColor, colorDesc, FilterMode.Bilinear, TextureWrapMode.Clamp, name: "_AfterimagePlayerColor");
            RenderingUtils.ReAllocateHandleIfNeeded(ref _playerDepth, depthDesc, FilterMode.Point, TextureWrapMode.Clamp, name: "_AfterimagePlayerDepth");
        }

        private void AddClearPass(RenderGraph renderGraph, RTHandle target, string name)
        {
            TextureHandle imported = renderGraph.ImportTexture(target);
            using var builder = renderGraph.AddRasterRenderPass<ClearPassData>(name, out var passData, profilingSampler);
            passData.Target = imported;
            builder.SetRenderAttachment(passData.Target, 0);
            builder.SetRenderFunc((ClearPassData _, RasterGraphContext ctx) =>
            {
                ctx.cmd.ClearRenderTarget(false, true, Color.clear);
            });
        }

        private class AccumPassData
        {
            internal TextureHandle Source;
            internal TextureHandle HistoryIn;
            internal TextureHandle HistoryOut;
            internal Material Material;
            internal float Persistence;
            internal float Mix;
            internal float HueShiftPerSecond;
        }

        private class CopyPassData
        {
            internal TextureHandle Source;
            internal TextureHandle CameraColor;
            internal TextureHandle Destination;
            internal Material Material;
            internal float Mix;
            internal TextureHandle StencilDepth;
        }

        private class ClearPassData
        {
            internal TextureHandle Target;
        }

        private class PlayerCapturePassData
        {
            internal RendererListHandle RendererList;
            internal TextureHandle ColorTarget;
            internal TextureHandle DepthTarget;
        }

        private RendererListHandle CreatePlayerRendererList(ContextContainer frameData, RenderGraph renderGraph)
        {
            var renderingData = frameData.Get<UniversalRenderingData>();
            var cameraData = frameData.Get<UniversalCameraData>();
            var lightData = frameData.Get<UniversalLightData>();

            var sortFlags = cameraData.defaultOpaqueSortFlags;

            RenderingUtils.CreateDrawingSettings(_shaderTags, renderingData, cameraData, lightData, sortFlags);

            var rendererListParams = new RendererListDesc(_shaderTags.ToArray(), renderingData.cullResults, cameraData.camera)
            {
                layerMask =  _settings.playerRenderingLayer,
                stateBlock =  _stencilStateBlock,
                renderQueueRange = RenderQueueRange.opaque,
                sortingCriteria = sortFlags
            };
            return renderGraph.CreateRendererList(rendererListParams);
        }
    }
}
```
:::

:::details AfterimageAccumulation.shader
```hlsl
Shader "Hidden/Afterimage/Accumulation"
{
    Properties { }
    SubShader
    {
        Tags { "RenderPipeline"="UniversalPipeline" }
        ZWrite Off ZTest Always Cull Off Blend One Zero

        Pass
        {
            Name "Afterimage"
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment frag
            #pragma target 4.5

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"

            TEXTURE2D_X(_BlitTexture);    SAMPLER(sampler_BlitTexture);
            TEXTURE2D_X(_HistoryTex);     SAMPLER(sampler_HistoryTex);
            float _Persistence;
            float _Mix;
            float _HueShiftSpeed;
            float4 _BlitScaleBias;

            struct attributes
            {
                uint vertex_id : SV_VertexID;
            };

            struct varyings
            {
                float4 position_cs : SV_POSITION;
                float2 tex_coord   : TEXCOORD0;
            };

            float3 rgb_to_hsv(float3 rgb)
            {
                float4 K = float4(0.0, -1.0 / 3.0, 2.0 / 3.0, -1.0);
                float4 p = lerp(float4(rgb.bg, K.wz), float4(rgb.gb, K.xy), step(rgb.b, rgb.g));
                float4 q = lerp(float4(p.xyw, rgb.r), float4(rgb.r, p.yzx), step(p.x, rgb.r));

                float d = q.x - min(q.w, q.y);
                float e = 1.0e-10;
                float3 hsv;
                hsv.x = abs(q.z + (q.w - q.y) / (6.0 * d + e));
                hsv.y = d / (q.x + e);
                hsv.z = q.x;
                return hsv;
            }

            float3 hsv_to_rgb(float3 hsv)
            {
                float4 K = float4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
                float3 p = abs(frac(hsv.x + K.xyz) * 6.0 - K.www);
                return hsv.z * lerp(K.xxx, saturate(p - K.xxx), hsv.y);
            }

            varyings Vert(attributes input)
            {
                varyings output;
                output.position_cs = GetFullScreenTriangleVertexPosition(input.vertex_id);
                output.tex_coord = GetFullScreenTriangleTexCoord(input.vertex_id);
                return output;
            }

            float4 frag(varyings input) : SV_Target
            {
                float2 uv = input.tex_coord * _BlitScaleBias.xy + _BlitScaleBias.zw;
                float4 current = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv);
                float4 history = SAMPLE_TEXTURE2D_X(_HistoryTex, sampler_HistoryTex, uv);

                float hueShift = _Time.y * _HueShiftSpeed;
                float3 historyHsv = rgb_to_hsv(history.rgb);
                historyHsv.x = frac(historyHsv.x + hueShift);
                float3 historyShifted = hsv_to_rgb(historyHsv);

                float4 accumulated = max(current, float4(historyShifted, history.a) * _Persistence);
                return lerp(current, accumulated, _Mix);
            }
            ENDHLSL
        }

        Pass
        {
            Name "AfterimageComposite"
            Blend One Zero
            ZWrite Off ZTest Always Cull Off
            Stencil
            {
                Ref 1
                Comp NotEqual // プレイヤーが描かれている場所（Ref=1）はスキップ
                Pass Keep
            }
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment frag
            #pragma target 4.5

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"

            TEXTURE2D_X(_BlitTexture);    SAMPLER(sampler_BlitTexture);
            TEXTURE2D_X(_HistoryTex);     SAMPLER(sampler_HistoryTex);
            float _Mix;
            float4 _BlitScaleBias;

            struct attributes
            {
                uint vertex_id : SV_VertexID;
            };

            struct varyings
            {
                float4 position_cs : SV_POSITION;
                float2 tex_coord   : TEXCOORD0;
            };

            varyings Vert(attributes input)
            {
                varyings output;
                output.position_cs = GetFullScreenTriangleVertexPosition(input.vertex_id);
                output.tex_coord = GetFullScreenTriangleTexCoord(input.vertex_id);
                return output;
            }

            float4 frag(varyings input) : SV_Target
            {
                float2 uv = input.tex_coord * _BlitScaleBias.xy + _BlitScaleBias.zw;
                float4 baseColor = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv);
                float4 history = SAMPLE_TEXTURE2D_X(_HistoryTex, sampler_HistoryTex, uv);
                return baseColor + history * _Mix;
            }
            ENDHLSL
        }
    }
}
```
:::
## おわりに

今回はRenderGraphの練習でシンプルな残像エフェクトを実装してみました。以前のURPでもブルーム等のポストエフェクトを実装したことはあるんですが、RenderGraphのほうがシンプルに実装できることがよく分かりました。また今回触れていませんがRenderGraph Viewerを使うとレンダーパスの依存関係や各パスで使用されているリソースを可視化できるので実装やデバッグも楽になります。

## 参考にさせていただいた資料

- [Unity6からRenderGraphを使いこなそう！応用実装編1「RenderPassの構築」](https://blog.sge-coretech.com/entry/2024/09/11/105032)
- [Unity、URP、RenderFeature を用いたマルチパスレンダリング【 解説編 】](https://engineering.cocone.io/2025/10/16/unity-urp-multipass-simple-implementation-vol2/)