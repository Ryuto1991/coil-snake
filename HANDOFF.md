# HANDOFF — コイル・バクテリア（試作 v4）

Claude Code / 後任開発者向けの完全引き継ぎ資料。コードの関数名・変数名は現行 `index.html` に準拠。

- **公開プレイ:** https://ryuto1991.github.io/coil-snake/
- **リポジトリ:** https://github.com/Ryuto1991/coil-snake （public / GitHub Pages / `main`）
- **本体:** `index.html` 単一ファイル（約 42KB、バニラ JS + Canvas、外部依存なし）
- **仮タイトル:** コイル・バクテリア（正式名は未定）

---

## 1. コンセプト（プロダクト）

オフライン・シングルプレイヤーの **Vampire Survivors 風（bullet heaven / survivors-like）× slither.io**。

コア体験:
1. 蛇が尻尾をらせんに巻いて敵の群れを囲む（自分の尾と交差するとキャプチャゾーン「包囲！」）
2. 同じ場所を重ねて巻くとチャージが上がる（最大 6、「N重巻き！」）
3. ダッシュでループを締め、内側を一掃する（ダッシュ締めは XP ×2）
4. **頭だけがダメージを受ける**。尾は無敵で敵をすり抜ける
5. 楽しさの核 = **尻尾の動きを計算に入れて敵を誘い、囲んで殺す**こと

テーマ: **顕微鏡下の細胞世界**
- 尻尾が伸びるとカメラがズームアウト（`P.S` / `targetS`）
- 敵スケール: **細菌 → 細胞 / ミトコンドリア → 細胞核**

---

## 2. 決定済み方針

| 項目 | 決定 |
|------|------|
| ライバル蛇 | **なし**（v3 で削除。復活させない） |
| オンライン | **今はなし**。レベルアップのカード選択がゲームをポーズするため。将来マルチにするならインスタント強化ルールが別途必要 |
| ダッシュ倍率 | **2.3×**（v2 の 3.0× から減速。残像付き） |
| モバイル操作 | **短タップでダッシュ**（向きをずらさない）、右下 DASH ボタン、2本指ダッシュも残存 |
| カード | **ダッシュ延長**あり（最大 5 スタック） |
| 序盤レベルアップ | v4 で XP 曲線を上げて静かにした（初回カードは目安 20〜30 秒） |
| 配信順 | まず Web（CrazyGames / itch.io）→ 後で Godot 4 移植して Steam |
| 収益 | スキン中心（Web は広告視聴アンロック、Steam は DLC）。CrazyGames の IAP は Xsolla 招待制 |

個人・金銭情報は本リポジトリに書かないこと（公開リポジトリ）。

---

## 3. バージョン履歴（要約）

| Ver | 内容 |
|-----|------|
| **v1** | 基本ループ成立。ダッシュ 0.3s。オーナーが「めちゃくちゃ楽しい」と評価した原点 |
| **v2** | ダッシュ 0.55s（長押し延長）、ライバル蛇 3 体、アップグレードカード 7 種、包囲チャージ |
| **v3** | ライバル削除。ダッシュ 2.3× + 残像。スケールステージ（細菌→細胞→核）。スマホ DASH ボタン。カード縦並び |
| **v4**（現行） | 短タップダッシュ、**ダッシュ延長**カード、XP 曲線静音化（`xpNeed` 初回 55、式 `round(40+lvl*10+lvl²*1.5)`）、毎レベル尻尾 ×1.105 |

---

## 4. コード構造（`index.html`）

全体は IIFE（`'use strict'; (()=>{ ... })();`）。状態はクロージャ変数。

### 4.1 画面・定数

- `cv` / `ctx` — canvas と 2D コンテキスト
- `W`, `H`, `DPR`, `NARROW`（`W < 620` でモバイル HUD/カード縦並び）
- `resize()` — DPR 制限 2、ビネット用オフスクリーン `vign`
- `Q` / `DEMO` — `URLSearchParams`。`?demo`, `?swarm`, `?tail=`
- `TOUCH` — タッチ検出（実行中に touchstart で true にもなる）
- `TAU`, `rand`, `clamp`, `angDiff`

### 4.2 敵タイプとステージ

```js
STAGES = ['細菌','細胞','細胞核']
STAGE_TAIL = [0, 1000, 1700]   // 尻尾長でのステージ閾値
STAGE_S = [1, 1.43, 2.3]
MAX_TAIL = 3600
```

`ETYPES[]` — 各要素 `{k, st, r, hp, spd, dmg, xp, col, core, glow, rot?}`  
起動時に `bodySprite` / `glowSprite` で `spr`, `fspr`, `gspr`, `gR` を付与。

| stage | k | 概要 |
|-------|---|------|
| 0 | germ / virus / blob | 細菌系 |
| 1 | mito / cell / bigcell | ミトコンドリア・細胞 |
| 2 | nucleus / amoeba / giant | 細胞核・巨大 |

`stageOf(len)`, `targetS(len)` — ステージ判定と目標スケール（`pow(len/640, 0.8)` clamp 1..2.8）  
`checkStage()` — ステージ上昇時バナー「スケールアップ！」、旧敵 HP を削る、クラスタ追加スポーン

### 4.3 入力

変数: `keys`, `mdown`, `mx`, `my`, `mouseUsed`, `dashTouch`, `steerTouch`, `tapCand`

- **キーボード:** WASD/矢印で移動、Space でダッシュ、`1/2/3` でカード、`R` でリトライ、Enter でタイトルから開始
- **マウス:** 移動方向 = 画面中心→カーソル。左クリックで `act('click')` → プレイ中はダッシュ
- **タッチ（v4）:**
  - 右下 `dashBtn()` 内 → 即ダッシュ
  - 2本目の指 → ダッシュ
  - 1本目は `tapCand` で保留。`TAP_MS=200`, `TAP_MOVE=18`
    - 短くほぼ動かず離す → 照準を復元して `dash()`（向きヤンク防止）
    - 動かす / 長押し → `promoteSteer` で操縦
- `act(kind)` — title/over/level/play の分岐エントリ

### 4.4 メイン状態

```js
state ∈ 'title' | 'play' | 'level' | 'over'
```

グローバル配列・値: `P`, `enemies`, `orbs`, `parts`, `texts`, `rings`, `captures`, `ghosts`, `T`, `kills`, `bestCombo`, `spawnT`, `shake`, `hitstop`, `choices`, `overT`, `pendingLv`, `flashT`, `banner`, `stageT`

**プレイヤー `P`（`reset` で初期化）:**

| フィールド | 意味 | 初期値めやす |
|-----------|------|-------------|
| `x,y,a` | 位置・向き | 0 |
| `spd` / `turn` | 移動速度 / 旋回 | 195 / 4.0 |
| `hp` / `maxHp` / `inv` | HP・無敵時間 | 100 / 100 / 0 |
| `tailLen` | 尻尾長 | `?tail=` or 640 |
| `dashCd` / `dashT` / `dashCdT` / `dashAge` | CD・残り・CD残り・経過 | 2.2 / 0 / … |
| `dashBase` / `dashHoldMax` / `dashStacks` | 基本時間・長押し上限・延長スタック | 0.55 / 0.85 / 0 |
| `ghostT` | 残像間隔タイマ | |
| `trail[]` | `{x,y,s}` 軌跡。`serial` / `minSerial` / `loopCd` | |
| `S` / `stage` | 現在スケール・ステージ | |
| `pulseR/Dmg/Int/T` | 自動パルス | 72 / 1 / 1.1 / … |
| `magnet` / `blast` | XP 吸引半径 / 締め爆発倍率 | 95 / 1 |
| `lvl` / `xp` / `xpNeed` | レベル | 1 / 0 / **55** |

### 4.5 ループ検出・包囲・締め

幾何ヘルパ: `segInt`, `polyArea`, `centroid`, `inPoly`, `bbox`

- `advanceTrail(dt)` — 頭が十分動いたら軌跡点を追加。新しい辺と古い辺の交差を `segInt` で検出し、面積が閾値超えなら `onLoop(poly)`。`loopCd` 0.1s で連発抑制。尻尾長超過分は先頭を切り詰め
- `onLoop(poly)` — 近い未締めキャプチャがあれば `charge = min(6, charge+1)`（「N重巻き！」）、なければ新規 `{poly,c,rad,charge:1,timer:CAPT_TIME=1.6,cinchT:-1,dash:false}`（「包囲！」）。ポリゴン内の敵に `e.cap = cap`
- `dash()` — CD 中でなければ `dashT = dashBase`、未締めキャプチャ全てに `cinchT=0`, `dash=true`
- キャプチャ更新（`update` 内）: `cinchT<0` なら `timer` 減少 → 0 で自動締め（`dash=false`）。`cinchT>=0` なら 0.2s で `explode(cap)`
- `explode(cap)` — 内側キル + 衝撃波。XP 倍率 `(dash?2:1)*(1+0.35*(charge-1))`。コンボ文字・hitstop・shake

### 4.6 戦闘・スポーン・XP

- `pickType()` / `spawnCluster(n)` — ステージと経過時間で敵種ウェイト。HP は `1+T/240` スケール。キャップ 520/400/320
- `hurt` / `kill` — 撃破で XP オーブ `orbs`（溢れ時は直接 `addXP`）
- `addXP(v)` — レベルアップ時 `xpNeed = Math.round(40 + lvl*10 + lvl*lvl*1.5)`、`pendingLv++`
- 頭接触: 極小敵（`r < 7*S`）は餌として即キル。ダッシュ中はダメージ押し。通常は `P.hp -= e.t.dmg`、`inv=0.75`、HP0 で `over`
- 自動パルス: `pulseT` 周期で範囲ダメージ
- 空間ハッシュ: `buildGrid` / `CELL`（敵同士の押し合い）

### 4.7 アップグレードカード `UPG[]`

`openLevel()` がプールからランダム 3 枚。`pick(i)` で適用し、**毎回** `tailLen *= 1.105`（`MAX_TAIL` まで）。`DEMO` 時は自動選択。

| n | 効果 `f` |
|---|----------|
| 尾が伸びる | `tailLen *= 1.18`（MAX_TAIL まで） |
| ダッシュ回復 | `dashCd *= 0.82`（下限 0.6） |
| **ダッシュ延長** | `dashStacks++`（最大 5）、`dashBase += 0.12`、`dashHoldMax += 0.185` → 最大 0.55→1.15s 前後 |
| パルス強化 | `pulseR += 16`、`pulseDmg += 1`、`pulseInt *= 0.92`（下限 0.45） |
| 締め爆発 | `blast += 0.25` |
| 鞭毛ブースト | `spd *= 1.08`、`turn *= 1.08` |
| DNA吸引 | `magnet += 45` |
| 細胞膜 | `maxHp += 20`、HP 全回復 |

カード UI: `cardRects()`（`NARROW` なら縦 3 枚）、`cardAt(x,y)`、`levelScreen()`

### 4.8 移動・ダッシュ（`update`）

- 目標角度: WASD / マウス相対（画面中心基準）/ `DEMO` はサイン波操縦
- 速度: `spd * S * (dashT>0 ? 2.3 : 1)`
- ダッシュ中: 長押し（`mdown` or Space）かつ `dashAge < dashHoldMax` なら `dashT` を維持。`ghosts` に残像

### 4.9 カメラ・描画

- `P.S` が `targetS(tailLen)` に追従。ワールドは `Z = 1/S` でスケール変換
- `render()` — 背景モート → グリッド → キャプチャ → オーブ → 敵 → 尾 → リング → 残像 → 頭 → パーティクル → スクリーン空間（テキスト・ビネット・バナー・HUD）
- `hud()` — XP バー、HP、Lv、撃破、最大コンボ、タイマー、スケール進行、DASH ゲージ、巻きチャージ（最大 6 マス）
- `titleScreen` / `levelScreen` / `overScreen`
- `drawHead`, `drawBody`, スプライト系

### 4.10 ゲームループ

```js
frame(now) {
  dt = min(0.033, …)
  if state==='play': hitstop 中でなければ update(dt); updateFx(dt)
  else if over: overT += dt; updateFx
  else if level: updateFx(0)  // ポーズ相当
  render(); requestAnimationFrame(frame)
}
```

デバッグ: `window.__g` — `s`（スナップショット getter）、`P`、`dash`、`addXP`

---

## 5. ロードマップ（目安）

| 時期 | 内容 |
|------|------|
| **〜2026年12月中旬** | プロトタイプ磨き込み（バランス・スマホ実機・ジュース） |
| **12月中旬〜** | 友達テスト（本 Pages リンクを LINE で共有） |
| **2026年12月末** | CrazyGames **Basic Launch** 申請（目標: 平均プレイ 10分+、D1 リテンション 10–15%）。itch.io も並行 |
| **2027年1–2月** | CrazyGames **Full Launch**（SDK 組み込み）。メトリクスが良ければ |
| **以降** | **Godot 4 + GDScript** へ移植（SteamPipe + GodotSteam） |
| **2027年4月上旬** | Steam ページ公開（Coming Soon）。Steam Direct 手数料・審査リードタイムに注意 |
| **Next Fest** | 2027年6月開催想定。登録締切 **2027-04-25**（要再確認） |
| **〜2027年9月** | Steam 発売目安 |
| **任意・後続** | Steam Networking P2P で 2〜4 人協力（要・ポーズしない強化ルール） |

Web は CrazyGames 非独占 Basic から。ブラウザ独占オプション（+50% など）はメトリクス見て判断。Poki は長期独占になりやすいので現状は CrazyGames 優先。

---

## 6. マネタイズ方針

- **スキン**が主軸（コピーされやすいゲーム性のため）
- Web: 広告視聴でスキン解除（CrazyGames SDK 後）
- Steam: スキンを DLC 販売
- CrazyGames のアプリ内課金は **Xsolla 招待制**（最初から当てにしない）
- itch.io は配布・フィードバック用。本命の広告レベニュは CrazyGames 側

---

## 7. Open TODO / アイデア

- [ ] **ボス** — 複数重巻きが必要な大型（現状の giant 以上の専用エンティティ）
- [ ] **実機バランス** — 細菌→細胞→核の各帯でテンポ・難度を iPhone/Android 実機で調整（現状はヘッドレス相当のエミュ確認が中心）
- [ ] **友達テスト設問** — 何分遊んだか / また遊びたいか / どこが気持ちよかったか / どこで死にたくなったか
- [ ] **正式ゲーム名** 決定
- [ ] ストア用アセット（アイコン、カバー、GIF/動画、短い説明文）
- [ ] サウンド / ヒットストップや画面効果のジュース強化
- [ ] ダッシュ・XP・ステージ閾値の再バランス（初回レベルアップ感、細胞到達 3〜4 分目安は維持しつつ調整）
- [ ] （将来）ポーズしないインスタント強化ルールの設計メモ — オンライン化の前提

---

## 8. 既知の限界

- **実機未検証が残っている** — ヘッドレス / エミュ中心。実 iPhone・Android のタッチ遅延・セーフエリア・ブラウザ差はオーナー確認待ち
- バランスは**第一パス**。v4 の XP 静音化・尻尾成長は自動プレイでざっくり合わせた値
- 音なし、永続セーブなし、スキンなし（プロトタイプ範囲）
- シングルファイル前提なので、肥大化したら Godot 移植タイミングを優先してよい

---

## 9. Claude Code への作業依頼の出し方（例）

```
CLAUDE.md と HANDOFF.md を読んでから作業して。
変更は index.html のみ。外部ライブラリ禁止。UI は日本語のまま。
終わったら ?demo と 390×844 タッチ相当で JS エラーなし・60fps 前後を確認し、
main に commit & push（Pages 更新）までやって。
```

具体タスク例:
- 「ボス敵を stage2 に追加。包囲 3 重以上でないと倒せない」
- 「実機でダッシュが反応しにくいので TAP_MS / TAP_MOVE を調整」
- 「細胞ステージ到達が遅いので STAGE_TAIL と尾成長を見直す」

---

## 10. クイックリファレンス

| 項目 | 値 |
|------|-----|
| ダッシュ速度倍率 | 2.3 |
| ダッシュ基本 / 長押し上限 | 0.55s / 0.85s（延長カードで伸びる） |
| ダッシュ CD | 2.2s（カードで短縮、下限 0.6） |
| 包囲タイマー | `CAPT_TIME = 1.6` |
| チャージ上限 | 6 |
| ダッシュ締め XP | ×2（さらに多重ボーナス） |
| 初回 xpNeed | 55 |
| xpNeed 式 | `round(40 + lvl*10 + lvl²*1.5)` |
| 毎レベル尻尾 | ×1.105 |
| ステージ閾値（尾） | 0 / 1000 / 1700 |
| テスト URL | `?demo` `&swarm` `&tail=1900` |
