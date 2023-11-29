---
title: "Pythonで作るマークルツリー"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "solidity", "ethereum", "markletree", "crypto"]
published: false
---

# マークルツリーとは何か
マークルツリーは、包含証明の効率的なためにデザインされたデータ構造の一例です。軽量クライアントなど、完全なブロックチェーンデータを保持できない状況では、マークルツリーを使用してブロックヘッダのマークルルートフィールドを確認することで、トランザクションの整合性を検証し、取引の安全性を確保できます。

この技術は、データのサマリを作成し、そのサマリと元データの整合性を確認することで、データの改ざんが行われていないかを検証するものです。特に、部分的なデータしか知らなくても検証が可能である点が重要です。

このメカニズムは、ウォレットを使用する際に役立ちます。ウォレットがブロックチェーン上の全トランザクションデータを持つと重くなるため、代わりにデータのサマリだけを保持します。ただし、これだけでは自身のトランザクションデータを見ることができません。そこで、自身のアドレスのみを使用して改ざんの有無を検証し、その後元データから必要なトランザクションデータを取得します。

マークルツリーの構成は[サトシナカモトの論文](https://bitcoin.org/bitcoin.pdf)が分かりやすいです。
![merkletree](/images/merkletree.png)


マークルツリーを構築するうえで必要なものは以下の2点です。
1. アイテムの順序付きリスト: マークルツリーは、ブロック内のトランザクションなどのアイテムが順序付きリストとして提供されることで構築されます。これにより、ツリーの構造が形成されます。
2. 暗号学的ハッシュ関数: マークルツリーでは、各ノードのデータは暗号学的ハッシュ関数によってハッシュ化されます。通常、hashlib.sha256のような安全なハッシュ関数が使用され、それによってツリー内の各ノードが一意のハッシュ値を持つようになります。

これらの要素を組み合わせて、マークルツリーを構築することが可能です。順序付きリストからアイテムを取り、それぞれのアイテムに対してハッシュ関数を適用し、ツリーの上位レベルへと進んでいくことで、最終的に一つのルートハッシュ（マークルルート）が得られます。これによって、マークルツリーを使用したデータの整合性を検証することができます。

マークルツリーを構築するために以下のアルゴリズムを利用します。
1. ハッシュ関数を使用して、順序付きリストのアイテム全てをハッシュ化する。
2. ハッシュの個数が1つになれば、処理を完了する。
3. ハッシュの個数が2つ以上でかつ奇数なら、末尾のハッシュを複製して偶数にする。
4. ハッシュを順番にペアにして、連結したものをハッシュ化して親レベルのハッシュを生成する。親レベルのハッシュの個数は半分になる。
5. 生成されたハッシュが1つでなければ、2に戻る。

このアルゴリズムによって得られるマークルルートは、マークルツリー全体の整合性を確認する際に極めて重要な要素です。マークルツリーは、アイテムの順序付きリスト全体を単一のハッシュにまとめる手法であり、このハッシュが変更されていない限り、元のデータに変更が加えられていないことを示します。

以下はマークルツリーに関する用語の整理です。
1. リーフ：マークルツリーの最下位に位置するノードで、元のデータ（例: トランザクション）を表します。各リーフはハッシュ関数によってハッシュ化されています。
2. 内部ノード： リーフ以外のノードで、マークルツリーの中間レベルに位置します。内部ノードはリーフや他の内部ノードのハッシュから計算されます。
3. マークルルート：マークルツリーの最上位にあるノードで、全体の整合性を示す単一のハッシュです。これが変更されると、マークルツリー全体の変更が検知されます。
4. マークルペアレント:  二つのハッシュを取り、それらを連結してハッシュ化したものです。この操作により、新しいノード（親ノード）が生成されます。
5. マークルペアレントレベル: マークルツリーにおいて、各ペアの親ハッシュを得るための特定のレベル。通常、順序付きハッシュリストが3つ以上存在する場合にマークルペアレントレベルが形成されます。
6. マークルパス: マークルツリーのルートからリーフまでの最短経路に含まれるハッシュ値の列。これを使用することで、特定のリーフがマークルルートに正しく結びついていることを検証できます。

# マークルツリーを構築する
:::details マークルペアレントを求める
```py: main.py
from helper import hash256

leftHash = bytes.fromhex('c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5')
rightHash = bytes.fromhex('c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5')
parent = hash256(leftHash + rightHash)

print(parent.hex())
```
結果:
```
8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
```
:::
ハッシュをペアにし、新しいハッシュを生成するプロセスでは、通常、左側のハッシュ（L）と右側のハッシュ（R）として、それらを連結した結果を親ハッシュ（P）またはマークルペアレントと呼びます。
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
結果:
```
8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
7f4e6f9e224e20fda0ae4c44114237f97cd35aca38d83081c9bfd41feb907800
3ecf6115380c77e8aae56660f5634982ee897351ba906a6837d15ebc3a225df0
```
:::
ハッシュの個数が奇数の場合、末尾のハッシュを複製して偶数に調整し、それからハッシュをペアにして連結して新しいハッシュを生成する手順は、マークルツリーの構築において一般的な処理です。例として、5つのハッシュから始め、末尾のハッシュを複製してハッシュリストを偶数に整理した後、ペアを作成して連結すると、最終的に要素数3のハッシュリストが得られることが期待されます。


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
結果:
```
acbcab8bcc1af95d8d563b77d24c3d19b18f1486383d75a5085c4e86c86beed6
```
:::
while ループの繰り返し条件としてハッシュリストの長さが1より大きいかどうかを判定し、長さが1になればループを終了します。

:::details マークルツリーの構造を把握する
```py: main.py
import math

# マークルツリーの構造を把握
from helper import validate_merkle_root

# リーフ数は16（軽量クライアントが最初に必要な情報）
total = 16

# 二分木探索方法に必要な深さを求める（ここではlog2^16=4）
max_depth = math.ceil(math.log(total, 2))
merkle_tree = []
for depth in range(max_depth + 1):
    num_items = math.ceil(total / 2**(max_depth - depth))
    level_hashes = [None] * num_items
    merkle_tree.append(level_hashes)
for level in merkle_tree:
    print(level)
```
結果:
```
[None]
[None, None]
[None, None, None, None]
[None, None, None, None, None, None, None, None]
[None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None]
```
:::
軽量クライアントがフルノードから受信する情報の中で、必要なのはリーフ数（トランザクション数）です。これは、マークルツリーの構造を理解するための基本的な情報であり、リーフ数を知ることでマークルツリーの深さを計算できます。具体的には、マークルツリーの深さは log2^(リーフ数) で把握できます。そして、この深さ情報を for ループの繰り返し条件に利用しています。

:::details ハッシュリストを指定してマークルツリーの構造を把握する
```py: main.py
from merkleblock import MerkleTree
from helper import merkle_parent_level
hex_hashes = [
    "9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb",
    "5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b",
    "82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05",
    "507ccae5ed9b340363a0e6d765af148be9cb1c8766ccc922f83e4ae681658308",
    "a7a4aec28e7162e1e9ef33dfa30f0bc0526e6cf4b11a576f6c5de58593898330",
    "bb6267664bd833fd9fc82582853ab144fece26b7a8a5bf328f8a059445b59add",
    "ea6d7ac1ee77fbacee58fc717b990c4fcccf1b19af43103c090f601677fd8836",
    "457743861de496c429912558a106b810b0507975a49773228aa788df40730d41",
    "7688029288efc9e9a0011c960a6ed9e5466581abf3e3a6c26ee317461add619a",
    "b1ae7f15836cb2286cdd4e2c37bf9bb7da0a2846d06867a429f654b2e7f383c9",
    "9b74f89fa3f93e71ff2c241f32945d877281a6a50a6bf94adac002980aafe5ab",
    "b3a92b5b255019bdaf754875633c2de9fec2ab03e6b8ce669d07cb5b18804638",
    "b5c0b915312b9bdaedd2b86aa2d0f8feffc73a2d37668fd9010179261e25e263",
    "c9d52c5cb1e557b92c84c52e7c4bfbce859408bedffc8a5560fd6e35e10b8800",
    "c555bc5fc3bc096df0a0c9532f07640bfb76bfe4fc1ace214b8b228a1297a4c2",
    "f9dbfafc3af3400954975da24eb325e326960a25b87fffe23eef3e7ed2fb610e",
]
tree = MerkleTree(len(hex_hashes))
tree.nodes[4] = [bytes.fromhex(h) for h in hex_hashes]
tree.nodes[3] = merkle_parent_level(tree.nodes[4])
tree.nodes[2] = merkle_parent_level(tree.nodes[3])
tree.nodes[1] = merkle_parent_level(tree.nodes[2])
tree.nodes[0] = merkle_parent_level(tree.nodes[1])

print(tree)
```
結果:
```
*597c4baf.*
6382df3f..., 87cf8fa3...
3ba6c080..., 8e894862..., 7ab01bb6..., 3df760ac...
272945ec..., 9a38d037..., 4a64abd9..., ec7c95e1..., 3b67006c..., 850683df..., d40d268b..., 8636b7a3...
9745f717..., 5573c8ed..., 82a02ecb..., 507ccae5..., a7a4aec2..., bb626766..., ea6d7ac1..., 45774386..., 76880292..., b1ae7f15..., 9b74f89f..., b3a92b5b..., b5c0b915..., c9d52c5c..., c555bc5f..., f9dbfafc...
```
:::
ハッシュリストを指定することで、マークルツリーとその最上位にあるマークルルートを確認することができ、これによってデータの整合性を視認しやすくなります。

:::details マークルパスの探索（偶数・奇数）
偶数
```py: main.py
from merkleblock import MerkleTree
from helper import merkle_parent
hex_hashes = [
    "9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb",
    "5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b",
    "82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05",
    "507ccae5ed9b340363a0e6d765af148be9cb1c8766ccc922f83e4ae681658308",
]
tree = MerkleTree(len(hex_hashes))
tree.nodes[2] = [bytes.fromhex(h) for h in hex_hashes]
while tree.root() is None:
    if tree.is_leaf():
        tree.up()
    else:
        left_hash = tree.get_left_node()
        if left_hash is None:
            tree.left()
        elif tree.right_exists():
            right_hash = tree.get_right_node()
            if right_hash is None:
                tree.right()
            else:
                tree.set_current_node(merkle_parent(left_hash, right_hash))
                tree.up()
        else:
            tree.set_current_node(merkle_parent(left_hash, left_hash))
            tree.up()
print(tree)
```
結果:
```
3ba6c080...
272945ec..., 9a38d037...
9745f717..., 5573c8ed..., 82a02ecb..., 507ccae5...
```

奇数
```py: main.py
from merkleblock import MerkleTree
from helper import merkle_parent
hex_hashes = [
    "9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb",
    "5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b",
    "82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05",
]
tree = MerkleTree(len(hex_hashes))
tree.nodes[2] = [bytes.fromhex(h) for h in hex_hashes]
while tree.root() is None:
    if tree.is_leaf():
        tree.up()
    else:
        left_hash = tree.get_left_node()
        if left_hash is None:
            tree.left()
        elif tree.right_exists():
            right_hash = tree.get_right_node()
            if right_hash is None:
                tree.right()
            else:
                tree.set_current_node(merkle_parent(left_hash, right_hash))
                tree.up()
        else:
            tree.set_current_node(merkle_parent(left_hash, left_hash))
            tree.up()
print(tree)
```
結果:
```
a779fe9f...
272945ec..., bdca1c60...
9745f717..., 5573c8ed..., 82a02ecb...
```
:::
バイナリツリー（二分木）の走査には主に2つのアプローチがあります: 幅優先探索と深さ優先探索。

1. 幅優先探索 (Breadth-First Search, BFS): レベルごとに左から右に進んでいく探索方法です。同じレベルのノードを全て調査してから、次のレベルに進みます。幅優先探索は通常、キューを使用して実装されます。
2. 深さ優先探索 (Depth-First Search, DFS): 各ノードにおける子ハッシュのうち、ある方向（通常左）に進んでから、他の方向に進む探索方法です。深さ優先探索は通常、再帰を使用して実装されます。

マークルツリーの構築においては、通常深さ優先探索が採用されます。これは、リーフに到達すると親ハッシュに戻る必要がなく、またマークルツリーの構造において深さ優先探索が都合が良いからです。特に、リーフが奇数の場合でも、深さ優先探索ではスムーズに処理できます。

実装においては、再帰や明示的なスタックを使用して深さ優先探索を実現することが一般的です。

# マークルルートを検証する
これまでのコードでマークルツリーの構築が確認できました。本項では、これまでの知識を整理し、技術検証を進めます。

:::details マークルルートの導出
```py: verify.py
class Tree:
    def __init__(self, leaves):
        self.leaves = [Node(leaf) for leaf in leaves]
        self.layer  = self.leaves[::]
        self.root   = None
        self.build_tree()
```
入力:
```
block = ["あめ", "あめ", "みかん", "みかん", "みかん", "りんご", "りんご", "どーなっつ", "どーなっつ"]
print(block)

merkletree = Tree(block)

merkle_root = merkletree.root
print(merkle_root)
```
出力:
```
bf03b42cf8e4e8581e08c7595775dc2d9d79c34f9b95885a5d42df100786a218
```
:::
このリストはブロック内のトランザクションに対するマークルパスであり、最終的に求まったマークルルートはウォレットにおいてそのブロック内の全トランザクションの整合性を検証するための重要な情報です。

ウォレットがマークルツリーを用いてマークルルートを計算し、それがブロックヘッダのマークルルートと一致することを確認することで、ブロック内のトランザクションデータが改ざんされていないことを検証できます。これにより、ウォレットは効率的に自身の残高やトランザクションの整合性を確認することができます。

:::details マークルパスの探索
```py: verify.py
def search(self, data):
    target = None
    hash_value = sha256(data.encode()).hexdigest()
    for node in self.leaves:
        if node.hash == hash_value:
            target = node
    return target

def get_pass(self, data):
    target = self.search(data)
    markle_pass = []
    if not(target):
        return
    markle_pass.append(target.hash)
    while target.parent:
        sibling = target.sibling
        markle_pass.append((sibling.hash, sibling.position))
        target = target.parent
    return markle_pass   
```
入力:
```
merkle_pass = merkletree.get_pass("みかん")
print(merkle_pass)
```
出力:
```
['1cbe93b936a4bd4d40c2224462023917cd77c962376feb9128d9e569c9856aaf', ('4261abfc91324dc5319312592125610a16b0b0a996fcdfae1d24766b918afae9', 'right'), ('0a6b2aa3a1fc41a9394c48adbf1cb1c3eb3e3a9db975906fb743308f23e1f15b', 'right'), ('2165f2270bbe8ce292395df14cb4e65e271fe608013c018d6b5864ec9f1865c8', 'left'), ('94f4c2ba6d697dc41109e2194343bf721e2bbc5de48068609578c0f4ba5481ed', 'right')]
```
:::
merkle_pass を求めると"みかん"ノードからマークルツリーのルートまでの経路が示されます。このマークルパスを使うことで、特定のデータがマークルツリー内で正当な位置にあることを検証できます。

0. `1cbe93b936a4bd4d40c2224462023917cd77c962376feb9128d9e569c9856aaf`: "みかん"ノードのハッシュ値
1. `('4261abfc91324dc5319312592125610a16b0b0a996fcdfae1d24766b918afae9', 'right')`: "みかん"ノードの親ノードの右側にある兄弟ノードのハッシュ値とその位置情報。"みかん"ノードとその兄弟ノード（右側のノード）のハッシュを連結して親ノードを作り、そのハッシュがこの要素になります。
2. `('0a6b2aa3a1fc41a9394c48adbf1cb1c3eb3e3a9db975906fb743308f23e1f15b', 'right')`: これは上記のノード（親ノードの右側のノード）の親ノードの右側にある兄弟ノードのハッシュ値とその位置情報です。同様に、ハッシュを連結して親ノードを作り、そのハッシュがこの要素になります。
3. `('2165f2270bbe8ce292395df14cb4e65e271fe608013c018d6b5864ec9f1865c8', 'left')`: 同様の説明で、これは上記ノードの親ノードの左側にある兄弟ノードのハッシュ値とその位置情報です。
4. `('94f4c2ba6d697dc41109e2194343bf721e2bbc5de48068609578c0f4ba5481ed', 'right')`: 同様の説明で、これは上記ノードの親ノードの右側にある兄弟ノードのハッシュ値とその位置情報です。

これらの要素をたどることで、"みかん" ノードからマークルツリーのルートノードまでの経路が示されます。

:::details マークルパスの検証
```py: verify.py
def caluculator(markle_pass):
    value = markle_pass[0]
    for node in markle_pass[1:]:
        sib = node[0]
        position = node[1]
        if position == "right":
            value = sha256(value.encode() + sib.encode()).hexdigest()
        else:
            value = sha256(sib.encode() + value.encode()).hexdigest()
    return value   
```
入力:
```
candidate = caluculator(merkle_pass)
print(candidate)
```
出力:
```
bf03b42cf8e4e8581e08c7595775dc2d9d79c34f9b95885a5d42df100786a218
```
:::
マークルツリーとそのマークルパスを使用してトランザクションの整合性を確認し、最終的にマークルルートが一致することで改ざんが行われていないことを検証できました。
これにより、軽量クライアントやウォレットは全体のブロックチェーンデータを保持せずに、特定のトランザクションの整合性を確認することができるというメリットがあります。


:::details マークルパスの検証
```py: verify.py
def caluculator(markle_pass):
    value = markle_pass[0]
    for node in markle_pass[1:]:
        sib = node[0]
        position = node[1]
        if position == "right":
            value = sha256(value.encode() + sib.encode()).hexdigest()
        else:
            value = sha256(sib.encode() + value.encode()).hexdigest()
    return value   
```
入力:
```
block = ["あめ", "あめ", "みかん", "みかん", "りんご", "りんご", "どーなっつ", "どーなっつ"]
print(block)

merkletree = Tree(block)

merkle_pass = merkletree.get_pass("みかん")
print(merkle_pass)

candidate = caluculator(merkle_pass)
print(candidate)

merkle_root = merkletree.root
print(merkle_root)
```
出力:
```
['あめ', 'あめ', 'みかん', 'みかん', 'りんご', 'りんご', 'どーなっつ', 'どーなっつ']
['1cbe93b936a4bd4d40c2224462023917cd77c962376feb9128d9e569c9856aaf', ('1cbe93b936a4bd4d40c2224462023917cd77c962376feb9128d9e569c9856aaf', 'left'), ('aa280a624da2b88334623a7033f5a6eac1da7012ace335a6366828f59377ed1b', 'left'), ('4d8a97b9474b34c013c92ba1662bb98ae8cab97681ae21b9843db45ee2854730', 'right')]
b8cc63db762632990e49bea7832293ae2d18cff643c78c1e2274c7fcf757354a
b8cc63db762632990e49bea7832293ae2d18cff643c78c1e2274c7fcf757354a
```
:::
最後にブロックの内容を改ざんしてみて、それによってマークルツリーの整合性が損なわれ、マークルルートが異なることを確認してみます。
みかんを1つ食べて要素を1つ少なくします。
マークルツリーからマークルパスを取得し、マークルルートを算出すると、改ざん前のツリーのマークルルートと異なる事が確認できます。

- 改ざん前: `bf03b42cf8e4e8581e08c7595775dc2d9d79c34f9b95885a5d42df100786a218`
- 改ざん後: `b8cc63db762632990e49bea7832293ae2d18cff643c78c1e2274c7fcf757354a`

# 参考
https://github.com/learn-co-students/session7-jsong-programming-blockchain-demo/blob/master/index.ipynb
https://zenn.dev/sakuracase/articles/4f58609f3da6e8
https://alis.to/gaxiiiiiiiiiiii/articles/3dy7vLZn0g89