---
title: "Robloxゲーム開発で大事だと思うこと②「Utility」"
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
