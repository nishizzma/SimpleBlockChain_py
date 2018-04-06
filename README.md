# SimpleBlockChain V1.1 SimpleBlockChain_net
基于Python的区块链的网络层与数据同步的简单实现

本次系列文章将从实际代码出发，来加深你对区块链网络同步和区块共识的理解。

# 准备工作

由于用到了网络，我们需要借助一个网络框架flask实现网络节点，同时一致性共识的算法建立在挖矿的基础上，我们需要对挖矿的原理有所了解。

##### 1.Flask网络框架

Flask是一个使用 Python 编写的轻量级 Web 应用框架。我们在这里使用flask框架模拟实现区块链网络中的各个节点，以实现节点同步和节点中的最长链选择。

打开Pycharm，新建一个flask项目。我们来初步了解一下，我们将怎么使用它来构造节点。

新建后，默认的flask项目代码是：

from flask import Flask

app = Flask(__name__)

@app.route('/')  #映射根目录
def hello_world():
    return 'Hello World!'  #返回网页信息

if __name__ == '__main__':
    app.run()

我们运行一下项目，将会在consle看到：

* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
在浏览器输入网址：http://127.0.0.1:5000/：


http://127.0.0.1:5000/
我们一看就明白了，flask通过app.route映射网页链接，然后返回网页内容。我们对上述代码稍作修改：

from flask import Flask

app = Flask(__name__)

from flask import Flask

app = Flask(__name__)

@app.route('/')  #映射根目录
def hello_world():
    return 'this is a chainNode!'  #返回网页信息

@app.route('/zhangsan')  
def node_zhangsan():
    return '张三的区块链节点'  

@app.route('/lisi')  
def node_lisi():
    return '李四的区块链节点'  

@app.route('/wangwu')  
def node_wangwu():
    return '王五的区块链节点'  

if __name__ == '__main__':
    app.run()

运行结果依然是：

* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
我们访问网址发现：


节点根目录:http://127.0.0.1:5000/
张三的节点:http://127.0.0.1:5000/zhangsan
王五的节点: http://127.0.0.1:5000/wangwu
这样，我们就通过flask框架构造了网络中不同的网络节点：

http://127.0.0.1:5000/zhangsan    #张三的节点
http://127.0.0.1:5000/lisi        #李四的节点
http://127.0.0.1:5000/wangwu      #王五的节点
...
##### 2.区块链中挖矿概念的简单理解

在区块链中，挖矿其实是一个数学问题。通俗地讲，就是在不断计算一个区块的哈希值，直到计算出的结果符合系统设定的条件，我们就说“成功地挖到一个区块”。

忽略其他因素，我们通过代码简单地模拟一下挖矿的过程：

#通过哈希来挖掘区块
from hashlib import sha256

if __name__ == '__main__':
    x = 11
    y = 0

    #f'{x*y}' 将x*y的结果转化为字符串 encode("utf-8") 默认utf-8，可以不写
    #当x*y的结果的前三位是=都是0的时候找到目标哈希值
    #hexdigest：哈希结果的二进制字符串
    while sha256(f'{x*y}'.encode("utf-8")).hexdigest()[-3:] != "000":
    # while sha256(f'{x*y}'.encode("utf-8")).hexdigest()[-5:] != "01010":
    # while sha256(f'{x*y}'.encode("utf-8")).hexdigest()[:8] != "00001111":
        y += 1  #不符合条件，计算次数统计加1
        print(y)

    print("y = {}".format(y))  #找到目标值时候的计算次数
我们运行程序，看一下结果如何：

sha256(f'{x*y}'.encode("utf-8")).hexdigest()[-3:] == "000"
sha256(f'{x*y}'.encode("utf-8")).hexdigest()[-5:] == "01010"
sha256(f'{x*y}'.encode("utf-8")).hexdigest()[:8] == "00001111"

我们依次增大目标条件的难度值(根据改变匹配的位数)，发现计算的次数越来越大。甚至第三种条件我的Mac本跑了五分钟都没有出结果。。。

```
 #计算了10383次得到目标值
sha256(f'{x*y}'.encode("utf-8")).hexdigest()[-3:] == "000"
 #计算了874020次得到目标值
sha256(f'{x*y}'.encode("utf-8")).hexdigest()[-5:] == "01010"
 #Mac本计算了5分钟，91517163次仍然没有得到目标值！！！这个好难哦...
sha256(f'{x*y}'.encode("utf-8")).hexdigest()[:8] == "00001111"
```

这就是区块链“挖矿”的一个基本原理，挖矿难度值也在于上述代码中的目标条件设定。因此，我们可以通过这个条件设置来更改挖矿难度。

#####3.区块链架构整合与代码完善

前面，为了更直观清楚地理解区块链数据层的结构，我们构造了交易类，交易记录类，区块类，区块链类等来分层分模块详细理解各项功能。

现在，我们将主要的区块链功能整合到一个区块链类里，然后加上必要的工作量证明，区块校验，节点同步，最长链一致性算法等核心代码。并且对之前的Python代码做一个规范化，以后我们谈到的区块链代码框架也一般用今天写的这个。

引入必要模块

import hashlib  #信息安全加密
import json  #网络数据传递格式
import datetime  #时间
from typing import Optional, Dict, Any, List  #必要数据类型
from urllib.parse import urlparse  #url编码解码
from uuid import uuid4  #签名
import requests  #网络请求
from flask import Flask, jsonify, request  #flask网络框架
构造方法

创建一个区块链类ChaorsCoinBlockChain，首先我们需要实现init方法，init方法创造一个区块链时需要有一个创世区块，需要包含一个区块列表和当前交易列表，同时一个区块链必须还有节点信息。

    def __init__(self):
        self.chain = []  #区块列表
        self.current_transactions = []  #交易列表
        self.nodes = set()  #结点
        self.new_block(proof=100, prev_hash=1)  #创建创世区块
新增区块

上篇文章我们已经知道，每个区块包含属性：索引（index），Unix时间戳（timestamp），交易列表（transactions），工作量证明以及前一个区块的Hash值。

block = {
    'index': 1,
    'timestamp': 1506057125.900785,
    'transactions': [
        ...
    ],
    'proof': 324984774000,
    'previous_hash': "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"
}
因此，新增一个区块需要以上各项信息

   # 新增区块
    def new_block(self,
                  proof:int,  #指定工作量类型
                  prev_hash:Optional[str]  #默认是字符串
                  )->Dict[str, Any]:  #指定返回值类型为字典

        block = {
            "index":len(self.chain) + 1,  #新增区块，原有区块索引加1
            "timestamp":datetime.datetime.now(),  #时间戳
            "transactions":self.current_transactions.clear(),  #交易
            "proof":proof,  #工作量证明
            "prev_hash":prev_hash or self.hash(self.chain[-1]) #上个区块的哈希
        }

        self.current_transactions = []  #开辟新的区块，即交易记录加入区块，当前交易需要被清空！！！
        self.chain.append(block)  #将区块追加到区块链表中

        return block
# 新增交易

一个交易信息包括发起者，接受者和交易数额。

交易的数据结构
[
  {
            'sender': "8527147fe1f5426f9dd545de4b27ee00",
            'recipient': "a77f5cdfa2934df3954a5c7c7da5df1f",
            'amount': 5,
   }
   ...
]
    #新增交易
    def new_transaction(self, sender:str, recipient:str, amount:int)->int:
        #生成交易信息
        self.current_transactions.append({
            "sender":sender,  #付款方
            "recipient":recipient,  #收款方
            "amount":amount  #交易数额
        })

        return self.last_block["index"] + 1  #索引标记交易的数量
# 求哈希值

接下来，我们需要定义一个静态方法来求区块的哈希值。

@staticmethod
    def hash(block:Dict[str, Any])->str:  #传入一个字典类型，返回一个字符串
        #对模块进行哈希处理 json.dumps:将区块处理为字符串
        block_str = json.dumps(block, sort_keys=True).encode("utf-8")

        return hashlib.sha256(block_str).hexdigest()  #返回哈希值
# 上个区块

我们还需要一个属性用来访问区块链上一个区块

@property
    def last_block(self)->Dict[str, Any]:

        return self.chain[-1]
# 工作量证明

这里就需要用到，我们上面讲到的挖矿原理(回顾请看上述准备工作2)。由于区块链挖矿需要依赖于上一个区块，所以这里需要传入上个区块作为参数。

    #挖矿依赖于上一个模块
    def proof_of_work(self, last_block)->int:  #挖矿获取工作量证明
        last_proof = last_block["proof"]  #取出算力证明
        last_hash = self.hash(last_block)

        proof = 0  #循环求解符合条件的合法哈希
        #valid_proof:用于验证工作量证明是否符合条件
        while self.valid_proof(last_proof, proof) is False:
            proof += 1

        return proof
# 工作量证明验证

这个方法用于验证矿工的工作量证明是否满足系统条件，挖矿的难度也是在这里设置的。

    @staticmethod
    def valid_proof(last_proof:int, proof:int)->bool:  #验证工作量证明
        guess = f'{last_proof}{proof}'.encode("utf-8")
        guess_hash = hashlib.sha256(guess).hexdigest()

        return guess_hash[:6] == "000000"  #计算难度
        # return guess_hash[-5:] == "24689"  #这里的条件更改可以改变工作量证明计算的难度(即挖矿难度)
# 节点注册

区块链网络中的各个节点需要加入到区块链中，才能完成全网的共识同步。

    def register_node(self, addr:str)->None:  #加入网络中的其他节点，用于更新
        now_url = urlparse(addr)  #url解析
        self.nodes.add(now_url.netloc)  #增加网络节点

# 区块链合法验证

区块链合法性校验需要满足两个条件：1.区块哈希合法性；2.区块工作量证明合法性。二者缺一不可，必须同时满足才能说明该区块合法。而链上所有区块都合法，才能说明区块链合法。

    def valid_chain(self, chain:List[Dict[str, Any]])->bool:  #区块链校验
        last_block = chain[0]  #从第一块开始校验
        current_index = 1

        while current_index < len(chain):  #循环校验
            block = chain[current_index]
            if block["prev_hash"] != self.hash(last_block):  #区块校哈希验
                return False

            if not self.valid_proof(last_block["proof"],
                                    block["proof"]):  #工作量校验
                return False

            last_block = block
            current_index += 1

        return True
# 一致性共识算法

我们知道，有时候由于有不同矿工同时“挖矿”成功，区块链会出现短暂分叉。但是一段时间后，区块链会重新成为一条单独的主链，这个过程依赖于共识算法，而共识又必要依赖于节点同步，这里简单地实现下通过节点同步帮助主链选择最长链的方法。

    def resolve_conflicts(self)->bool:  #冲突，一致性算法的一种
        #取得互联网中最长的链来替换当前的链
        neighbours = self.nodes  #备份节点  eg:127.0.0.1是一个节点，另一个不同的节点192.168.1.
        new_chain = None  #更新后的最长区块链
        max_length = len(self.chain)  #先保存当前节点的长度

        for node in neighbours:  #刷新每个网络节点，获取最长跟新
            response = requests.get(f'http://{node}/chain')  #取得其他节点的区块链
            if response.status_code == 200:  #网络请求成功
                length = response.json()["length"]  #取得邻节点区块链长度
                chain = response.json()["chain"]    #取得邻节点区块链

                #如果找到区块链长度大于当前节点区块链长度并且区块链校验合法，刷新并保存最长区块链
                if length > max_length and self.valid_chain(chain):
                    max_length = length
                    new_chain = chain


        if new_chain:  #判断是否更新成功
            return True
        else:
            return  False
这样，我们就简单搭建了一个简陋但是功能还比较齐全的区块链架构。


# 节点同步

##### 初始化flask框架
前面已经基本了解了flask映射网页的基本使用，这里的主页打印一个信息，用来表示该网络运行正常。
```
app = Flask(__name__)  #初始化flask框架
@app.route("/")
def index_page():
    return "welcome to ChaorsCoin..."
```

##### 查看区块链
新建一个网页用于查看当前节点区块链，构造一个response(其实就是一个字典，因为网络上传输数据的格式是Json)来显示区块链长度和所有区块
```
@app.route("/chain")  #查看所有区块链
def index_chain():
    response = {
        "chain":chaorsCoin.chain,  #区块链
        "length":len(chaorsCoin.chain)  #区块链长度
    }
    return jsonify(response), 200
```

##### 挖矿
新区快的挖掘依赖于上一个合法区块，每当产生一个区块的时候，系统会产生一笔奖励。每一个区块的第一笔交易都是作为系统奖励矿工的一个交易，叫CoinBaseTransaction。
同样，我们将想要显示的block信息封装到一个json字典里，显示在网页。
```
@app.route("/mine")  #挖矿
def index_mine():
    last_block = chaorsCoin.last_block
    proof = chaorsCoin.proof_of_work(last_block)

    #系统奖励比特币
    chaorsCoin.new_transaction(
        sender="0",  #0代表系统奖励,即coinBaseTransaction
        recipient=node_id,
        amount=12.5
    )

    block = chaorsCoin.new_block(proof, chaorsCoin.hash(last_block))  #新增区块

    response = {
        "message":"new block created...",
        "index":block["index"],
        "transactions":block["transactions"],
        "proof":block["proof"],
        "hash":chaorsCoin.hash(block),
        "prev_hash":block["prev_hash"]
    }
    return jsonify(response), 200
```
每调用一次这个页面，就会产生一个新区快加入到当前区块链里。所以要注意，我们这里测试设置的工作量难度不要太大，不然等好久才会生成一个新新区块。
```
   @staticmethod
    def valid_proof(last_proof:int, proof:int)->bool:  #验证工作量证明
        guess = f'{last_proof}{proof}'.encode("utf-8")
        guess_hash = hashlib.sha256(guess).hexdigest()

        return guess_hash[:3] == "000"  #计算难度,这里的计算难度不要设置的太高。不然等好久才会生成一个新区块
        # return guess_hash[-5:] == "24689"
```

##### 创建交易
交易的创建需要传入参数，所以这里使用POST请求。
```
@app.route("/new_transcations", methods=["POST"])  # 创建一个新的交易
def index_new_transcations():
    values = request.get_json()  #抓取网络传输的信息
    required = ["sender", "recipient", "amount"]

    #判断提交的json数据key值是否合法
    if not all(key in values for key in required):
        return "数据不完整或格式错误", 400

    index = chaorsCoin.new_transaction(values["sender"],
                                       values["recipient"],
                                       values["amount"])  #新增交易

    response = {
        "message":f"交易加入到区块{index}"
    }

    return jsonify(response), 200
```

##### 节点新增
在区块链上，每一个用户都是一个独立的节点。当全网数据发生改变，每一个节点都需要去连接周围的节点以同步最新数据。所以，这里需要将周围的节点增加到当前区块链的节点集合中。
```
@app.route("/new_node", methods=["POST"])  #新增节点
def index_new_node():
    values = request.get_json()
    nodes = values.get("nodes")  #获取所有节点

    if nodes is None:
        return "怎么是空节点"

    for node in nodes:
        chaorsCoin.register_node(node)

    response = {
        "message": "网络节点加入到区块",
        "nodes":list(chaorsCoin.nodes)
    }

    return jsonify(response), 200
```
我们这里只是通过flask网络请求，模拟区块链上的节点同步原理。至于真实的节点同步可能是很复杂的，我们只是为了更好地理解其原理。

##### 节点更新
可能全网不同节点的区块链是不同的，张三的区块链Height可能是100，李四的可能是105，那么王五请求全网同步数据时到底该选择谁的区块链呢？我们都知道会选择长度较长且区块链合法的链作为最长链。
```
@app.route("/node_refresh")  #刷新节点
def index_node_refresh():
    replaced = chaorsCoin.resolve_conflicts()  #一致性算法进行最长链选择

    print(replaced)

    if replaced:
        response = {
        "message": "区块链被替换为最长有效链",
        "new chain": chaorsCoin.chain
        }
    else:
        response = {
            "message": "当前区块链为最长无需替换",
            "chain": chaorsCoin.chain
        }
    return jsonify(response), 200
```

这样我们就用flask框架构建了一个具备主要功能的区块链节点，接下来运行并测试一下。
 ```
if __name__ == '__main__':
    app.run("127.0.0.1", port=5005)  #当提示address被占用的时候，更改一下port即可
```
## 单节点网络测试
程序运行结果：
```
当前节点钱包地址: 2a35a1b48e0843269ffc5d7cd6b684e1
 * Running on http://127.0.0.1:5005/ (Press CTRL+C to quit)
```
打开consle输出的网址，试着添加不同的后缀访问不同的功能。





