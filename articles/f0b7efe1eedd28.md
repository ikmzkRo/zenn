---
title: "Pythonで作るマークルツリー"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "solidity", "ethereum", "markletree", "crypto"]
published: false
---

# マークルツリーとは何か
マークルツリーとは、効率的な包含証明のために設計されたデータ構造の一つです。
完全なブロックチェーンデータを保持できないスマホのような軽量クライアントは、このマークルツリーを用いてブロックヘッダのマークルルートフィールドを確認することで、トランザクションの整合性を確認し、取引の安全性を確認できます。


マークルツリーの構成は[サトシナカモトの論文](https://bitcoin.org/bitcoin.pdf)が分かりやすいです。
![merkletree](/images/merkletree.png)


マークルツリーを構築するうえで必要なものは以下の2点です。
1. アイテムの順序付きリスト
2. 暗号学的ハッシュ関数


アイテムはブロック内のトランザクションで、以下が順序付きリストの例です。暗号学的ハッシュ関数は `hashlib.sha256` を使用します。
```
hex_hashes = [
    "9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb",
    "5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b",
    "82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05",
]
```


マークルツリーを構築するために以下のアルゴリズムを利用します。
1. ハッシュ関数を使用して、順序付きリストのアイテム全てをハッシュ化します
2. 1つのハッシュになれば、処理を完了します
3. 2とならない場合、ハッシュの個数が奇数なら末尾のハッシュを複製して偶数にします
4. ハッシュを順番にペアにして、連結したものをハッシュ化して親レベルのハッシュを生成します。※`親レベルのハッシュの個数は半分になります`
5. 2に戻ります


つまり、`マークルツリーはアイテムの順序付きリスト全体をただ1つのハッシュにするというアイデア`です。


マークルツリーに関する用語を整理します。
- リーフ：最下位のハッシュリスト
- 内部ノード：それ以外のハッシュリスト
- マークルルート：最高位のハッシュ
- マークルペアレント: ハッシュを順番にペアにして、連結したものをハッシュ化したもの
- マークルペアレントレベル: 各ペアの親ハッシュを得るための条件 (ex: 順序付きハッシュリストが３つ以上存在すること)
- マークルパス: マークルルートを求める最短経路

# マークルツリーを構築する
:::details マークルペアレントを求める
```py: main.py
from helper import hash256

leftHash = bytes.fromhex('c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5')
rightHash = bytes.fromhex('c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5')
parent = hash256(leftHash + rightHash)

print(parent.hex())
```
```
8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
```
:::

ハッシュをペアにして連結し新たなハッシュを生成していく際、２つのペアとなるハッシュについてはインデックス順に左ハッシュ(L)・右ハッシュ(R)と呼び、左右のハッシュを連結(H)した結果を親ハッシュもしくはマークルペアレント(P)と呼びます。
```
P = H(L||R)　※||は連結
```

:::details マークルペアレントレベルに対応する新たなハッシュリストを求める
```py: main.py
# 順序付きハッシュリストをインプットしてハッシュ化
hex_hashes = [
    'c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5',
    'c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5',
    'f391da6ecfeed1814efae39e7fcb3838ae0b02c02ae7d0a5848a66947c0727b0',
    '3d238a92a94532b946c90e19c49351c763696cff3db400485b813aecb8a13181',
    '10092f2633be5f3ce349bf9ddbde36caa3dd10dfa0ec8106bce23acbff637dae',
]
hashes = [bytes.fromhex(x) for x in hex_hashes]

# ハッシュ数が奇数なら末尾のハッシュ値を複製して連結
if len(hashes) % 2 == 1:
    hashes.append(hashes[-1])

# マークルペアレントレベル
parent_level = []

# ハッシュ値を２つずつ選択してペアを作成
for i in range(0, len(hashes), 2):
    parent = merkle_parent(hashes[i], hashes[i+1])
    parent_level.append(parent)
for item in parent_level:
    print(item.hex())
```
```
8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
7f4e6f9e224e20fda0ae4c44114237f97cd35aca38d83081c9bfd41feb907800
3ecf6115380c77e8aae56660f5634982ee897351ba906a6837d15ebc3a225df0
```
:::
ハッシュの個数が奇数なら末尾のハッシュを複製して偶数にし、ハッシュをペアにして連結し新たなハッシュを生成します。試験的に、５つのハッシュリストを用意すれば末尾のハッシュを複製してハッシュリストを偶数に整理した後、ペアを作って連結した結果、要素数３のハッシュリストが出来上がるはずです。


:::details マークルルートを求める
```py: main.py
from helper import merkle_parent_level
hex_hashes = [
    'c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5',
    'c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5',
    'f391da6ecfeed1814efae39e7fcb3838ae0b02c02ae7d0a5848a66947c0727b0',
    '3d238a92a94532b946c90e19c49351c763696cff3db400485b813aecb8a13181',
    '10092f2633be5f3ce349bf9ddbde36caa3dd10dfa0ec8106bce23acbff637dae',
    '7d37b3d54fa6a64869084bfd2e831309118b9e833610e6228adacdbd1b4ba161',
    '8118a77e542892fe15ae3fc771a4abfd2f5d5d5997544c3487ac36b5c85170fc',
    'dff6879848c2c9b62fe652720b8df5272093acfaa45a43cdb3696fe2466a3877',
    'b825c0745f46ac58f7d3759e6dc535a1fec7820377f24d4c2c6ad2cc55c0cb59',
    '95513952a04bd8992721e9b7e2937f1c04ba31e0469fbe615a78197f68f52b7c',
    '2e6d722e5e4dbdf2447ddecc9f7dabb8e299bae921c99ad5b0184cd9eb8e5908',
    'b13a750047bc0bdceb2473e5fe488c2596d7a7124b4e716fdd29b046ef99bbf0',
]
hashes = [bytes.fromhex(x) for x in hex_hashes]
current_hashes = hashes
while len(current_hashes) > 1:
    current_hashes = merkle_parent_level(current_hashes)
print(current_hashes[0].hex())
```
```
acbcab8bcc1af95d8d563b77d24c3d19b18f1486383d75a5085c4e86c86beed6
```
:::
while ループの繰り返し条件としてハッシュリストの長さが1より大きいかどうかを判定し、長さが1になればループを終了します。








# おまけ


# 参考
https://bitcoin.org/bitcoin.pdf
https://books.google.co.jp/books/about/%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0_%E3%83%93%E3%83%83%E3%83%88%E3%82%B3%E3%82%A4%E3%83%B3.html?id=FagHzgEACAAJ&source=kp_book_description&redir_esc=y
https://github.com/learn-co-students/session7-jsong-programming-blockchain-demo/blob/master/index.ipynb
https://alis.to/mozk/articles/3PYokoOWP7XY
https://github.com/sskgik/merkletree_with_python/blob/main/merkletree.py
https://github.com/sskgik/Merkletree/blob/master/Program.cs