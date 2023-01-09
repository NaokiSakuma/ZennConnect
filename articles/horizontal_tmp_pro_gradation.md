---
title: "TextMeshProでグラデーションをかける"
emoji: "📧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [unity]
published: true
---

# はじめに

TextMeshProで横方向にグラデーションをかけると、1文字ごとにグラデーションがかかってしまいます。
それを、テキストごとにかかるようにしました。

![](/images/tmp_horizontal/result.png =400x)

# 環境

Unity 2022.2.0b16

# スクリプト

## TMPGradation.cs

TextMeshProにグラデーションをかけるものになります。

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using Extensions;
using TMPro;
using UnityEngine;

namespace TMPGradation
{
    [ExecuteAlways]
    public class TMPGradation : MonoBehaviour
    {
        private const int GradientNum = 4;

        private TMP_Text textMeshProText;
        private bool isChange;

        private void OnEnable()
        {
            if (textMeshProText == null)
            {
                textMeshProText = GetComponent<TMP_Text>();
            }

            TMPro_EventManager.TEXT_CHANGED_EVENT.Add(TMPChangeEvent);
        }

        private void OnDisable()
        {
            TMPro_EventManager.TEXT_CHANGED_EVENT.Remove(TMPChangeEvent);
        }

        private void TMPChangeEvent(object obj)
        {
            if ((TMP_Text)obj != textMeshProText)
            {
                return;
            }

            isChange = true;
        }

        private void Update()
        {
            if (!isChange)
            {
                return;
            }

            isChange = false;
            UpdateGradient();
        }

        private void UpdateGradient()
        {
            if (!textMeshProText.enableVertexGradient)
            {
                return;
            }

            var colorMode = GetColorMode();

            if (colorMode is ColorMode.Single or ColorMode.VerticalGradient)
            {
                return;
            }

            // 通常の処理時間の前に、テキストの再生成を強制する関数
            textMeshProText.ForceMeshUpdate();

            var textInfo = textMeshProText.textInfo;
            var characterCount = textInfo.characterCount;

            var gradients = GetVertexGradients(textMeshProText.colorGradient, characterCount, colorMode);

            for (var i = 0; i < characterCount; i++)
            {
                var materialIndex = textInfo.characterInfo[i].materialReferenceIndex;
                var colors = textInfo.meshInfo[materialIndex].colors32;
                var vertexIndex = textInfo.characterInfo[i].vertexIndex;

                if (!textInfo.characterInfo[i].isVisible)
                {
                    continue;
                }

                colors[vertexIndex] = gradients[i].bottomLeft;
                colors[vertexIndex + 1] = gradients[i].topLeft;
                colors[vertexIndex + 2] = gradients[i].bottomRight;
                colors[vertexIndex + 3] = gradients[i].topRight;
            }

            // 変更した頂点の更新
            textMeshProText.UpdateVertexData(TMP_VertexDataUpdateFlags.Colors32);
        }

        private ColorMode GetColorMode()
        {
            return textMeshProText.colorGradient switch
            {
                { topLeft: var topLeft, topRight: var topRight, bottomLeft: var bottomLeft, bottomRight: var bottomRight }
                    when topLeft == topRight && topLeft == bottomLeft && topLeft == bottomRight => ColorMode.Single,
                { topLeft: var topLeft, topRight: var topRight, bottomLeft: var bottomLeft, bottomRight: var bottomRight }
                    when topLeft == bottomLeft && topRight == bottomRight => ColorMode.HorizontalGradient,
                { topLeft: var topLeft, topRight: var topRight, bottomLeft: var bottomLeft, bottomRight: var bottomRight }
                    when topLeft == topRight && bottomLeft == bottomRight => ColorMode.VerticalGradient,
                _ => ColorMode.FourCornersGradient
            };
        }

        private VertexGradient[] GetVertexGradients(VertexGradient vertexGradient, int characterCount, ColorMode colorMode)
        {
            var vertexColors = colorMode switch
            {
                ColorMode.HorizontalGradient => GetHorizontalColors(vertexGradient, characterCount),
                ColorMode.FourCornersGradient => GetFourCornersColors(vertexGradient, characterCount),
                _ => throw new ArgumentOutOfRangeException(nameof(colorMode), colorMode, null)
            };

            var gradients = vertexColors.Chunk(GradientNum).Select(x =>
            {
                var colors = x.ToArray();
                return new VertexGradient(colors[0], colors[1], colors[2], colors[3]);
            });

            return gradients.ToArray();
        }

        private IReadOnlyCollection<Color> GetHorizontalColors(VertexGradient vertexGradient, int characterCount)
        {
            var topLeft = vertexGradient.topLeft;
            var topRight = vertexGradient.topRight;
            var topLeftRatio = (topRight - topLeft) / characterCount;
            var colors = new List<Color>();

            for (var i = 0; i < characterCount; i++)
            {
                colors.Add(topLeft + topLeftRatio * i);
                colors.Add(topLeft + topLeftRatio * (i + 1));
                colors.Add(topLeft + topLeftRatio * i);
                colors.Add(topLeft + topLeftRatio * (i + 1));
            }

            return colors;
        }

        private IReadOnlyCollection<Color> GetFourCornersColors(VertexGradient vertexGradient, int characterCount)
        {
            var step = characterCount * GradientNum;

            var topLeft = vertexGradient.topLeft;
            var topRight = vertexGradient.topRight;
            var bottomLeft = vertexGradient.bottomLeft;
            var bottomRight = vertexGradient.bottomRight;

            var topLeftRatio = (topRight - topLeft) / step;
            var bottomLeftRatio = (bottomRight - bottomLeft) / step;

            var colors = new List<Color>();

            for (var i = 0; i < step; i += GradientNum)
            {
                colors.Add(topLeft + topLeftRatio * i);
                colors.Add(bottomLeft + bottomLeftRatio * (i + 1));
                colors.Add(bottomLeft + bottomLeftRatio * (i + 2));
                colors.Add(topLeft + topLeftRatio * (i + 3));
            }

            return colors;
        }
    }
}

```

## IEnumerableExtension.cs

要素をn個ずつにまとめるものになります。

```cs
using System.Collections.Generic;
using System.Linq;

namespace Extensions
{
    // ReSharper disable once InconsistentNaming
    public static class IEnumerableExtension
    {
        public static IEnumerable<IEnumerable<T>>Chunk<T>(this IEnumerable<T> enumerable, int size)
        {
            while (enumerable.Any())
            {
                yield return enumerable.Take(size);
                enumerable = enumerable.Skip(size);
            }
        }
    }
}
```

# TextMeshProでグラデーションをかける

通常のTextMeshProの機能でグラデーションをかけてみます。
`TextMeshPro - Text`の`Color Mode`から、`Horizontal Gradient`を選択します。

![](/images/tmp_horizontal/color_mode_horizontal.png  =400x)

すると、1文字ずつグラデーションがかかってしまいます。
(Verticalは文字全体にかかります)

![](/images/tmp_horizontal/color_mode_horizontal_result.png =400x)

正攻法としては、シェーダーをDistance Field等に変えて、Textureにアタッチします。
そして、Mappingを変更すると横方向のグラデーションをすることができます。

ただし、

* マテリアルがグラデーションごとに増えてしまう
* シェーダーを変えることがコスパ的に厳しい
* グラデーションテクスチャをつど用意する必要がある

といったデメリットも抱えています。

![](/images/tmp_horizontal/frontal_attack.png)

かといって専用のシェーダーを用意するのも辛いので、今回はc#側で対応しようと思います。

# 不必要な場合はreturnする

inspector上の`Color Gradient`にチェックボックスが入っていないときや、
ColorModeが`Single`、`Vertical Gradient`のときには不要なのでreturnします。

```cs
private void UpdateGradient()
{
    if (!textMeshProText.enableVertexGradient)
    {
        return;
    }

    var colorMode = GetColorMode();

    if (colorMode is ColorMode.Single or ColorMode.VerticalGradient)
    {
        return;
    }

    ...
}

private ColorMode GetColorMode()
{
    return textMeshProText.colorGradient switch
    {
        { topLeft: var topLeft, topRight: var topRight, bottomLeft: var bottomLeft, bottomRight: var bottomRight }
            when topLeft == topRight && topLeft == bottomLeft && topLeft == bottomRight => ColorMode.Single,
        { topLeft: var topLeft, topRight: var topRight, bottomLeft: var bottomLeft, bottomRight: var bottomRight }
            when topLeft == bottomLeft && topRight == bottomRight => ColorMode.HorizontalGradient,
        { topLeft: var topLeft, topRight: var topRight, bottomLeft: var bottomLeft, bottomRight: var bottomRight }
            when topLeft == topRight && bottomLeft == bottomRight => ColorMode.VerticalGradient,
        _ => ColorMode.FourCornersGradient
    };
}
```

# グラデーション部分の作成

## メッシュを強制的に更新する

`ForceMeshUpdate()`を呼んでメッシュを強制的に更新します。
そうしないと後述する`textInfo`等でエラーが発生します。

```cs
private void UpdateGradient()
{
    ...

    // 通常の処理時間の前に、テキストの再生成を強制する関数
    textMeshProText.ForceMeshUpdate();

    ...
}
```

## 色をVertexGradientに変換する

`VertexGradient`はTextMeshPro側で用意されている型になります。

:::details 実装

```cs
[Serializable]
public struct VertexGradient
{
public Color topLeft;
public Color topRight;
public Color bottomLeft;
public Color bottomRight;

public VertexGradient (Color color)
{
    this.topLeft = color;
    this.topRight = color;
    this.bottomLeft = color;
    this.bottomRight = color;
}

/// <summary>
/// The vertex colors at the corners of the characters.
/// </summary>
/// <param name="color0">Top left color.</param>
/// <param name="color1">Top right color.</param>
/// <param name="color2">Bottom left color.</param>
/// <param name="color3">Bottom right color.</param>
public VertexGradient(Color color0, Color color1, Color color2, Color color3)
{
    this.topLeft = color0;
    this.topRight = color1;
    this.bottomLeft = color2;
    this.bottomRight = color3;
}
```

:::

各ColorModeに応じてColorを取得し、それをVertexGradient型に変換しています。

```cs
private VertexGradient[] GetVertexGradients(VertexGradient vertexGradient, int characterCount, ColorMode colorMode)
{
    var vertexColors = colorMode switch
    {
        ColorMode.HorizontalGradient => GetHorizontalColors(vertexGradient, characterCount),
        ColorMode.FourCornersGradient => GetFourCornersColors(vertexGradient, characterCount),
        _ => throw new ArgumentOutOfRangeException(nameof(colorMode), colorMode, null)
    };

    var gradients = vertexColors.Chunk(GradientNum).Select(x =>
    {
        var colors = x.ToArray();
        return new VertexGradient(colors[0], colors[1], colors[2], colors[3]);
    });

    return gradients.ToArray();
}
```


### HorizontalGradientのときのColor取得

テキストの文字数に合わせて割合を出しています。
そうすることより、その文字あたりの色を取得することができます。
HorizontalGradientのときには、VertexGradientの`topLeft`と`topRight`にinspector上で入れた値が入っています。

VertexGradient型は左上、右上、左下、右下の順番に定義されているので
左側はtopLeftに近い色、右側はtopRightに近い色を計算しています。

```cs
private IReadOnlyCollection<Color> GetHorizontalColors(VertexGradient vertexGradient, int characterCount)
{
    var topLeft = vertexGradient.topLeft;
    var topRight = vertexGradient.topRight;
    var topLeftRatio = (topRight - topLeft) / characterCount;
    var colors = new List<Color>();

    for (var i = 0; i < characterCount; i++)
    {
        colors.Add(topLeft + topLeftRatio * i);
        colors.Add(topLeft + topLeftRatio * (i + 1));
        colors.Add(topLeft + topLeftRatio * i);
        colors.Add(topLeft + topLeftRatio * (i + 1));
    }

    return colors;
}

```

### FourCornersGradientのときのColor取得

HorizontalGradientのときと同じような考え方で色を計算しています。
今回は上側だけではなく下側もあるので、`GradientNum`(4)を乗算しています。


```cs
private IReadOnlyCollection<Color> GetFourCornersColors(VertexGradient vertexGradient, int characterCount)
{
    var step = characterCount * GradientNum;

    var topLeft = vertexGradient.topLeft;
    var topRight = vertexGradient.topRight;
    var bottomLeft = vertexGradient.bottomLeft;
    var bottomRight = vertexGradient.bottomRight;

    var topLeftRatio = (topRight - topLeft) / step;
    var bottomLeftRatio = (bottomRight - bottomLeft) / step;

    var colors = new List<Color>();

    for (var i = 0; i < step; i += GradientNum)
    {
        colors.Add(topLeft + topLeftRatio * i);
        colors.Add(bottomLeft + bottomLeftRatio * (i + 1));
        colors.Add(bottomLeft + bottomLeftRatio * (i + 2));
        colors.Add(topLeft + topLeftRatio * (i + 3));
    }

    return colors;
}
```

# TextMeshProに反映

取得したVertexGradientを、TextMeshProに反映します。
TextMeshProの`color32`は左下、左上、右下、右上の順に入っているので、そのとおりに代入します。

`UpdateVertexData`で色を変更したことをTextMeshPro側に伝えます。

```cs
for (var i = 0; i < characterCount; i++)
{
    var materialIndex = textInfo.characterInfo[i].materialReferenceIndex;
    var colors = textInfo.meshInfo[materialIndex].colors32;
    var vertexIndex = textInfo.characterInfo[i].vertexIndex;

    if (!textInfo.characterInfo[i].isVisible)
    {
        continue;
    }

    colors[vertexIndex] = gradients[i].bottomLeft;
    colors[vertexIndex + 1] = gradients[i].topLeft;
    colors[vertexIndex + 2] = gradients[i].bottomRight;
    colors[vertexIndex + 3] = gradients[i].topRight;
}

// 変更した頂点の更新
textMeshProText.UpdateVertexData(TMP_VertexDataUpdateFlags.Colors32);
```

# テキストが変更されたときのみ、グラデーションをかける

Update()等で毎フレーム行う必要がないので、変更されたときのみかけるようにしています。
`TMPro_EventManager.TEXT_CHANGED_EVENT`でTextMeshProの変更時イベントを追加することができるので追加します。
`isChange`がTrueになったときのみ、グラデーションをかけるようにしています。

```cs
private void OnEnable()
{
    if (textMeshProText == null)
    {
        textMeshProText = GetComponent<TMP_Text>();
    }

    TMPro_EventManager.TEXT_CHANGED_EVENT.Add(TMPChangeEvent);
}

private void OnDisable()
{
    TMPro_EventManager.TEXT_CHANGED_EVENT.Remove(TMPChangeEvent);
}

private void TMPChangeEvent(object obj)
{
    if ((TMP_Text)obj != textMeshProText)
    {
        return;
    }

    isChange = true;
}
```

# 結果


https://youtu.be/V-ccsSRu4es

https://youtu.be/28hhEy1H5Gs


# 備考

改行には対応していないので、別途対応が必要です。
