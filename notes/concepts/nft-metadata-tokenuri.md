# NFT, Metadata, tokenURI

## 一句话理解

NFT 本身不是图片。

NFT 合约核心记录的是：哪个地址拥有哪个 `tokenId`。

图片、名字、描述、属性这些展示信息，通常通过 `tokenURI(tokenId)` 指向 metadata。

## NFT 合约核心存什么

ERC721 最核心可以理解成：

```solidity
mapping(uint256 tokenId => address owner) owners;
```

也就是：

```text
tokenId 1 belongs to Alice
tokenId 2 belongs to Bob
```

所以 NFT 的链上核心是：

```text
contract address + tokenId + owner
```

## tokenURI 和 metadata

`tokenURI(tokenId)` 返回这个 NFT 的 metadata 位置，或者直接返回 metadata 本身。

metadata 通常是 JSON：

```json
{
  "name": "My NFT #1",
  "description": "A simple NFT",
  "image": "ipfs://...",
  "attributes": []
}
```

钱包和 NFT 市场显示图片的大致流程：

```text
ownerOf(tokenId)
tokenURI(tokenId)
read metadata JSON
read image field
display image
```

## URI 和 URL

URI 是更大的概念，表示“资源标识符”。

URL 是 URI 的一种，表示“资源在哪里以及怎么访问”。

例子：

- `https://example.com/1.json`: URL，也是 URI
- `ipfs://Qm.../1.json`: URI，不是传统 HTTP URL
- `data:application/json;base64,...`: URI，内容直接在字符串里

简单记：

```text
URL tells where to fetch.
URI identifies or contains the resource.
```

## NFT 图片可以放哪里

### Centralized server

```text
https://example.com/metadata/1.json
```

简单便宜，但服务器可能挂，项目方也可能换内容。

### IPFS

```text
ipfs://Qm...
```

IPFS 是内容寻址存储。它不是按服务器地址找文件，而是按内容 hash 找文件。

优点是内容更难被偷偷篡改；缺点是需要有人 pin 文件，否则也可能访问不到。

### Arweave

偏长期/永久存储，经常用于 NFT metadata 或图片。

### Fully on-chain

metadata 和 image 都由合约直接生成。

常见返回：

```text
data:application/json;base64,<Base64(JSON)>
```

JSON 里的 image 可能是：

```text
data:image/svg+xml;base64,<Base64(SVG)>
```

## SVG 和 Base64

SVG 是文本形式的图片，适合链上生成。

Base64 不是加密，只是编码。它常用来把 JSON 或 SVG 变成 URI 安全的字符串。

on-chain SVG NFT 的结构通常是：

```text
tokenURI
  -> data:application/json;base64,<JSON>
      -> image: data:image/svg+xml;base64,<SVG>
```

## baseURI

`baseURI` 通常是 `tokenURI` 的前缀。

普通 NFT 可能是：

```text
baseURI = ipfs://QmMetadata/
tokenURI(1) = ipfs://QmMetadata/1
```

链上 SVG NFT 里也可能是：

```solidity
return "data:application/json;base64,";
```

这表示后面接的是 base64 编码后的 JSON metadata。

## metadata 能不能改

取决于合约设计。

- `https://` URL: 项目方可能改服务器内容
- 固定 IPFS CID: 内容更固定
- owner 可以 `setBaseURI`: 项目方可以换 metadata
- fully on-chain: 更固定，但更贵

Reveal 项目通常会先用统一 metadata，之后 owner 改 `baseURI` 显示真正图片。

## ERC721 vs ERC1155

- ERC721: 一个 `tokenId` 只有一个，适合头像、艺术品、唯一资产
- ERC1155: 一个 `tokenId` 可以有多个数量，适合游戏道具、门票、半同质化资产

## 审计时注意

- `tokenURI` 返回的 metadata 是否可信、是否可变
- `baseURI` 是否可以被 owner 修改
- 是否有事件记录 metadata/baseURI 修改
- metadata reveal 是否存在项目方过度控制
- 链上 SVG/Base64 拼接是否可能出错
- NFT ownership 和 metadata 展示不要混淆

## 我的理解

NFT 的所有权在链上，展示内容靠 metadata。

`tokenURI` 是连接链上 token 和链下/链上 metadata 的入口。

metadata 里的 `image` 才是图片位置。SVG 只是图片格式之一，适合完全链上 NFT。
