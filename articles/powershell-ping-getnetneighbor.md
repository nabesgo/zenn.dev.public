---
title: "Powershellで並列pingとarp -a"
emoji: "🖥️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['PowerShell']
published: true
---

# 導入

- ネットワークトラブル時に同じセグメントにいる端末をザっと知りたい時がある。
- 一定のアドレス範囲に並列でpingを打ち(結果が早く得られる)、arpテーブルを見て近隣の端末を知る、をpowershellでやる。

## recipe

- 192.168.1.1～254の場合。

```powershell
  Test-Connection @(1..254 | %{"192.168.1.$($_)"}) -AsJob -Count 1 | Wait-Job | Receive-Job | Out-Null
  Get-NetNeighbor -AddressFamily IPv4 | ?{$_.State -ne "Unreachable"}
```
