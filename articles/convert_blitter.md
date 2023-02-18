---
title: "既存のRendererFeatureをURP14のBlitに対応させる"
emoji: "🪞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [unity,urp]
published: true
---

# はじめに

URP14ではRendererFeatureでのポストエフェクトのかけ方が変わりました。

https://zenn.dev/sakutaro/articles/bliter_api_change

過去バージョンでは使用できていたRendererFeatureを使って、URP14への対応を行っていきます。

# 環境

Unity 2022.2.2f1
Universal RP 14.0.4

# 過去バージョンのコードを持ってくる

実際にURP10環境で動いていた、RendererFeatureをURP14に持ってきます。
機能としては至ってシンプルで、グレースケールをかけるだけになります。

## RendererFeature

```cs
using UnityEngine;
using UnityEngine.Rendering.Universal;

namespace ConvertBlitter
{
    public class ConvertBlitterGrayscaleRendererFeature : ScriptableRendererFeature
    {
        [SerializeField]
        private Shader shader;

        private ConvertBlitterGrayscalePass grayscalePass;

        public override void Create()
        {
            grayscalePass = new ConvertBlitterGrayscalePass(shader);
        }

        public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        {
            grayscalePass.SetRenderTarget(renderer.cameraColorTarget);
            renderer.EnqueuePass(grayscalePass);
        }
    }
}
```

## RenderPass

```cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

namespace ConvertBlitter
{
    public class ConvertBlitterGrayscalePass : ScriptableRenderPass
    {
        private const string ProfilerTag = nameof(ConvertBlitterGrayscalePass);

        private readonly Material material;

        private RenderTargetHandle tmpRenderTargetHandle;
        private RenderTargetIdentifier cameraColorTarget;

        public ConvertBlitterGrayscalePass(Shader shader)
        {
            material = CoreUtils.CreateEngineMaterial(shader);
            renderPassEvent = RenderPassEvent.AfterRenderingTransparents;
            tmpRenderTargetHandle.Init("_TempRT");
        }

        public void SetRenderTarget(RenderTargetIdentifier target)
        {
            cameraColorTarget = target;
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            if (renderingData.cameraData.isSceneViewCamera)
            {
                return;
            }

            var cmd = CommandBufferPool.Get(ProfilerTag);

            var descriptor = renderingData.cameraData.cameraTargetDescriptor;
            descriptor.depthBufferBits = 0;

            cmd.GetTemporaryRT(tmpRenderTargetHandle.id, descriptor);
            cmd.Blit(cameraColorTarget, tmpRenderTargetHandle.Identifier(), material);
            cmd.Blit(tmpRenderTargetHandle.Identifier(), cameraColorTarget);
            cmd.ReleaseTemporaryRT(tmpRenderTargetHandle.id);

            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
}
```

## シェーダー

```cs
Shader "ConvertBlitter/ConvertBlitterGrayScale"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "Renderpipeline"="UniversalPipeline" }

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct Varyings
            {
                float2 uv : TEXCOORD0;
                float4 positionHCS : SV_POSITION;
            };

            TEXTURE2D(_MainTex);
            SAMPLER(sampler_MainTex);

            CBUFFER_START(UnityPerMaterial)
            float4 _MainTex_ST;
            CBUFFER_END

            Varyings vert (Attributes IN)
            {
                Varyings OUT;
                OUT.positionHCS = TransformObjectToHClip(IN.positionOS.xyz);
                OUT.uv = TRANSFORM_TEX(IN.uv, _MainTex);
                return OUT;
            }

            half4 frag (Varyings IN) : SV_Target
            {
                half4 col = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv);
                half gray = dot(col.rgb, half3(0.299, 0.587, 0.114));
                return half4(gray, gray, gray, col.a);
            }
            ENDHLSL
        }
    }
}
```


# 描画できるようにする

このままですと、エラーで描画ができません。
1つ1つエラーを読み解いて、正しくポストエフェクトがかかるようにします。

## SetupRenderPassesに移行する

上記のソースコードそのままですと以下のようなエラーが出ます。

![](/images/convert_blitter/error_color_target.png =400x)
*You can only call cameraColorTarget inside the scope of a ScriptableRenderPass. Otherwise the pipeline camera target texture might have not been created or might have already been disposed.*

これは、RenderTargetTextureが作成されていない、もしくは破棄されている可能性があります。
`cameraColorTarget`は、`ScriptableRenderPass`のスコープで呼んでください。
という意味合いです。

このエラー文通り、`ScriptableRenderPass`で呼ぶとエラーが消えてグレースケールのポストエフェクトがかかります。

```diff cs:ConvertBlitterGrayscaleRendererFeature.cs
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
-       grayscalePass.SetRenderTarget(renderer.cameraColorTarget);
        renderer.EnqueuePass(grayscalePass);
    }

+    public override void SetupRenderPasses(ScriptableRenderer renderer, in RenderingData renderingData)
+    {
+        grayscalePass.SetRenderTarget(renderer.cameraColorTarget);
+    }

```

### 描画結果

![](/images/convert_blitter/color_target_to_scriptable_render_pass.png =500x)

# 警告を消す

これで描画自体はできましたが、警告が出ているので直します。

## RTHandleを渡すようにする

![](/images/convert_blitter/warning_colot_target.png =500x)
*Assets/ConvertBliter/Scripts/ConvertBliterGrayscalePass.cs(13,17): warning CS0618: 'RenderTargetHandle' is obsolete: 'Deprecated in favor of RTHandle'*

`RenderTargetHandle`は廃止予定なので、`RTHandle`に置換してねという警告です。
ですので、`RTHandle`に置換してあげます。

```diff cs:ConvertBlitterGrayscaleRendererFeature.cs
    public override void SetupRenderPasses(ScriptableRenderer renderer, in RenderingData renderingData)
    {
-       grayscalePass.SetRenderTarget(renderer.cameraColorTarget);
+       grayscalePass.SetRenderTarget(renderer.cameraColorTargetHandle);
    }
```

ついでに呼び出し側の`SetRenderTarget`の引数も`RenderTargetIdentifier`から`RTHandle`に変えます。

```diff cs:ConvertBlitterGrayscalePass.cs
-   private RenderTargetIdentifier cameraColorTarget;
+   private RTHandle cameraColorTarget;

    ・・・

-   public void SetRenderTarget(RenderTargetIdentifier target)
+   public void SetRenderTarget(RTHandle target)
    {
        cameraColorTarget = target;
    }
```

`private RenderTargetHandle tmpRenderTargetHandle;`に関しては、次のフェーズで消します。

### RTHandleをRenderTargetIdentifierに代入できる理由

`RTHandle`が、`RenderTargetIdentifier`を持っており、その値と比較してくれているからになります。

```cs:Library/PackageCache/com.unity.render-pipelines.core@14.0.4/Runtime/Textures/RTHandle.cs
/// <summary>
/// Implicit conversion operator to RenderTargetIdentifier
/// </summary>
/// <param name="handle">Input RTHandle</param>
/// <returns>RenderTargetIdentifier representation of the RTHandle.</returns>
public static implicit operator RenderTargetIdentifier(RTHandle handle)
{
    return handle != null ? handle.nameID : default(RenderTargetIdentifier);
}
```

# Blitterで描画

URP14ですので`Blitter`を使用して描画します。

## Blitterに置換

`ConvertBlitterGrayscalePass`で`cmd.Blit`をしていたのを`Blitter.BlitCameraTexture`へと変えます。

```diff cs:ConvertBlitterGrayscalePass
-   private RenderTargetHandle tmpRenderTargetHandle;

    ・・・

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        if (renderingData.cameraData.isSceneViewCamera)
        {
            return;
        }

        var cmd = CommandBufferPool.Get(ProfilerTag);

-       var descriptor = renderingData.cameraData.cameraTargetDescriptor;
-       descriptor.depthBufferBits = 0;
-
-       cmd.GetTemporaryRT(tmpRenderTargetHandle.id, descriptor);
-       cmd.Blit(cameraColorTarget, tmpRenderTargetHandle.Identifier(), material);
-       cmd.Blit(tmpRenderTargetHandle.Identifier(), cameraColorTarget);
-       cmd.ReleaseTemporaryRT(tmpRenderTargetHandle.id);

+       Blitter.BlitCameraTexture(cmd, cameraColorTarget, cameraColorTarget, material, 0);
        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
```

エラーは出ていませんが、gameビュー上でポストエフェクトが適応されなくなってしまいました。

![](/images/convert_blitter/to_blitter_render_pass.png =500x)

そこで、FrameDebugを見てみます。
そうすると、ポストエフェクトのPassが通っていることがわかります。
なので、シェーダー側に問題がありそうです。

![](/images/convert_blitter/frame_debug.png =700x)

## シェーダーをBlitterに対応させる

`Blit.hlsl`に処理を任せても良いですが、頂点シェーダーを変更したいケースだと仮定してなるべく`Blit.hlsl`に任せないようにします。

Blitした画像の解像度やRenderTargetは`Blit.hlsl`にしかないのでincludeします。
includeすると構造体の名前が重複してしまうので、変更します。

```diff cs:ConvertBlitterGrayScale.shader
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
+   #include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"

-   struct Attributes
+   struct GrayscaleAttributes
    {
        float4 positionOS : POSITION;
        float2 uv : TEXCOORD0;
    };

-   struct Varyings
+   struct GrayscaleVaryings
    {
        float2 uv : TEXCOORD0;
        float4 positionHCS : SV_POSITION;
    };

    // 以降のAttributesとVaryingsも置換する
```

後は、頂点シェーダーとフラグメントシェーダーを対応させて完了になります。

```diff cs:ConvertBlitterGrayScale.shader
-   struct GrayscaleAttributes
-   {
-       float4 positionOS : POSITION;
-       float2 uv : TEXCOORD0;
-   };
+   #if SHADER_API_GLES
+       struct GrayscaleAttributes
+       {
+           float4 positionOS : POSITION;
+           float2 uv : TEXCOORD0;
+       };
+   #else
+       struct GrayscaleAttributes
+       {
+           uint vertexID : SV_VertexID;
+       };
+   #endif

    GrayscaleVaryings vert (GrayscaleAttributes IN)
    {
        GrayscaleVaryings OUT;

+       #if SHADER_API_GLES
+           float4 pos = input.positionOS;
+           float2 uv  = input.uv;
+       #else
+           float4 pos = GetFullScreenTriangleVertexPosition(IN.vertexID);
+           float2 uv  = GetFullScreenTriangleTexCoord(IN.vertexID);
+       #endif

-       OUT.positionHCS = IN.positionOS;
-       OUT.uv = IN.uv * _BlitScaleBias.xy + _BlitScaleBias.zw;
+       OUT.positionHCS = pos;
+       OUT.uv = uv;
        return OUT;
    }

    half4 frag (GrayscaleVaryings IN) : SV_Target
    {
-       half4 col = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv);
+       half4 col = SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearRepeat, IN.uv);
        half gray = dot(col.rgb, half3(0.299, 0.587, 0.114));
        return half4(gray, gray, gray, col.a);
    }
```

グレースケールになり、Blitterを使用したポストエフェクトをかけることができました。

![](/images/convert_blitter/result.png =500x)

# 対応後コード

最後に対応後のコードを紹介して終わりになります。

## RendererFeature

```cs
using UnityEngine;
using UnityEngine.Rendering.Universal;

namespace ConvertBlitter
{
    public class ConvertBlitterGrayscaleRendererFeature : ScriptableRendererFeature
    {
        [SerializeField]
        private Shader shader;

        private ConvertBlitterGrayscalePass grayscalePass;

        public override void Create()
        {
            grayscalePass = new ConvertBlitterGrayscalePass(shader);
        }

        public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        {
            renderer.EnqueuePass(grayscalePass);
        }

        // renderer.cameraColorTargetはSetupRenderPasses内で呼ぶ
        public override void SetupRenderPasses(ScriptableRenderer renderer, in RenderingData renderingData)
        {
            // cameraColorTarget -> cameraColorTargetHandleにする
            grayscalePass.SetRenderTarget(renderer.cameraColorTargetHandle);
        }
    }
}
```

## RenderPass

```cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

namespace ConvertBlitter
{
    public class ConvertBlitterGrayscalePass : ScriptableRenderPass
    {
        private const string ProfilerTag = nameof(ConvertBlitterGrayscalePass);

        private readonly Material material;

        private RTHandle cameraColorTarget;

        public ConvertBlitterGrayscalePass(Shader shader)
        {
            material = CoreUtils.CreateEngineMaterial(shader);
            renderPassEvent = RenderPassEvent.AfterRenderingTransparents;
        }

        public void SetRenderTarget(RTHandle target)
        {
            cameraColorTarget = target;
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            if (renderingData.cameraData.isSceneViewCamera)
            {
                return;
            }

            var cmd = CommandBufferPool.Get(ProfilerTag);
            // Blitterで描画する
            Blitter.BlitCameraTexture(cmd, cameraColorTarget, cameraColorTarget, material, 0);
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
}
```

## シェーダー

```cs
Shader "ConvertBlitter/ConvertBlitterGrayScale"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "Renderpipeline"="UniversalPipeline" }

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"

            #if SHADER_API_GLES
                struct GrayscaleAttributes
                {
                    float4 positionOS : POSITION;
                    float2 uv : TEXCOORD0;
                };
            #else
                struct GrayscaleAttributes
                {
                    uint vertexID : SV_VertexID;
                };
            #endif

            struct GrayscaleVaryings
            {
                float2 uv : TEXCOORD0;
                float4 positionHCS : SV_POSITION;
            };

            TEXTURE2D(_MainTex);
            SAMPLER(sampler_MainTex);

            CBUFFER_START(UnityPerMaterial)
            float4 _MainTex_ST;
            CBUFFER_END

            GrayscaleVaryings vert (GrayscaleAttributes IN)
            {
                GrayscaleVaryings OUT;
                #if SHADER_API_GLES
                    float4 pos = input.positionOS;
                    float2 uv  = input.uv;
                #else
                    float4 pos = GetFullScreenTriangleVertexPosition(IN.vertexID);
                    float2 uv  = GetFullScreenTriangleTexCoord(IN.vertexID);
                #endif

                OUT.positionHCS = pos;
                OUT.uv = uv;
                return OUT;
            }

            half4 frag (GrayscaleVaryings IN) : SV_Target
            {
                half4 col = SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearRepeat, IN.uv);
                half gray = dot(col.rgb, half3(0.299, 0.587, 0.114));
                return half4(gray, gray, gray, col.a);
            }
            ENDHLSL
        }
    }
}
```