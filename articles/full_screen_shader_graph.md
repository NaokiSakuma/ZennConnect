---
title: "FullScreen ShaderGraphを使って、ShaderGraphでポストエフェクトをかける"
emoji: "🎆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [unity, urp]
published: true
---

# はじめに

URP14から追加された、`FullScreen ShaderGraph`を触ってみます。

# 環境

Unity 2022.2.0b16
Universal RP 14.0.4
Shader Graph 14.0.4


# FullScreen ShaderGraphを作成する

`Create/Shader Graph/URP/Fullscreen Shader Graph`から作成することができます。

![](/images/full_screen_shader_graph/add_full_screen_shader_graph.png =400x)

または、ShaderGraph上の`Graph Inspector`の`Graph Settings`タブを開き
`Universal/Material/Fullscreen`を選択することでも可能です。


![](/images/full_screen_shader_graph/graph_inspector_full_screen.png =400x)

ノードが以下のようになれば成功です。

![](/images/full_screen_shader_graph/full_screen_default_node.png =500x)

# Universal Renderer Assetの設定

UniversalRendererAssetから`Add Renderer Feature/Full Screen Pass Renderer Feature`を選択します。

![](/images/full_screen_shader_graph/full_screen_renderer_feature.png =400x)

ゲームビューが以下のようになれば成功です。

![](/images/full_screen_shader_graph/default_game_view.png =400x)

その後、作成したFullScreen ShaderGraphのマテリアルを`Pass Material`にアタッチします。

![](/images/full_screen_shader_graph/attach_material.png =600x)

特に何も設定していないので、画面がグレーになります。

![](/images/full_screen_shader_graph/game_view_gray.png =600x)

# Full Screen Pass Renderer Featureのプロパティ

| 名前            | 説明                                                                                                                                                                                                                                                                        | 
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | 
| Name            | Passの名前                                                                                                                                                                                                                                                                  | 
| Pass Material   | RendererFeatureがレンダリングに使うマテリアル                                                                                                                                                                                                                               | 
| Injection Point | Passを追加するタイミング<br>・Before Rendering Transparents : SkyBoxPassの後、TransparentPassの前<br>・Before Rendering Post Processing : TransparentPassの後、PostProcessingPassの前<br>・After Rendering Post Processing : PostProcessingPassの後、AfterRenderingPassの前 | 
| Requirements    | 使用するRendererFeatureのpass<br>・None : なし<br>・Everything : 全て<br>・Depth : 深度<br>・Normal : 法線<br>・Color : カラーデータ、シェーダー内の`_BlitTexture`にコピーする<br>・Motion : モーションベクター<br>                                                         | 
| Pass Index      | Pass Materialで指定したMaterialの使用するpassのindex                                                                                                                                                                                                                        | 

![](/images/full_screen_shader_graph/renderer_feature_property.png =500x)

以下で中身を読んでいるのでそちらも合わせて見ると参考になるかと思います。

https://zenn.dev/sakutaro/articles/full_screen_pass_renderer_feature

# 画面の色を取得する

`URP Sample Buffer`ノードを作り、`Source Buffer`を`BlitSource`にすることによって
画面の色を取得することができます。

![](/images/full_screen_shader_graph/urp_sample_buffer.png =600x)

# グレースケールのポストエフェクトをかける

試しに画面の色を取得して、グレースケールをかけてみます。

![](/images/full_screen_shader_graph/gray_scale_node.png =600x)

以下のように、グレースケールがかかります。

![](/images/full_screen_shader_graph/result.png =600x)