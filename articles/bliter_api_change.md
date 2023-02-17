---
title: "URP14でポストエフェクトのかけ方が変わった"
emoji: "🪞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [unity,urp]
published: true
---

# はじめに

今まではポストエフェクトをつける際に`cmd.Blit`でかけていたかと思います。
これがURP14からは`Bliter`APIを使用するようになったので、それについて調べてみます。

# 環境

Unity 2022.2.2f1
Universal RP 14.0.4

# What's new in URP14

URP14のWhat's newを見ているとBliterAPIのことについて記載されています。

> Full screen draws in URP now use the SRP Core Bliter API
All the calls to cmd.Blit method are replaced with the Blitter API. This ensures a correct and consistent way to perform full screen draws.
>
> In the current URP version, using cmd.Blit might implicitly enable or disable XR shader keywords, which breaks XR SPI rendering.
>
> Refer to the Perform a full screen blit in URP page to read how to use the Blitter API.

要約すると、以下です。

* URPのポストエフェクトでは、`SRP Core Bliter API`というものを使用するようになった
* 今までの`cmd.Blit`メソッドは`Bliter API`に置き換えられるが、XR環境では壊れるかもしれない

https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/whats-new/urp-whats-new.html

# BliterAPIを使用して、ポストエフェクトをかける

以下にポストエフェクトをかける例が記載されています。
これを参考にポストエフェクトをかけてみます。

https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/renderer-features/how-to-fullscreen-blit.html

:::details FullScreenPassRendererFeatureの中身

## RendererFeature

```cs:ColorBlitRendererFeature.cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

internal class ColorBlitRendererFeature : ScriptableRendererFeature
{
    public Shader m_Shader;
    public float m_Intensity;

    Material m_Material;

    ColorBlitPass m_RenderPass = null;

    public override void AddRenderPasses(ScriptableRenderer renderer,
                                    ref RenderingData renderingData)
    {
        if (renderingData.cameraData.cameraType == CameraType.Game)
            renderer.EnqueuePass(m_RenderPass);
    }

    public override void SetupRenderPasses(ScriptableRenderer renderer,
                                        in RenderingData renderingData)
    {
        if (renderingData.cameraData.cameraType == CameraType.Game)
        {
            // Calling ConfigureInput with the ScriptableRenderPassInput.Color argument
            // ensures that the opaque texture is available to the Render Pass.
            m_RenderPass.ConfigureInput(ScriptableRenderPassInput.Color);
            m_RenderPass.SetTarget(renderer.cameraColorTargetHandle, m_Intensity);
        }
    }

    public override void Create()
    {
        m_Material = CoreUtils.CreateEngineMaterial(m_Shader);
        m_RenderPass = new ColorBlitPass(m_Material);
    }

    protected override void Dispose(bool disposing)
    {
        CoreUtils.Destroy(m_Material);
    }
}
```

## RenderPass

```cs:ColorBlitPass.cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

internal class ColorBlitPass : ScriptableRenderPass
{
    ProfilingSampler m_ProfilingSampler = new ProfilingSampler("ColorBlit");
    Material m_Material;
    RTHandle m_CameraColorTarget;
    float m_Intensity;

    public ColorBlitPass(Material material)
    {
        m_Material = material;
        renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
    }

    public void SetTarget(RTHandle colorHandle, float intensity)
    {
        m_CameraColorTarget = colorHandle;
        m_Intensity = intensity;
    }

    public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
    {
        ConfigureTarget(m_CameraColorTarget);
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        var cameraData = renderingData.cameraData;
        if (cameraData.camera.cameraType != CameraType.Game)
            return;

        if (m_Material == null)
            return;

        CommandBuffer cmd = CommandBufferPool.Get();
        using (new ProfilingScope(cmd, m_ProfilingSampler))
        {
            m_Material.SetFloat("_Intensity", m_Intensity);
            Blitter.BlitCameraTexture(cmd, m_CameraColorTarget, m_CameraColorTarget, m_Material, 0);
        }
        context.ExecuteCommandBuffer(cmd);
        cmd.Clear();

        CommandBufferPool.Release(cmd);
    }
}
```

## シェーダー

```cs:ColorBlit.shader
Shader "ColorBlit"
{
        SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline" = "UniversalPipeline"}
        LOD 100
        ZWrite Off Cull Off
        Pass
        {
            Name "ColorBlitPass"

            HLSLPROGRAM
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            // The Blit.hlsl file provides the vertex shader (Vert),
            // input structure (Attributes) and output strucutre (Varyings)
            #include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"

            #pragma vertex Vert
            #pragma fragment frag

            TEXTURE2D_X(_CameraOpaqueTexture);
            SAMPLER(sampler_CameraOpaqueTexture);

            float _Intensity;

            half4 frag (Varyings input) : SV_Target
            {
                UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(input);
                float4 color = SAMPLE_TEXTURE2D_X(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, input.texcoord);
                return color * float4(0, _Intensity, 0, 1);
            }
            ENDHLSL
        }
    }
}
```

:::

すると、ポストエフェクトがかかっています。

![](/images/bliter_api_change/unity_sample_blit.png =500x)

今までとどのように変化したか、処理を見ていきます。

# ColorBlitRendererFeature.cs内

## レンダラーのセットアップ

RendererFeatureのインスタンスがレンダラーに割り当てられ、使用可能になったときに呼び出される新しいAPIになります。

`AddRenderPasses`で`renderer.cameraColorTarget`や`renderer.cameraDepthTarget`をしていましたが、この`SetupRenderPasses`に移植する必要があります。

```cs
public override void SetupRenderPasses(ScriptableRenderer renderer,
    in RenderingData renderingData)
{
    if (renderingData.cameraData.cameraType == CameraType.Game)
    {
        // Calling ConfigureInput with the ScriptableRenderPassInput.Color argument
        // ensures that the opaque texture is available to the Render Pass.
        m_RenderPass.ConfigureInput(ScriptableRenderPassInput.Color);
        m_RenderPass.SetTarget(renderer.cameraColorTargetHandle, m_Intensity);
    }
}
```

# ColorBlitPass.cs内

## 新しいRenderTargetのHandlingクラス

今までは、`RenderTargetHandle`クラスを用いてRenderTextureを制御してきました。
URP14では`RTHandle`に置き換えることが推奨されています。
いずれ、`RenderTargetHandle`クラスは無くなるそうです。

https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@13.1/manual/upgrade-guide-2022-1.html

```cs
RTHandle m_CameraColorTarget;
```

## Blit

今までは`cmd.Blit`を使用していました。
URP14からは`Blitter`を用いてBlitを行います。

```cs
public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
{
    ・・・
    using (new ProfilingScope(cmd, m_ProfilingSampler))
    {
        m_Material.SetFloat("_Intensity", m_Intensity);
        Blitter.BlitCameraTexture(cmd, m_CameraColorTarget, m_CameraColorTarget, m_Material, 0);
    }
    ・・・
}
```

中身は以下のようになっており、解像度を考慮してBlitがされています。

```cs:Library/PackageCache/com.unity.render-pipelines.core@14.0.4/Runtime/Utilities/Blitter.cs
public static void BlitCameraTexture(CommandBuffer cmd, RTHandle source, RTHandle destination, Material material, int pass)
{
    Vector2 viewportScale = source.useScaling ? new Vector2(source.rtHandleProperties.rtHandleScale.x, source.rtHandleProperties.rtHandleScale.y) : Vector2.one;
    // Will set the correct camera viewport as well.
    CoreUtils.SetRenderTarget(cmd, destination);
    BlitTexture(cmd, source, viewportScale, material, pass);
}
```

# Blit.shader内

## 頂点シェーダー

`Blit.shader`内では頂点シェーダーは定義されていません。
`Blit.hlsl`にて定義されています。

```cs
// The Blit.hlsl file provides the vertex shader (Vert),
// input structure (Attributes) and output strucutre (Varyings)
#include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"

#pragma vertex Vert
```

### 頂点シェーダーの定義

実装自体はシンプルで、フラグメントシェーダー側に座標とuvを渡しているだけになります。

```cs:Library/PackageCache/com.unity.render-pipelines.core@14.0.4/Runtime/Utilities/Blit.hlsl
#if SHADER_API_GLES
struct Attributes
{
    float4 positionOS       : POSITION;
    float2 uv               : TEXCOORD0;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
#else
struct Attributes
{
    uint vertexID : SV_VertexID;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
#endif

struct Varyings
{
    float4 positionCS : SV_POSITION;
    float2 texcoord   : TEXCOORD0;
    UNITY_VERTEX_OUTPUT_STEREO
};

Varyings Vert(Attributes input)
{
    Varyings output;
    UNITY_SETUP_INSTANCE_ID(input);
    UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(output);

#if SHADER_API_GLES
    float4 pos = input.positionOS;
    float2 uv  = input.uv;
#else
    float4 pos = GetFullScreenTriangleVertexPosition(input.vertexID);
    float2 uv  = GetFullScreenTriangleTexCoord(input.vertexID);
#endif

    output.positionCS = pos;
    output.texcoord   = uv * _BlitScaleBias.xy + _BlitScaleBias.zw;
    return output;
}
```

### _BlitScaleBias

`Blit.hlsl`で出てきた、`_BlitScaleBias`は、
`BlitCameraTexture()`で呼ばれている`BlitTexture`で値を入れられています。

```cs:Library/PackageCache/com.unity.render-pipelines.core@14.0.4/Runtime/Utilities/Blitter.cs
public static void BlitTexture(CommandBuffer cmd, RTHandle source, Vector4 scaleBias, Material material, int pass)
{
    s_PropertyBlock.SetVector(BlitShaderIDs._BlitScaleBias, scaleBias);
    s_PropertyBlock.SetTexture(BlitShaderIDs._BlitTexture, source);
    DrawTriangle(cmd, material, pass);
}
```

## テクスチャのサンプリング

Unityの例では`_CameraOpaqueTexture`を渡しています。
ですが、RenderPipelineAssetの設定次第では渡していないこともあるかと思います。
ですので`_BlitTexture`でサンプリングしてあげても良さそうです。

```diff cs
- float4 color = SAMPLE_TEXTURE2D_X(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, input.texcoord);
+ float4 color = SAMPLE_TEXTURE2D_X(_BlitTexture, sampler_LinearRepeat, input.texcoord);
```