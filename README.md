# How to build my own NDMF tool

1. `[assembly: ExportPlugin(typeof(MyPlugin))]`
2. 処理を書く
3. 前後関係を考える ← ないなら何もしなくていい
5. AAOとの互換性: 自分のコンポーネントを変換したら消す←DestroyImmediate(component)
     * しないと警告
     * Destroyはエラー
6. コンポーネントはIEditorOnlyを継承→VRCSDKの勘違い防止
7. API: asmdefを必要に応じて切る
8. 適宜Version Defines
9. auto referencedを切るとbreaking changeになるので初めから切っておくべき
10. ハック: VisibleInternalsTo
11. 差分を小さくするためにSerializeReference
12. 既存のアセットは常にクローンせよ←Instantiateはシャロークローンなので必要に応じて再帰的に呼び出しが必要
13. 静的アセットはGUIDで参照せよ (Assembly Definition含む)
14. namespace切って←asmdefはrootNamespace推奨
15. 既存ファイル消すのはbreaking changeになりうる
16. 移動は大丈夫
17. IMGUIは辛いのでUIElementsを使う
18. 適宜UPMと**最小**サポートバージョンの定義
19. 真に必須なパッケージを定義
    * 例: VRC以外に対応するならVRCSDKは真に必須ではない

