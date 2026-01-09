# 区块链实习每日总结

## 第一天 (2025/12/27)

主要完成了: geth 的安装与配置，熟悉了以太坊节点的基本操作。能够用 geth 启动起来一个节点.

笔者用的是 NixOS, 编辑 Home Manager 的 Nix 配置文件, 在 `home.packages` 中添加 `geth` 包,
然后运行 `nh home switch path:.` 来安装 geth.

整个过程非常顺利, `geth` 很容易就安装好了.

## 第二天 (2025/12/28)

由于实际需求, 需要把 `geth` 降级, 降成能够运行 London 时代支持 PoW 的 Ethereum 区块链节点的那个版本.
多亏 Flake 的灵活特性, 我直接到 nixpkgs 版本查询网站, 我要用的版本是 1.10.1, 所以打开
[这个网站](https://lazamar.co.uk/nix-versions/), 在上面搜索 `go-ethereum`, 找到 `1.10.1` 版本对应的
rev commit 为 `0ffaecb6f04404db2c739beb167a5942993cfd87`. 于是到 `flake.nix` 文件中去添加 input:

```nix
        go-ethereum-legacy-nixpkgs.url = "github:NixOS/nixpkgs/0ffaecb6f04404db2c739beb167a5942993cfd87";
```

随后回到 Home Manager 的配置, 在 `home.packages` 中添加:

```nix
        inputs.go-ethereum-legacy-nixpkgs.legacyPackages.${system}.go-ethereum
```

然后 rebuild. 此时运行 `geth version` 能看到版本已经变成了 `1.10.1`.

```
Geth
Version: 1.10.1-stable
Architecture: amd64
Go Version: go1.16.2
Operating System: linux
GOPATH=
GOROOT=go
```

## 第三天 (2025/12/29)

尝试配置不同 geth 之间组网. 发现不同的 geth 之间即使网络连接畅通, 但 geth 却未能主动发起
peer 对等连接.

尝试用 Wireshark 抓包, 甚至对 geth 应用了 strace 看 geth 到底试图建立了网络连接没, 结论是
没有. 最后的解决要点在于删除了不同 geth 数据文件夹下的 nodekey 文件. 由于直接复制节点的
data 文件夹, 导致 nodekey 文件也连同一起被复制了. 但 nodekey 相同会导致 geth 认为节点间是
"相同节点", 从而不去主动发起连接, 即使手动建立 static 节点关系. 删除节点数据文件夹下的
`geth/nodekey` 文件, 可让 `geth` 重新建立 nodekey, 这样不同的 geth 节点才可认为是不同的,
geth 才会去建立 P2P 连接.

## 第四天 (2025/12/30)

尝试向链上部署简单的智能合约. 我选择 Foundry 为框架, 随手写了个最简单的:

```sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract Counter {
    uint256 public number;

    function helloWorld() public pure returns (string memory) {
        return "Hello, World!";
    }

    function setNumber(uint256 newNumber) public {
        number = newNumber;
    }

    function getNumber() public view returns (uint256) {
        return number;
    }

    function increment() public {
        number = number + 1;
    }
}
```

附加写一个脚本来部署:

```sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script} from "forge-std/Script.sol";
import {Counter} from "../src/Counter.sol";

contract CounterScript is Script {
    Counter public counter;

    function setUp() public {}

    function run() public {
        vm.startBroadcast();

        counter = new Counter();

        vm.stopBroadcast();
    }
}
```

`forge build` 后 `forge script` 来部署. 此处需要用到链上的私钥来填写 `--private-key` 参数.
这个得自己写个 JS 脚本去获取:

```
const fs = require('fs');
const keythereum = require('keythereum');

const keystoreDir = ''; # TODO: keystore 文件夹路径
const keystoreFile = ''; # TODO: 具体的 keystore 文件名
const password = '123456';
const fullPath = `${keystoreDir}/${keystoreFile}`;

try {
  // 方式 1: 直接从文件加载（推荐，手动指定文件）
  const keyObject = JSON.parse(fs.readFileSync(fullPath, 'utf8'));

  // 方式 2: 如果知道地址，可用 importFromFile（会自动查找匹配地址的文件）
  // const address = "0xYourEthereumAddressWithout0xPrefix".toLowerCase();
  // const keyObject = keythereum.importFromFile(address, keystoreDir);

  const privateKeyBuffer = keythereum.recover(password, keyObject);
  const privateKeyHex = privateKeyBuffer.toString('hex');

  console.log('地址:', keyObject.address);
  console.log('明文私钥 (0x 前缀): 0x' + privateKeyHex);

  // 注意：不要保存到文件！直接复制后立即删除脚本输出
} catch (err) {
  console.error('错误:', err.message);
  if (err.message.includes('MAC')) console.error('可能是密码错误');
}
```

获取完成, 拷贝输出的私钥, 填入 `forge script` 的参数中. 还要注意我们部署的目标区块链是 legacy 的,
所以还得添加 `--legacy --evm-version london` 参数.

完事了, 用 `cast call` 和 `cast send` 来测试.

## 第五天 (2025/12/31)

尝试在 Firefox 中安装 MetaMask (小狐狸钱包) 并添加自己的私有链节点. 到 Firefox 扩展设置页面安装 MetaMask,
此处不过多赘述.

设置好自己的钱包后, 为 MetaMask 添加自己的私有链网络.

点击 Networks -> Add a custom network, 填写自己的 HTTP RPC URL 等信息, 成功添加后, 向该地址转点钱.
等过一段时间, 应该能够看到自己的钱包余额增加了.

接下来, 尝试为私有链部署 dApp. 我选择了这款[宠物商店dApp](https://github.com/OffchainLabs/demo-dapp-pet-shop).
但是这个项目原本用的是 truffle 框架, 不是很符合 NixOS 的风格, 很烦人, 我把它迁移到 Foundry 框架上了.

部署成功后, 用 MetaMask 与之交互. 打开网页界面, 可以看到各种各样的 pets. 选择一个自己钟意的, 点击 Adopt,
在 MetaMask 中确认交易. 等交易确认后, 能够看到 Adopt 变成 Success 了, 代表领养成功.

