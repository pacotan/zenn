---
title: "Robloxゲーム開発では特段DRYを意識すべき"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

こんにちは、今年の頭からRoblox開発をはじめてみて、約2ヶ月で1つのゲームとして公開まで行ったので、この過程での知見や、不足していることなどをまとめます。


## Roblox開発では特に「DRY」を意識すべき

「**Don't Repeat Yourself.（繰り返すな。）**」

プログラミングの原則の1つともされる「DRY」、Roblox開発ではことさらこの原則を意識すべきと感じます。

というのも、RobloxではObject単体ごとにScriptが簡単に作れます。

大体初めてのScript作成手順が、Partの下にScriptを生成して、`script.Parent`でPartにアクセスしてPartを動かす、かと思います。

当然ながら、これはこのPartというObject専用のプログラムになっています(精確にはそうなることが多いはず)。

また、ツールボックスからScript付きのものを利用する場合も同様に、ObjectにScriptがぶら下がっていて動く、ということが多いと思います。

最初のうちや、テスト段階なら問題ないですが、いざゲーム本実装を進める際にもこのノリでScriptつきObjectを量産していくとどうなるか。

はい、**同じようなコードがそこら中にあふれた状態**となります。

こうなると、複数のObjectに対して共通の調整をしたい（Script上での速度変更、エフェクト追加、など）の場合に、**対象の全Scriptに対して手を入れる必要があり、調整作業コストがめちゃ高い**ということになります。

調整コストだけじゃなく、**変更漏れが起きないかや、今後の維持の難しさなど、管理・保守コストも高い**状態となります。

DRYはプログラマならある程度意識をもっている人も多いと思いますが、**Robloxでは最初のワークフローやScript配置の気軽さの結果、コードの複製が起きやすい、という印象なので注意したい**ですね、というのが本記事の意図となります。


## コードの複製をおこさないための対処法


このDRYを起こさない、すなわちコード複製を抑えるための対処方法が、今考える限りは

1. 共通処理をModuleScriptにまとめて共通利用する
1. Script自体をCloneで作るような構成にする
1. ModuleScriptのインスタンス化でObjectごとのクラス実装する

あたりかなと思ってます。以下それぞれ見ていきます。

### 1.共通処理をModuleScriptにまとめて共通利用する

1つめ、ModuleScriptの活用は、例えばTweenで床を動かすとした場合

```lua
function ObjectManager.MovePart(part : BasePart, move : Vector3)
	local tweenInfo = TweenInfo.new(2, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut, -1, true)
	local goal = { Position = part.Position + move }
	local tween = TweenService:Create(part, tweenInfo, goal)
	tween:Play()
end
```
ってObjectManagerというModuleScriptにTweenで動かす処理をまとめておけば使う側は

```lua
ObjectManager.MovePart(script.Parent, Vector3.new(0, 0, 10))
```
みたいな動く量だけを設定するだけの1行で済ませることができ、移動の詳細調整（カーブパラメータ変更など）はModuleScirptでまとめて変えられるわけでし、様々なところからこの処理を使い回すこともできます。

&nbsp;

### 2.Script自体をCloneで作るような構成にする

2つめ、Script自体をCloneで作る場合、やりたい処理はReplicatedStorageにおいておきます。

そして、全体管理のScriptから、付与したいObject配下にScriptをCloneしてくっつけてあげます。↓こんな感じ。

```lua
for _, movePart in ipairs(workspace.MoveParts:GetChildren()) do
	local moveScript = game.ReplicatedStorage.MoveScript:Clone()
	moveScript.Parent = movePart
end
```

こうすれば、全く同じプログラムを多くのObjectに適用できますが、Scriptファイルは1つで済んでますね。

&nbsp;

### 3.ModuleScriptのインスタンス化でObjectごとのクラス実装する

3つめは、一番現代的なObject指向、Componentベース的な作り方。

コード的にはまずインスタンス化できるModuleScript（実装関数をメタテーブルに与えたtable）をnewで作れるように用意。

```lua
local TweenService = game:GetService("TweenService")
local ObjectMover = {}

ObjectMover.__index = ObjectMover

function ObjectMover.new(part : BasePart)
	local obj = setmetatable({}, ObjectMover)
	obj.Part = part
	return obj
end

function ObjectMover:Move(move : Vector3)
	local tweenInfo = TweenInfo.new(2, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut, -1, true)
	local goal = { Position = self.Part.Position + move }
	local tween = TweenService:Create(self.Part, tweenInfo, goal)
	tween:Play()
end

return ObjectMover
```
使う側ではnewしてそのModuleScriptを生成（インスタンス化）、その際対象となるPartをもたせます。

```lua
local mover = ObjectMover.new(movePart)
mover:Move(Vector3.new(0, 10, 0))
```

そうすれば、構成としては、そのPartがObjectMoverというクラスで実装されたインスタンス、というように振る舞うようになります。

ここらへんは、ある程度luaプログラムに慣れてないと理解しにくい部分ではありますが、こんなやり方もある、とまずは覚えると良いかと思います。

&nbsp;

## まとめ：大事なのはDRYの意識

というわけで、Robloxでカオスを起こさないために、最も意識すべきと思うDRYのための意識と、具体的な対処方法を挙げました。

Roblox(Lua)の開発歴はまだまだ浅いので、他にもなにか手段があるやもしれません。

ただ、とにかく大事になのは、「**つねにDRYの意識をもってコーディングする**」ということです。

同じようなScriptが複数出来てるなとか、コードのコピペ箇所が多いなとか、それに気づいて、それに対処しようとすること自体が大事、手段は二の次です。

Script単体で直接書く方が楽なことも多いので、疲れているときなど楽な方楽な方へ流れたくなるものですが、結局後で直すみたいな二度手間になるものです。

**可能な限りDRY意識して、調整、管理、保守、しやすいコードにしていきましょうっ**。

ということで、最後に先日作ったRobloxゲームを置いておくので、 よかったら、遊んだり、フォローしたり、いいねしたりしてくれると泣いて喜びます！

https://www.roblox.com/games/12332171673/Housing-Village

Twitterでも紹介しております

https://twitter.com/keita0x12/status/1632696529399193600

