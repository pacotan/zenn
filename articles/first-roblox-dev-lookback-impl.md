---
title: "Roblox開発振り返り：実装編"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

こんにちは、今年の頭からRoblox開発をはじめてみて、約2ヶ月で1つのゲームとして公開まで行ったので、この過程での知見や、不足していることなどをまとめます。


## 作ったゲーム

まずは宣伝！

作ったゲームは以下の「Housing Village」です。

https://www.roblox.com/games/12332171673/Housing-Village

直訳で住宅村、ミニゲームなどでお金を稼いで立派な家を建てるゲームです。

本作品は、Roblox開発者コミュニティの「Devlox」さんと、住宅メーカー「Open House」さんによるRoblox作品コンテストの応募作品です。

コンテストテーマが「HOUSE」でしたので、まず家の建築をする前提で企画・実装していきました。

課金もないですし、ユーザー数でコンテスト有利とかもないので、特に宣伝の意味があるわけじゃないですが、遊んでくれると単純に喜びます。 :)


## 開発振り返り①：Roblox開発では特に「DRY」を意識スべし

とにかくまず思ったことは、RobloxはObjectごとのスクリプトが簡単に作れる分、**同じ実装がそこら中に散らばりやすい**、ということです。

Robloxの場合、大体Scriptをはじめる第一歩は、

- 適当なPartをWorkspaceに置いて
- その下にScriptを作って
- script.Parent でPartにアクセスして
- プログラムでPartを好きに動かす

というのが定番なんじゃないかなと思います。

また、ツールボックスからScript付きのオブジェクトを利用する場合も、こういった構成で利用することが多いと思います。

最初としては問題ないのですが、このノリで、世界にPartをたくさん配置していくとどうなるか。

上述もした通り、そこら中に同じようなコードのコピペがあふれることになります。

こうなると、同じようなギミックに対して、共通の調整をしたい（速度感覚を変えたい、色を変えたい、エフェクトを出したい、など）場合に

対象の全Scriptに対して調整が必要となり、調整、管理、保守のコストが高く、とてもよろしくないということになります。

これへの対処としては、主には

1. 共通処理をModuleScriptにまとめて共通利用する
1. Script自体をCloneで作るような構成にする
1. ModuleScriptのインスタンス化でObjectごとのクラス実装する

あたりかなと思います。以下それぞれ見ていきます。

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

というわけで、振り返り以上に、DRYのためのコーディング方法の解説みたいな内容になっちゃいましたが、常にDRYを意識してこんな実装にしていくと、保守しやすいです。

こうゆうのを怠ると、プロジェクトの最初の進みは良くても、後半になって、あぁ散乱していて、なおしてられん・・ってなりますので常に意識したいこと、だと思いました。

## 振り返り②：Modelクラスインスタンスを扱いやすく

Robloxの大きな特徴1つに、ツリー階層構造でなんでもObjectが構成できてしまうところ、があると思います。

これが、Modelクラスに対しても同様で、通常Modelクラスと聞くと、そのObjectの見た目を包括したような機能をもったイメージをするのですが、

Robloxでは、ほんとにただの「階層構造をまとめるグループ化用のインスタンス」でしかないです。

Model層には、Modelとしての

- 座標
- 表示状態
- カラーやマテリアル設定

などがないのです。正直、表示状態が変えられないのは、驚愕でした。

ですが、リッチなゲームになっていくほど、単純なPartではなくModelを扱うケースは多くなります。

ということで、Modelを簡単に扱える工夫が必須です。

例えば↓のように、Model用utilityのModuleScriptを用意して、配下のPart or Meshのプロパティをまとめて変えられるようにします。

こうすることで、Model全体として、透明にしたり、色を変えたり、といったことが簡単にできます。

```lua
function ModelUtil.SetProperty(model, propName, val)
	if model:IsA("BasePart") or model:IsA("MeshPart") then
		model[propName] = val
	else
		for _, part in ipairs(model:GetDescendants()) do
			if part:IsA("BasePart") or part:IsA("MeshPart") then
				part[propName] = val
			end
		end
	end	
end
```

そこらじゅうで、`model:GetDescendants()`や`model:GetChildren()`しだしていたらそれ自体が危険、と認識しましょう。

## Playerを扱いやすく

Modelを扱いやすくするのに似てますが、Playerについても扱いやすくする必要性が高いです。

というのもインゲームでのPlayer情報は、以下のような大きく2箇所に分離してます。

- Players配下にいる、Player固有情報（StarterPlayerなどの情報）
- Workspace配下にいる、Character情報（Humanoid、Meshなどの情報）

もちろん、PlayerからCharacterへの参照関係はあるので、PlayerからCharacterへアクセスできますが、大きな単位でみるとこう分かれています。

これを、ゲーム実装中に、あの情報がほしいからCharacterにアクセス、パラメータはPlayerか、とか都度意識したくないわけです。

ですので、こちらもPlayerのUtilityやラッパーになるクラスを作ると効率的な開発が可能になるかと思います。

例えば以下のうようなコードです。


インスタンス化可能な、Player処理のラッパーモジュールの実装
```lua
local PlayerWrapper = {}
PlayerWrapper.__index = PlayerWrapper

function PlayerWrapper.new(player : Player)
	local obj = setmetatable({}, PlayerWrapper)
	obj.Player = player
	return obj
end

function PlayerWrapper:SetMoney(value)
	local leaderstats = self.Player:FindFirstChild("leaderstats")
	if leaderstats then
		local money = leaderstats:FindFirstChild("Money")
		if money then
			money.Value = value
		end
	end
end

return PlayerWrapper
```

こうゆうモジュールを作ることでお金を入れたいとき、使う側では↓とだけ書けばよくなります。
```lua
local pw = PlayerWrapper.new(player)
pw:SetMoney(value)
```

## サーバー・クライアントどちらですべき処理かの意識をする

これはRobloxに限らず、オンラインゲームを作っていれば、必須なことですが

**サーバーで処理するべきことなのか、クライアントで処理するべきことなのか、は必ず意識**しないとなりません。

適当に実装していると、簡単にチートできるような


## はじめてのRoblox開発振り返りのまとめ

ということで、振り返りでした。

Robloxは開発をはじめる敷居の低さがすごい分、冗長なコード実装になりやすかったり、

各Objectがシンプルかつ解釈しやすいツリー構造からなる前提のために、逆に操作しにくい部分があったり、

フレームワーク上より少々解釈しにくい部分があったり、

という部分があるので、そこらへんの自分で対処した効率的なコーディングができると、開発効率挙がるな、ととても思った、という振り返りでした。