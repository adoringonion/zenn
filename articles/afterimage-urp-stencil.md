---
title: "Earth, Wind & FireっぽいエフェクトをRenderGraphで実装する"
emoji: "🌈"
type: "tech"
topics: ["unity", "urp", "shader", "rendergraph"]
published: false
---

この記事は [Anthrotech Advent Calendar 2025](https://adventar.org/calendars/11972) の1日目の記事です。Antorotechについては [こちら](https://anthrotech.dev/) をご覧ください。

## はじめに

RenderGraphはUnity6からURPに導入され、6.4からはデフォルトになって従来の書き方ができなくなるので今のうちに慣れておこうと思い、エフェクトを作ってみます。

今回作るのはEarth Wind and FireのLet's Grooveっぽいエフェクトです。
https://youtu.be/Lrle0x_DHBM?si=gDZG7Sx1TySWDIH4&t=60
大体1分あたりに出てくる、人物にカラフルな残像がつくエフェクトをRenderGraphで実装してきます。

---

## 方針

そもそも残像エフェクトってどう作るねんって話ですが、今回のエフェクトの要件は

- プレイヤーの動きに合わせて残像がつく
- 残像は時間経過でフェードアウトする
- 残像は色相が時間で変化する

という感じなので、以下の画像のような流れで実装します。





## 1. 残像を作る

まずは残像を作っていきましょう。

:::details AfterimageFeature.cs 全体コード

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Experimental.Rendering;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;
using UnityEngine.Rendering.RenderGraphModule;

public class AfterimageRenderFeature : ScriptableRendererFeature
{
    private static readonly int Persistence = Shader.PropertyToID("_Persistence");
    private static readonly int Mix = Shader.PropertyToID("_Mix");
    private static readonly int HistoryTex = Shader.PropertyToID("_HistoryTex");
    private static readonly int BlitTextureId = Shader.PropertyToID("_BlitTexture");

    [System.Serializable]
    public class Settings
    {
        [Range(0f, 1f)] public float trailPersistence = 0.9f; // higher = longer trail
        [Range(0f, 1f)] public float mix = 1f;                 // blend back into camera target
        public Shader shader;
        public LayerMask playerRenderingLayer; // player-only rendering layer
        public LayerMask fallbackLayerMask = ~0;                                 // only used if Rendering Layer is not set
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
            EnsureHistory(desc);
            EnsurePlayerTargets(desc);

            TextureHandle activeColor = resources.activeColorTexture;
            if (!activeColor.IsValid())
                return;

            TextureHandle historyRead = renderGraph.ImportTexture(_toggle ? _historyA : _historyB);
            TextureHandle historyWrite = renderGraph.ImportTexture(_toggle ? _historyB : _historyA);
            TextureHandle playerColor = renderGraph.ImportTexture(_playerColor);
            TextureHandle playerDepth = renderGraph.ImportTexture(_playerDepth);

            if (!_historyValid)
            {
                AddClearPass(renderGraph, _historyA, "Clear Afterimage History A");
                AddClearPass(renderGraph, _historyB, "Clear Afterimage History B");
                _historyValid = true;
            }

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

            using (var builder = renderGraph.AddRasterRenderPass<CopyPassData>("Afterimage Composite", out var passData, profilingSampler))
            {
                passData.Source = historyWrite;
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


---

## 2. 色相をシフトする


## 3. 
**狙い:** 履歴側の色相を回して、残像をカラフルに。

- Pass0 内で HSV 変換し、`_HueShiftSpeed` × `_Time.y` で H を回転。
- `historyShifted` を蓄積計算に用いるだけなので、基本のロジックは変えずに色味だけ動かせる。
- パラメータ: `Settings.hueShiftPerSecond`

---

## 4. 蓄積間隔を調整できるようにする
**狙い:** 毎フレーム更新が重い/強すぎるときに間引きたい。

- `Settings.framesBetweenTrails` を追加（0=毎フレーム）。
- フレームカウンタで Accumulate パスをスキップし、スキップ時は履歴を更新せず、前フレーム履歴をそのまま合成に使う。
- ping-pong は「蓄積したフレームだけ」進める。

---

## 5. ステンシルで本体と重ならないように合成
**狙い:** カメラカラーに残像を足す際、プレイヤー本体の画素では残像を描かない。

- 合成パス（Shader Pass1）にステンシルテストを追加。
- プレイヤー描画で Ref=1 を書いたので、`Comp NotEqual` で「Ref=1 以外」にのみ残像を描く。
- 深度添付として `_AfterimagePlayerDepth` をパスにセットしてステンシルを参照。

Shader Pass1 抜粋:
```hlsl
Pass {
    Name "AfterimageComposite"
    Stencil {
        Ref 1
        Comp NotEqual // プレイヤー領域をスキップ
        Pass Keep
    }
    float4 frag(...) : SV_Target {
        float4 baseColor = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_BlitTexture, uv);
        float4 history   = SAMPLE_TEXTURE2D_X(_HistoryTex, sampler_HistoryTex, uv);
        return baseColor + history * _Mix;
    }
}
```

---

## パラメータ一覧（Settings）
- `trailPersistence`: 履歴の残り具合（1 に近いほど長く残る）
- `mix`: 残像の合成量（大きいほど強い）
- `framesBetweenTrails`: 何フレームおきに蓄積するか（0=毎フレーム）
- `hueShiftPerSecond`: 履歴の色相回転速度
- `playerRenderingLayer`: 残像対象のレイヤー
- `fallbackLayerMask`: レイヤー未指定時のフォールバック

---

## まとめ
1. プレイヤーだけを専用 RT に描き、ステンシルを仕込む。
2. 専用 RT と履歴を使って残像を蓄積し、色相を回す。
3. 必要に応じて蓄積間隔を間引く。
4. ステンシル NotEqual で本体を避けて合成し、白飛びや二重描きを防ぐ。
