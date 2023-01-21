---
title: "URP14で追加された、Full Screen Pass Renderer Featureを読む"
emoji: "🎆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [unity,urp]
published: true
---

# はじめに

URP14から新たに`Full Screen Pass Renderer Feature`というものが追加されました。

https://docs.unity.cn/Packages/com.unity.render-pipelines.universal@14.0/manual/renderer-features/renderer-feature-full-screen-pass.html

これにより今までより簡単にポストエフェクトをかけることができるようになりました。
中身がどうなっているか気になったので、読んでみました。


:::details FullScreenPassRendererFeatureの中身

```cs
using UnityEditor;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class using UnityEditor;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class FullScreenPassRendererFeature : ScriptableRendererFeature
{
    public enum InjectionPoint
    {
        BeforeRenderingTransparents = RenderPassEvent.BeforeRenderingTransparents,
        BeforeRenderingPostProcessing = RenderPassEvent.BeforeRenderingPostProcessing,
        AfterRenderingPostProcessing = RenderPassEvent.AfterRenderingPostProcessing
    }

    public Material passMaterial;
    public InjectionPoint injectionPoint = InjectionPoint.AfterRenderingPostProcessing;
    public ScriptableRenderPassInput requirements = ScriptableRenderPassInput.Color;
    [HideInInspector] // We draw custom pass index entry with the dropdown inside FullScreenPassRendererFeatureEditor.cs
    public int passIndex = 0;

    private FullScreenRenderPass fullScreenPass;
    private bool requiresColor;
    private bool injectedBeforeTransparents;

    public override void Create()
    {
        fullScreenPass = new FullScreenRenderPass();
        fullScreenPass.renderPassEvent = (RenderPassEvent)injectionPoint;

        // This copy of requirements is used as a parameter to configure input in order to avoid copy color pass
        ScriptableRenderPassInput modifiedRequirements = requirements;

        requiresColor = (requirements & ScriptableRenderPassInput.Color) != 0;
        injectedBeforeTransparents = injectionPoint <= InjectionPoint.BeforeRenderingTransparents;

        if (requiresColor && !injectedBeforeTransparents)
        {
            // Removing Color flag in order to avoid unnecessary CopyColor pass
            // Does not apply to before rendering transparents, due to how depth and color are being handled until
            // that injection point.
            modifiedRequirements ^= ScriptableRenderPassInput.Color;
        }
        fullScreenPass.ConfigureInput(modifiedRequirements);
    }

    // Here you can inject one or multiple render passes in the renderer.
    // This method is called when setting up the renderer once per-camera.
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (passMaterial == null)
        {
            Debug.LogWarningFormat("Missing Post Processing effect Material. {0} Fullscreen pass will not execute. Check for missing reference in the assigned renderer.", GetType().Name);
            return;
        }
        fullScreenPass.Setup(passMaterial, passIndex, requiresColor, injectedBeforeTransparents, "FullScreenPassRendererFeature", renderingData);

        renderer.EnqueuePass(fullScreenPass);
    }

    protected override void Dispose(bool disposing)
    {
        fullScreenPass.Dispose();
    }

    class FullScreenRenderPass : ScriptableRenderPass
    {
        private Material m_PassMaterial;
        private int m_PassIndex;
        private bool m_RequiresColor;
        private bool m_IsBeforeTransparents;
        private PassData m_PassData;
        private ProfilingSampler m_ProfilingSampler;
        private RTHandle m_CopiedColor;

        public void Setup(Material mat, int index, bool requiresColor, bool isBeforeTransparents, string featureName, in RenderingData renderingData)
        {
            m_PassMaterial = mat;
            m_PassIndex = index;
            m_RequiresColor = requiresColor;
            m_IsBeforeTransparents = isBeforeTransparents;
            m_ProfilingSampler ??= new ProfilingSampler(featureName);

            var colorCopyDescriptor = renderingData.cameraData.cameraTargetDescriptor;
            colorCopyDescriptor.depthBufferBits = (int) DepthBits.None;
            RenderingUtils.ReAllocateIfNeeded(ref m_CopiedColor, colorCopyDescriptor, name: "_FullscreenPassColorCopy");

            m_PassData ??= new PassData();
        }

        public void Dispose()
        {
            m_CopiedColor?.Release();
        }


        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            m_PassData.effectMaterial = m_PassMaterial;
            m_PassData.passIndex = m_PassIndex;
            m_PassData.requiresColor = m_RequiresColor;
            m_PassData.isBeforeTransparents = m_IsBeforeTransparents;
            m_PassData.profilingSampler = m_ProfilingSampler;
            m_PassData.copiedColor = m_CopiedColor;

            ExecutePass(m_PassData, ref renderingData, ref context);
        }

        // RG friendly method
        private static void ExecutePass(PassData passData, ref RenderingData renderingData, ref ScriptableRenderContext context)
        {
            var passMaterial = passData.effectMaterial;
            var passIndex = passData.passIndex;
            var requiresColor = passData.requiresColor;
            var isBeforeTransparents = passData.isBeforeTransparents;
            var copiedColor = passData.copiedColor;
            var profilingSampler = passData.profilingSampler;

            if (passMaterial == null)
            {
                // should not happen as we check it in feature
                return;
            }

            if (renderingData.cameraData.isPreviewCamera)
            {
                return;
            }

            CommandBuffer cmd = renderingData.commandBuffer;
            var cameraData = renderingData.cameraData;

            using (new ProfilingScope(cmd, profilingSampler))
            {
                if (requiresColor)
                {
                    // For some reason BlitCameraTexture(cmd, dest, dest) scenario (as with before transparents effects) blitter fails to correctly blit the data
                    // Sometimes it copies only one effect out of two, sometimes second, sometimes data is invalid (as if sampling failed?).
                    // Adding RTHandle in between solves this issue.
                    var source = isBeforeTransparents ? cameraData.renderer.GetCameraColorBackBuffer(cmd) : cameraData.renderer.cameraColorTargetHandle;

                    Blitter.BlitCameraTexture(cmd, source, copiedColor);
                    passMaterial.SetTexture(Shader.PropertyToID("_BlitTexture"), copiedColor);
                }

                CoreUtils.SetRenderTarget(cmd, cameraData.renderer.GetCameraColorBackBuffer(cmd));
                CoreUtils.DrawFullScreen(cmd, passMaterial);
                context.ExecuteCommandBuffer(cmd);
                cmd.Clear();
            }
        }

        private class PassData
        {
            internal Material effectMaterial;
            internal int passIndex;
            internal bool requiresColor;
            internal bool isBeforeTransparents;
            public ProfilingSampler profilingSampler;
            public RTHandle copiedColor;
        }
    }

}
 : ScriptableRendererFeature
{
    public enum InjectionPoint
    {
        BeforeRenderingTransparents = RenderPassEvent.BeforeRenderingTransparents,
        BeforeRenderingPostProcessing = RenderPassEvent.BeforeRenderingPostProcessing,
        AfterRenderingPostProcessing = RenderPassEvent.AfterRenderingPostProcessing
    }

    public Material passMaterial;
    public InjectionPoint injectionPoint = InjectionPoint.AfterRenderingPostProcessing;
    public ScriptableRenderPassInput requirements = ScriptableRenderPassInput.Color;
    [HideInInspector] // We draw custom pass index entry with the dropdown inside FullScreenPassRendererFeatureEditor.cs
    public int passIndex = 0;

    private FullScreenRenderPass fullScreenPass;
    private bool requiresColor;
    private bool injectedBeforeTransparents;

    public override void Create()
    {
        fullScreenPass = new FullScreenRenderPass();
        fullScreenPass.renderPassEvent = (RenderPassEvent)injectionPoint;

        // This copy of requirements is used as a parameter to configure input in order to avoid copy color pass
        ScriptableRenderPassInput modifiedRequirements = requirements;

        requiresColor = (requirements & ScriptableRenderPassInput.Color) != 0;
        injectedBeforeTransparents = injectionPoint <= InjectionPoint.BeforeRenderingTransparents;

        if (requiresColor && !injectedBeforeTransparents)
        {
            // Removing Color flag in order to avoid unnecessary CopyColor pass
            // Does not apply to before rendering transparents, due to how depth and color are being handled until
            // that injection point.
            modifiedRequirements ^= ScriptableRenderPassInput.Color;
        }
        fullScreenPass.ConfigureInput(modifiedRequirements);
    }

    // Here you can inject one or multiple render passes in the renderer.
    // This method is called when setting up the renderer once per-camera.
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (passMaterial == null)
        {
            Debug.LogWarningFormat("Missing Post Processing effect Material. {0} Fullscreen pass will not execute. Check for missing reference in the assigned renderer.", GetType().Name);
            return;
        }
        fullScreenPass.Setup(passMaterial, passIndex, requiresColor, injectedBeforeTransparents, "FullScreenPassRendererFeature", renderingData);

        renderer.EnqueuePass(fullScreenPass);
    }

    protected override void Dispose(bool disposing)
    {
        fullScreenPass.Dispose();
    }

    class FullScreenRenderPass : ScriptableRenderPass
    {
        private Material m_PassMaterial;
        private int m_PassIndex;
        private bool m_RequiresColor;
        private bool m_IsBeforeTransparents;
        private PassData m_PassData;
        private ProfilingSampler m_ProfilingSampler;
        private RTHandle m_CopiedColor;

        public void Setup(Material mat, int index, bool requiresColor, bool isBeforeTransparents, string featureName, in RenderingData renderingData)
        {
            m_PassMaterial = mat;
            m_PassIndex = index;
            m_RequiresColor = requiresColor;
            m_IsBeforeTransparents = isBeforeTransparents;
            m_ProfilingSampler ??= new ProfilingSampler(featureName);

            var colorCopyDescriptor = renderingData.cameraData.cameraTargetDescriptor;
            colorCopyDescriptor.depthBufferBits = (int) DepthBits.None;
            RenderingUtils.ReAllocateIfNeeded(ref m_CopiedColor, colorCopyDescriptor, name: "_FullscreenPassColorCopy");

            m_PassData ??= new PassData();
        }

        public void Dispose()
        {
            m_CopiedColor?.Release();
        }


        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            m_PassData.effectMaterial = m_PassMaterial;
            m_PassData.passIndex = m_PassIndex;
            m_PassData.requiresColor = m_RequiresColor;
            m_PassData.isBeforeTransparents = m_IsBeforeTransparents;
            m_PassData.profilingSampler = m_ProfilingSampler;
            m_PassData.copiedColor = m_CopiedColor;

            ExecutePass(m_PassData, ref renderingData, ref context);
        }

        // RG friendly method
        private static void ExecutePass(PassData passData, ref RenderingData renderingData, ref ScriptableRenderContext context)
        {
            var passMaterial = passData.effectMaterial;
            var passIndex = passData.passIndex;
            var requiresColor = passData.requiresColor;
            var isBeforeTransparents = passData.isBeforeTransparents;
            var copiedColor = passData.copiedColor;
            var profilingSampler = passData.profilingSampler;

            if (passMaterial == null)
            {
                // should not happen as we check it in feature
                return;
            }

            if (renderingData.cameraData.isPreviewCamera)
            {
                return;
            }

            CommandBuffer cmd = renderingData.commandBuffer;
            var cameraData = renderingData.cameraData;

            using (new ProfilingScope(cmd, profilingSampler))
            {
                if (requiresColor)
                {
                    // For some reason BlitCameraTexture(cmd, dest, dest) scenario (as with before transparents effects) blitter fails to correctly blit the data
                    // Sometimes it copies only one effect out of two, sometimes second, sometimes data is invalid (as if sampling failed?).
                    // Adding RTHandle in between solves this issue.
                    var source = isBeforeTransparents ? cameraData.renderer.GetCameraColorBackBuffer(cmd) : cameraData.renderer.cameraColorTargetHandle;

                    Blitter.BlitCameraTexture(cmd, source, copiedColor);
                    passMaterial.SetTexture(Shader.PropertyToID("_BlitTexture"), copiedColor);
                }

                CoreUtils.SetRenderTarget(cmd, cameraData.renderer.GetCameraColorBackBuffer(cmd));
                CoreUtils.DrawFullScreen(cmd, passMaterial);
                context.ExecuteCommandBuffer(cmd);
                cmd.Clear();
            }
        }

        private class PassData
        {
            internal Material effectMaterial;
            internal int passIndex;
            internal bool requiresColor;
            internal bool isBeforeTransparents;
            public ProfilingSampler profilingSampler;
            public RTHandle copiedColor;
        }
    }

}
```

:::

`FullScreenPassRendererFeature.cs`と`FullScreenRenderPass.cs`にわかれています。

# FullScreenPassRendererFeature

## inspector上で操作可能な変数

```cs
public enum InjectionPoint
{
    BeforeRenderingTransparents = RenderPassEvent.BeforeRenderingTransparents,
    BeforeRenderingPostProcessing = RenderPassEvent.BeforeRenderingPostProcessing,
    AfterRenderingPostProcessing = RenderPassEvent.AfterRenderingPostProcessing
}

// 対象となるマテリアル
public Material passMaterial;

// 差し込むRenderPassEvent
public InjectionPoint injectionPoint = InjectionPoint.AfterRenderingPostProcessing;

// ポストエフェクトに必要なPass
public ScriptableRenderPassInput requirements = ScriptableRenderPassInput.Color;

// マテリアルの何Pass目を使用するか
[HideInInspector]
public int passIndex = 0;
```

### InjectionPoint

元々挿入するRenderPassを指定するものとして`RenderPassEvent`がありますが
ポストエフェクトで使用する前提のスクリプトですので、別の型として`InjectionPoint`が制作されています。

### ScriptableRenderPassInput

`ScriptableRenderPassInput`はポストエフェクトに必要なものを追加するもので、Flagsとなっています。

| 名前   | 意味                         |
| ------ | ---------------------------- |
| None   | テクスチャが不要             |
| Depth  | 深度テクスチャ               |
| Normal | 法線テクスチャ               |
| Color  | カラーテクスチャ             |
| Motion | モーションベクターテクスチャ |

:::details ScriptableRenderPassInputの定義
```cs
/// <summary>
/// Input requirements for <c>ScriptableRenderPass</c>.
/// </summary>
/// <seealso cref="ConfigureInput"/>
[Flags]
public enum ScriptableRenderPassInput
{
    /// <summary>
    /// Used when a <c>ScriptableRenderPass</c> does not require any texture.
    /// </summary>
    None = 0,

    /// <summary>
    /// Used when a <c>ScriptableRenderPass</c> requires a depth texture.
    /// </summary>
    Depth = 1 << 0,

    /// <summary>
    /// Used when a <c>ScriptableRenderPass</c> requires a normal texture.
    /// </summary>
    Normal = 1 << 1,

    /// <summary>
    /// Used when a <c>ScriptableRenderPass</c> requires a color texture.
    /// </summary>
    Color = 1 << 2,

    /// <summary>
    /// Used when a <c>ScriptableRenderPass</c> requires a motion vectors texture.
    /// </summary>
    Motion = 1 << 3,
}
```

:::

MotionVectorだけ聞いたことがなかったので、参考サイト様を添付します。
https://light11.hatenadiary.com/entry/2018/09/25/211539

### passIndex

`passIndex`はHideInInspectorですが、`FullScreenPassRendererFeatureEditor.cs`で描画されているので表示されています。

:::details 描画箇所

```cs
private void DrawAdditionalProperties()
{
    List<string> selectablePasses;
    bool isMaterialValid = m_AffectedFeature.passMaterial != null;
    selectablePasses = isMaterialValid ? GetPassIndexStringEntries(m_AffectedFeature) : new List<string>() {"No material"};

    // If material is invalid 0'th index is selected automatically, so it stays on "No material" entry
    // It is invalid index, but FullScreenPassRendererFeature wont execute until material is valid
    var choiceIndex = EditorGUILayout.Popup("Pass Index", m_AffectedFeature.passIndex, selectablePasses.ToArray());

    m_PassIndexToUse = choiceIndex;

}

private List<string> GetPassIndexStringEntries(FullScreenPassRendererFeature component)
{
    List<string> passIndexEntries = new List<string>();
    for (int i = 0; i < component.passMaterial.passCount; ++i)
    {
        // "Name of a pass (index)" - "PassAlpha (1)"
        string entry = $"{component.passMaterial.GetPassName(i)} ({i})";
        passIndexEntries.Add(entry);
    }

    return passIndexEntries;
}
```

:::

### その他の変数


```cs
// フルスクリーンのRendererPass
private FullScreenRenderPass fullScreenPass;

// ColorのRenderPassInputが必要か
private bool requiresColor;

// 透過オブジェクト描画前のRenderPassEventか
private bool injectedBeforeTransparents;
```

## 生成

RenderPassを生成している箇所になります。

```cs
public override void Create()
{
    // Passの生成
    fullScreenPass = new FullScreenRenderPass();
    fullScreenPass.renderPassEvent = (RenderPassEvent)injectionPoint;

    ScriptableRenderPassInput modifiedRequirements = requirements;

    requiresColor = (requirements & ScriptableRenderPassInput.Color) != 0;
    injectedBeforeTransparents = injectionPoint <= InjectionPoint.BeforeRenderingTransparents;

    // Colorを指定していて、半透明描画前ならColorフラグを削除する
    if (requiresColor && !injectedBeforeTransparents)
    {
        modifiedRequirements ^= ScriptableRenderPassInput.Color;
    }

    // ScriptableRenderPassInputの登録
    fullScreenPass.ConfigureInput(modifiedRequirements);
}
```

### Colorフラグの削除

半透明描画前ですと、まだ色が確定していないためColorPassが不要なものとなってしまう可能性があります。
なので削除しているようです。

```cs
// Colorを指定していて、半透明描画前ならColorフラグを削除する
if (requiresColor && !injectedBeforeTransparents)
{
    modifiedRequirements ^= ScriptableRenderPassInput.Color;
}
```

## RenderPassの追加

RenderPassを追加している箇所になります。

```cs
public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
{
    // inspectorからマテリアルが刺されていないならWarningを出す
    if (passMaterial == null)
    {
        Debug.LogWarningFormat("Missing Post Processing effect Material. {0} Fullscreen pass will not execute. Check for missing reference in the assigned renderer.", GetType().Name);
        return;
    }

    // Passの設定
    fullScreenPass.Setup(passMaterial, passIndex, requiresColor, injectedBeforeTransparents, "FullScreenPassRendererFeature", renderingData);

    // Passの追加
    renderer.EnqueuePass(fullScreenPass);
}
```

## 解放

```cs
protected override void Dispose(bool disposing)
{
    fullScreenPass.Dispose();
}
```

# FullScreenRenderPass

## 変数

```cs
// 対象のマテリアル
private Material m_PassMaterial;
// マテリアルの使用するPass
private int m_PassIndex;
// 色情報が必要か
private bool m_RequiresColor;
// 半透明描画前か
private bool m_IsBeforeTransparents;
// 受け渡し構造体
private PassData m_PassData;
// Profiling用
private ProfilingSampler m_ProfilingSampler;
// RenderTexture
private RTHandle m_CopiedColor;
```

## セットアップ

`FullScreenPassRendererFeature`の`AddRenderPasses`で呼び出している箇所になります。

```cs
public void Setup(Material mat, int index, bool requiresColor, bool isBeforeTransparents, string featureName, in RenderingData renderingData)
{
    m_PassMaterial = mat;
    m_PassIndex = index;
    m_RequiresColor = requiresColor;
    m_IsBeforeTransparents = isBeforeTransparents;
    // Profiling用、ScreenPassと直接は関係ない
    m_ProfilingSampler ??= new ProfilingSampler(featureName);

    // Descriptorをコピー
    var colorCopyDescriptor = renderingData.cameraData.cameraTargetDescriptor;
    // ポストエフェクト用なので深度を0に
    colorCopyDescriptor.depthBufferBits = (int) DepthBits.None;
    // RenderTextureが異なるなら上書き
    RenderingUtils.ReAllocateIfNeeded(ref m_CopiedColor, colorCopyDescriptor, name: "_FullscreenPassColorCopy");

    m_PassData ??= new PassData();
}
```

### renderingData.cameraData.cameraTargetDescriptor

RenderTextureの作成に必要な情報が全て入っている構造体になります。

https://docs.unity3d.com/2022.2/Documentation/ScriptReference/RenderTextureDescriptor.html

### RenderingUtils.ReAllocateIfNeeded

第一引数と第二引数に入っているRenderTextureの情報を比較しています。
同じだったら、そのままreturn
違う場合は第二引数の値を元に`RTHandle`を生成しています。

```cs
public static bool ReAllocateIfNeeded(
    ref RTHandle handle,
    in RenderTextureDescriptor descriptor,
    FilterMode filterMode = FilterMode.Point,
    TextureWrapMode wrapMode = TextureWrapMode.Repeat,
    bool isShadowMap = false,
    int anisoLevel = 1,
    float mipMapBias = 0,
    string name = "")
{
    // handleとdescriptorを比較している
    if (RTHandleNeedsReAlloc(handle, descriptor, filterMode, wrapMode, isShadowMap, anisoLevel, mipMapBias, name, false))
    {
        handle?.Release();
        // 生成
        handle = RTHandles.Alloc(descriptor, filterMode, wrapMode, isShadowMap, anisoLevel, mipMapBias, name);
        return true;
    }
    // 何もしない
    return false;
}
```

## 解放

RenderTextureの解放を行っています。

```cs
public void Dispose()
{
    m_CopiedColor?.Release();
}
```


## 実行

`PassData`に情報を伝播しています。

```cs
public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
{
    m_PassData.effectMaterial = m_PassMaterial;
    m_PassData.passIndex = m_PassIndex;
    m_PassData.requiresColor = m_RequiresColor;
    m_PassData.isBeforeTransparents = m_IsBeforeTransparents;
    m_PassData.profilingSampler = m_ProfilingSampler;
    m_PassData.copiedColor = m_CopiedColor;

    ExecutePass(m_PassData, ref renderingData, ref context);
}
```

## Passの実行

実際にPassを実行している箇所になります。
プレビューカメラは以下の右下のカメラになります。
![](/images/full_screen_pass_renderer_feature/preview_camera.png =500x)

```cs
private static void ExecutePass(PassData passData, ref RenderingData renderingData, ref ScriptableRenderContext context)
{
    var passMaterial = passData.effectMaterial;
    var passIndex = passData.passIndex;
    var requiresColor = passData.requiresColor;
    var isBeforeTransparents = passData.isBeforeTransparents;
    var copiedColor = passData.copiedColor;
    var profilingSampler = passData.profilingSampler;

    // マテリアルがないならreturn
    if (passMaterial == null)
    {
        return;
    }

    // プレビューカメラならreturn
    if (renderingData.cameraData.isPreviewCamera)
    {
        return;
    }

    CommandBuffer cmd = renderingData.commandBuffer;
    var cameraData = renderingData.cameraData;

    // Profiling用、ScreenPassと直接は関係ない
    using (new ProfilingScope(cmd, profilingSampler))
    {
        if (requiresColor)
        {
            // 半透明描画前ならバックバッファを、そうでないならカラーターゲットを返す
            var source = isBeforeTransparents ? cameraData.renderer.GetCameraColorBackBuffer(cmd) : cameraData.renderer.cameraColorTargetHandle;

            // Blit
            Blitter.BlitCameraTexture(cmd, source, copiedColor);
            passMaterial.SetTexture(Shader.PropertyToID("_BlitTexture"), copiedColor);
        }

        // RenderTargetのセットアップ
        CoreUtils.SetRenderTarget(cmd, cameraData.renderer.GetCameraColorBackBuffer(cmd));
        // 全画面を三角形に描画
        CoreUtils.DrawFullScreen(cmd, passMaterial);
        // コマンドバッファの実行
        context.ExecuteCommandBuffer(cmd);
        // コマンドバッファの解放
        cmd.Clear();
    }
}
```

### 間にRTHandleを挟む

```cs
if (requiresColor)
{
    // For some reason BlitCameraTexture(cmd, dest, dest) scenario (as with before transparents effects) blitter fails to correctly blit the data
    // Sometimes it copies only one effect out of two, sometimes second, sometimes data is invalid (as if sampling failed?).
    // Adding RTHandle in between solves this issue.
    var source = isBeforeTransparents ? cameraData.renderer.GetCameraColorBackBuffer(cmd) : cameraData.renderer.cameraColorTargetHandle;

    Blitter.BlitCameraTexture(cmd, source, copiedColor);
    passMaterial.SetTexture(Shader.PropertyToID("_BlitTexture"), copiedColor);
}
```


コメントを翻訳すると、以下になります。

```
// いくつかの理由で BlitCameraTexture(cmd, dest, dest) シナリオ (以前の透過効果の場合と同様) blitter は正しくデータをブリットすることに失敗します。
// 2つのエフェクトのうち1つだけをコピーすることもあれば、2つ目をコピーすることもあり、データが無効なこともあります（サンプリングに失敗したかのような）。
// 間に RTHandle を追加することで、この問題を解決します。
```

実際にこの箇所をコメントアウトすると、描画が崩れてしまいました。

![](/images/full_screen_pass_renderer_feature/diff_comment_out.png)

調べてもよくわからなかったので、わかるかたいたら共有してくださると助かります。

### バッファの選定

半透明描画前ですと、まだ色が確定していないためバックバッファを返しています。

```cs
var source = isBeforeTransparents ? cameraData.renderer.GetCameraColorBackBuffer(cmd) : cameraData.renderer.cameraColorTargetHandle;
```

### Blit

```cs
Blitter.BlitCameraTexture(cmd, source, copiedColor);
passMaterial.SetTexture(Shader.PropertyToID("_BlitTexture"), copiedColor);
```

`Blitter.BlitCameraTexture`は、第二引数の`RTHandle`を第三引数の`RTHandle`にBlitする関数になります。
この関数を使用することで解像度を適切に考慮することができます。

```cs
public static void BlitCameraTexture(CommandBuffer cmd, RTHandle source, RTHandle destination, float mipLevel = 0.0f, bool bilinear = false)
{
    // スケーリングを考慮
    Vector2 viewportScale = source.useScaling ? new Vector2(source.rtHandleProperties.rtHandleScale.x, source.rtHandleProperties.rtHandleScale.y) : Vector2.one;
    // カメラビューポートの設定
    CoreUtils.SetRenderTarget(cmd, destination);
    // Blit
    BlitTexture(cmd, source, viewportScale, mipLevel, bilinear);
}
```

URP14から`_SourceTex`が`_BlitTexture`に名前が変わったので、この名前で
BlitしたRTHandleをマテリアルに伝搬しています。

https://github.com/Unity-Technologies/Graphics/commit/42600cf3bf82a4cda785e52fdc5282495c4ed23e

## 受け渡し構造体

変数部分とほぼ同じなので割愛

```cs
private class PassData
{
    internal Material effectMaterial;
    internal int passIndex;
    internal bool requiresColor;
    internal bool isBeforeTransparents;
    public ProfilingSampler profilingSampler;
    public RTHandle copiedColor;
}
```