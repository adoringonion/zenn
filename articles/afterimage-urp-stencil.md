---
title: "Earth, Wind & FireっぽいエフェクトをRenderGraphで実装する"
emoji: "🌈"
type: "tech"
topics: ["Unity"]
published: false
---

この記事は [Anthrotech Advent Calendar 2025](https://adventar.org/calendars/11972) の1日目の記事です。Antorotechについては [こちら](https://anthrotech.dev/) をご覧ください。

## はじめに

RenderGraphはUnity6からURPに導入され、6.4からはデフォルトになって従来の書き方ができなくなるので今のうちに慣れておこうと思い、エフェクトを作ってみます。

今回作るのはEarth Wind and FireのLet's Grooveっぽいエフェクトです。
https://youtu.be/Lrle0x_DHBM?si=gDZG7Sx1TySWDIH4&t=60
大体1分あたりに出てくる、人物にカラフルな残像がつくエフェクトをRenderGraphで実装してきます。


## 方針

そもそも残像エフェクトってどう作るねんって話ですが、今回のエフェクトの要件は

- プレイヤーの動きに合わせて残像がつく
- 残像は時間経過でフェードアウトする
- 残像は色相が時間で変化する

という感じなので、以下の画像のような流れで実装します。



## 1. 残像を作る

まずは残像を作っていきましょう。

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

## 2. 色相をシフトする

:::details AfterimageFeature.cs (変更箇所のみ)
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
        [Range(0f, 1f)] public float trailPersistence = 0.9f; // 1 に近いほど履歴が長く残る
        [Range(0f, 1f)] public float mix = 1f;                 // どれだけ履歴を最終画像に戻すか
        [Range(0, 10)] public int framesBetweenTrails = 0;     // 何フレームおきに蓄積するか（0なら毎フレーム）
        public Shader shader;
        public LayerMask playerRenderingLayer; // 残像を描く対象レイヤー
        public LayerMask fallbackLayerMask = ~0;                                 // レイヤー未設定時のフォールバック
        [Range(-5f, 5f)] public float hueShiftPerSecond = 0.5f;                  // 残像の色相を毎秒どれだけ回すか
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
            internal float HueShiftPerSecond;
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
```
:::

:::details AfterimageAccumulation.shader (変更箇所のみ)

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

                // Rotate hue over time before blending history back in.
                float hueShift = _Time.y * _HueShiftSpeed;
                float3 historyHsv = rgb_to_hsv(history.rgb);
                historyHsv.x = frac(historyHsv.x + hueShift);
                float3 historyShifted = hsv_to_rgb(historyHsv);

                // Keep brighter pixels from current frame and fade the previous history.
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

## 3. ステンシルバッファを使って本体と重ならないように合成

現在の実装だとプレイヤーオブジェクトに残像エフェクトが重なり、プレイヤーオブジェクト自体が少し明るくなります。プレイヤーオブジェクトに影響せず残像を表示するためにステンシルバッファを使います

:::details AfterimageFeature.cs (変更箇所のみ)
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
        [Range(0f, 1f)] public float trailPersistence = 0.9f; // 1 に近いほど履歴が長く残る
        [Range(0f, 1f)] public float mix = 1f;                 // どれだけ履歴を最終画像に戻すか
        [Range(0, 10)] public int framesBetweenTrails = 0;     // 何フレームおきに蓄積するか（0なら毎フレーム）
        public Shader shader;
        public LayerMask playerRenderingLayer; // 残像を描く対象レイヤー
        public LayerMask fallbackLayerMask = ~0;                                 // レイヤー未設定時のフォールバック
        [Range(-5f, 5f)] public float hueShiftPerSecond = 0.5f;                  // 残像の色相を毎秒どれだけ回すか
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
            var stencilState = new StencilState(true, 0xFF, 0xFF, CompareFunction.Always, StencilOp.Replace, StencilOp.Replace, StencilOp.Replace);
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
                    // スキップ時は何もしない（RenderGraph のお作法として空関数をセット）
                    builder.SetRenderFunc((AccumPassData data, RasterGraphContext ctx) => { });
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
            depthDesc.depthStencilFormat = GraphicsFormat.D24_UNorm_S8_UInt; // ステンシルを持つ深度
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

:::details AfterimageAccumulation.shader (変更箇所のみ)
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
:::

## まとめ
1. プレイヤーだけを専用 RT に描き、ステンシルを仕込む。
2. 専用 RT と履歴を使って残像を蓄積し、色相を回す。
3. 必要に応じて蓄積間隔を間引く。
4. ステンシル NotEqual で本体を避けて合成し、白飛びや二重描きを防ぐ。
