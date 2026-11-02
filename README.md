# Radeon VII が Linux で `PSP create ring failed` で初期化失敗する → `pcie_aspm=off` で解決した記録

## 結論（TL;DR）

Radeon VII（Vega 20）の初期化が、ログがここまで進んだところで

```
amdgpu: Fetched VBIOS from VFCT
amdgpu: ATOM BIOS: 113-D3600200-106
amdgpu: VRAM: 16368M ...
amdgpu: PSP create ring failed!
amdgpu: PSP firmware loading failed
amdgpu: Fatal error during GPU init
```

と落ちる場合、当方環境では**カーネルパラメータ `pcie_aspm=off` を付けるだけで初期化が通った**。

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash pcie_aspm=off"
# 反映
sudo update-grub
```

代わりに `pcie_aspm.policy=performance` でも可（後述）。逆に、リンクを眠らせる設定（policy が default / powersave / powersupersave）では全て再現した。

> 注意: これは当方環境（下記）での n=1 の結果です。同じログでも原因が別の可能性はあります。ただし試すコストは再起動1回だけなので、同じ場所で止まっている人はまず試す価値があります。

## 環境

| 項目 | 内容 |
|---|---|
| マザーボード | ASRock B550M Pro4（BIOS P3.90 → P4.00 / AGESA 1.2.0.12 で再現） |
| CPU | Ryzen 9 3900X |
| GPU | Radeon VII（gfx906 / Vega 20、VBIOS 113-D3600200-106） |
| OS | Xubuntu 24.04.5 / カーネル 6.8.0-147（26.04 / 7.0、Mint 22.2 / 6.14 でも再現） |
| 備考 | 同じカードは Windows では正常動作 |

## 症状

- 起動すると画面が Ubuntu ロゴで凍結する（simpledrm のロゴが出た後、amdgpu がコンソールを奪って初期化に失敗するため、そのまま固まる）
- 画面以外（ネットワーク、SSH）は正常に上がるので、SSH で `journalctl` を読めば切り分けできる
- 失敗は amdgpu の初期化中、PSP（Platform Security Processor）のファームウェア読み込み段階。ドライバは VRAM 認識や RAS 初期化までは進む

失敗時のジャーナル抜粋:

```
amdgpu 0000:09:00.0: amdgpu: Fetched VBIOS from VFCT
amdgpu: ATOM BIOS: 113-D3600200-106
amdgpu 0000:09:00.0: amdgpu: VRAM: 16368M 0x0000008000000000 - 0x00000083FEFFFFFF (16368M used)
amdgpu 0000:09:00.0: amdgpu: PSP create ring failed!
amdgpu 0000:09:00.0: amdgpu: PSP firmware loading failed
amdgpu 0000:09:00.0: amdgpu: Fatal error during GPU init
```

※ カーネル 7.0 ではもう少し手前の症状（PSP のレジスタ読みが `ffffffff`）だった。

## 試して効かなかったもの

同じ症状の人が遠回りしないよう、先に列挙する。いずれも全く同じ失敗だった。

- カーネルの世代変更（7.0 / 6.8 / 6.14）
- ディストリビューション変更・OS 入れ直し（Xubuntu 26.04 / 24.04.5、Linux Mint 22.2）
- 電源全断＋放電後のコールドブート
- BIOS 設定: ReBAR ON/OFF 両方、Above 4G Decoding ON、CSM OFF、PCIe Gen3 固定
- 基板 BIOS の更新（P3.90 → P4.00）
- カーネルパラメータ `iommu=soft` / `iommu=off`、`mem_encrypt=off`
- VBIOS の更新（113-D3600200-106 が最終版であることを確認済み。更新先が存在しない）

「PSP firmware loading failed」という文面から VBIOS / VFCT の中身を疑うのが自然だが、当方ではそちらは原因ではなかった（VFCT が渡すイメージは参照用 ROM と同一ビルドであることまで照合した）。

## 解決策と切り分け結果

Gemini Pro への相談で出た「ASPM（PCIe のリンク電源管理）が初期化中にリンクを落としているのではないか」という説を検証したら当たっていた。

### パラメータの切り分け

| カーネルパラメータ | 結果 |
|---|---|
| `pcie_aspm=off amdgpu.aspm=0 amdgpu.dpm=0` | ✅ 成功 |
| `pcie_aspm=off amdgpu.aspm=0` | ✅ 成功 |
| **`pcie_aspm=off` のみ** | ✅ **成功** |
| 3つ全て無し（対照実験） | ❌ 失敗 |

決め手は `pcie_aspm=off` 単独。ドライバ側の `amdgpu.aspm=0` / `amdgpu.dpm=0` は無関係だった。

### ASPM ポリシー全レベル

`pcie_aspm=` 自体は `off` / `force` の2値だが、ポリシー（`/sys/module/pcie_aspm/parameters/policy`）には段階があるので全部試した。

| 設定 | 結果 |
|---|---|
| `pcie_aspm=off` | ✅ 成功 |
| `pcie_aspm.policy=performance` | ✅ 成功 |
| `pcie_aspm.policy=powersave` | ❌ 失敗 |
| `pcie_aspm.policy=powersupersave` | ❌ 失敗 |
| 無指定（default） | ❌ 失敗 |

境界は明快で、**リンクを眠らせる設定（L0s / L1 を許可）は全滅、眠らせない設定だけが通る**。

成功時のジャーナル抜粋:

```
kernel: PCIe ASPM is disabled
kernel: [drm] Initialized amdgpu 3.57.0 20150101 for 0000:09:00.0 on minor 1
kernel: fbcon: amdgpudrmfb (fb0) is primary device
```

## 補足

### `off` と `performance` の違い

- `pcie_aspm=off`: ASPM の仕組み自体を無効化する。起動後に変更はできない（ポリシーファイルへの書き込みがエラーになることを実機で確認）
- `pcie_aspm.policy=performance`: 仕組みは有効なまま、リンクを眠らせないポリシーにする。起動後にポリシーを変更できる
- リンクが眠らない点は両方同じで、他の PCIe デバイスにも効く。消費電力も実質変わらない

### 起動後に powersave へ戻すとどうなるか（実機で検証）

performance で起動した後、起動後に `powersave` へ書き換えても、少なくとも数分間はハングしなかった（GPU 温度の読み出し可、カーネルエラーなし）。致命的なのは「起動直後、PSP 初期化と競合する局面で眠りに入ること」らしい。ただし長時間・高負荷・スリープ/復帰の組み合わせは未検証。

### BIOS との関係

- `default` ポリシーは BIOS が設定した ASPM 状態を使うため、BIOS 側の設定が効く
- 基板によっては BIOS 側の ASPM 項目でも同じ効果が得られる可能性がある（当方の B550M Pro4 に該当項目があるかは未確認。Linux 側だけで解決できたので触っていない）

## 未検証・注意点

- 当方環境での n=1 の結果であること
- サスペンド/レジュームとの組み合わせは未検証
- 成功ブートでも SMU の I2C エラーと RAS EEPROM 読み取り失敗（-5）がログに出るが、初期化は完了し実害は未確認

## 謝辞

ASPM 説は Gemini Pro（3.1 Pro）にログを相談した際の提案がきっかけ。全ブートのジャーナル原文や VFCT 照合などの詳細な実験記録は、別途リポジトリに残す予定。
