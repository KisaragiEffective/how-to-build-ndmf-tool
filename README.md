# How to build my own NDMF tool

## 最小限の構成

最小限の構成は、[NDMFのサンプル](https://github.com/bdunderscore/ndmf/blob/1.5.0/Editor/Samples~/SetViewpointPlugin.cs) に示されているように至ってシンプルです。

1. `public class SetViewpointPlugin : Plugin<SetViewpointPlugin>` というように、自身の型を型引数として持つ`Plugin`を継承するクラスを作る
2. `Configure` メソッドを実装する
    * 内部で`InPhase(BuildPhase)`や`Run`などを用いて処理のTPOを記述する
3. `[assembly: ExportPlugin(typeof(MyPlugin))]` を同じアセンブリのファイルに記述する

3番を忘れるとNDMFは認識しません。

## データを格納する場所

Modular AvatarやAAO: Avatar Optimizerはそれぞれ独自のコンポーネントを持っています。

あなたも独自のコンポーネントを作りたくなることでしょう。コンポーネントを作るためにはRuntimeアセンブリの下に`MonoBehaviour`を継承したクラスを作ります。Unityでコンポーネントを作る方法は他にいい記事があるので割愛します。

VRChat対応を想定する場合はエラーを抑止するために `IEditorOnly` も継承して下さい。

### 付記: AAOと独自コンポーネントの互換性

AAO: Avatar Optimizer は知らないコンポーネントに遭遇すると警告を出します。

この警告を解消するためには[AAOの公式ページ](https://vpm.anatawa12.com/avatar-optimizer/ja/docs/developers/make-your-components-compatible-with-aao/) を見て適切な手法を選択して下さい。

### 属性
適宜 `[SerializeReference]` や `[SerializeField]` や `[Serializable]` を付けて下さい。なお、`readonly`なフィールドには対応していないぽいので注意が必要です。

## アセットの移動や削除
アセットを移動するのはそのアセットが固定パスで参照されていない限り参照が途切れません。なぜならGUIDを用いているからです。

しかし、削除はGUIDの参照でも途切れます。そのため、削除する時は十分考えて下さい。

## アセットの編集
非破壊プラグインということは、つまり上書きしないことが求められます。そのため、すべてのオブジェクトは`Object.Instantiate`で複製する必要があります。このメソッドはシャロークローンなので、必要に応じて再帰的に呼び出す必要があります。

もし参照されているかどうかを確かめたいなら、[nadena.dev.ndmf.util.VisitAssets.ReferencedAssets](https://github.com/bdunderscore/ndmf/blob/1.5.0/Editor/API/Util/VisitAssets.cs#L31) が役に立つでしょう。

### 未検証: Material Variant のほうが安い可能性
シェーダーを変更しないことが約束できるならば、Material Variantを使ったほうが安く済む可能性があります。Material VariantはPrefab Variantと同様、「親」に対する相対的な差分だけを保持するためです。

## ハック: InternalsVisibleTo
TODO: https://learn.microsoft.com/ja-jp/dotnet/api/system.runtime.compilerservices.internalsvisibletoattribute?view=netframework-4.8.1

## Assembly Definition

Assembly Definitionは避けては通れないでしょう。特に、APIを提供する場合には**絶対に避けて通れません**。
(理由: Assembly Definitionを定義しないとAssembly-CSharp-Editorアセンブリに入るが、他のAssembly Definitionを定義して明示的に区分けされたアセンブリはA-C-Eに存在するコードを参照できないため)

TODO:
* Use GUIDs
* Include Platforms (エディタ限定のものはEditorだけにする・AssetBundleのビルドでエラーが起きる・アップロードできない、など)
* `rootNamespace` (Riderのヒューリスティクスなど)

[Unityのドキュメント](https://docs.unity3d.com/ja/2022.3/Manual/ScriptCompilationAssemblyDefinitionFiles.html)を読んで理解するようにして下さい。

作ったそばからauto Referencedを消しておくのがおすすめです。

### Version Defines

パッケージング規則で宣言されている依存バージョンより高いバージョンにしか存在しないコンポーネントやオプショナルなツールを参照するときなどに役に立つ機能です。

例えば、以下のような状況で便利です:
* ツールが要求するVRChat Avatar SDKのバージョンは3.5.0以上だが、3.7.0で導入されたVRChat Constraintsを処理したい
* ツールはVRChat Avatar SDKがなくても動作するが、あったときは追加の機能を有効にしたい
* AAO: Avatar Optimizerが1.7以上なら追加の機能を有効にしたい

## 処理の構成

### 順序関係の構成

NDMFではプラグインの前後関係を設定できる方法があります。

「Modular Avatarのコンポーネントを生成したい」「AAO: Avatar Optimizerの最適化処理を制御したい」など、作ろうとしているものが前後関係を想定する場合は前後関係を設定しなければなりません。

#### フェーズの設定
`InPhase(BuildPhase)` は、プラグインがどの「フェーズ」で呼び出されるべきかを指示します。

フェーズは[`BuildPhase`](https://ndmf.nadena.dev/api/nadena.dev.ndmf.BuildPhase.html)によって指示することができ、v1.5.0現在では実行される順に次の4つがあります。

1. Resolving
2. Generating
3. Transforming
4. Optimizing

#### 他のプラグインとの前後関係

TODO: `BeforePlugin`

この方法はポンデロニウム研究所のしなのちゃん ([BOOTH](https://booth.pm/ja/items/6106863)) に付属する改変補助ツールや EXTENSION CLOTHINGの『変身！まじかる☆ステッキ』用導入補助ツールに用いられています。

## パッケージング

### パッケージング規則

TODO: https://docs.unity3d.com/ja/2022.3/Manual/upm-manifestPkg.html
* 最小Unityのバージョン
* 真に必要なパッケージにおいてサポートされる最小のバージョン (よくある勘違い: 範囲で指定←VPMでしかできない)
  * `optionalDependency` はありません

## UI
IMGUIは辛い上に毎フレーム描画してて遅いのでUIElementsを使いましょう。
