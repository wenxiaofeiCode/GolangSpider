# 通过 Golang 买卖 Bitcoin #

## 方案一: 通过ExinCore API进行币币交易 ##

[Exincore]( https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fexinone%2Fexincore ) 提供了基于Mixin Network的币币交易API.

你可以支付USDT给ExinCore, ExinCore会以最低的价格，最优惠的交易费将你购买的比特币转给你, 每一币交易都是匿名的，并且可以在区块链上进行验证，交易的细节只有你与ExinCore知道！

ExinCore 也不知道你是谁，它只知道你的UUID.

### 预备知识: ###

你先需要创建一个机器人, 方法在 [教程一]( https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fwenewzhang%2Fmixin_labs-php-bot%2Fblob%2Fmaster%2FREADME-zhchs.md ).

#### 安装依赖包 ####

正如教程一里我们介绍过的， 我们需要依赖 **mixin-sdk-go** , 你应该先安装过它了， 这儿我们再安装 **uuid, msgpack** 两个软件包.

` go get -u github.com/vmihailenco/msgpack go get -u github.com/satori/go.uuid 复制代码`

#### 充币到 Mixin Network, 并读出它的余额. ####

ExinCore可以进行BTC, USDT, EOS, ETH 等等交易， 这儿演示如果用 USDT购买BTC 或者 用BTC购买USDT, 交易前，先检查一下钱包地址！ 完整的步骤如下:

* 检查比特币或USDT的余额，钱包地址。并记下钱包地址。
* 从第三方交易所或者你的冷钱包中，将币充到上述钱包地址。
* 再检查一下币的余额，看到帐与否。(比特币的到帐时间是5个区块的高度，约100分钟)。

请注意，比特币与USDT的地址是一样的。

` if cmd == "2" { userInfo, userID := ReadAssetInfo( "BTC" ) fmt.Println( "User ID " ,userID, "'s BTC Address is: " , userInfo[ "data" ].( map [ string ] interface {})[ "public_key" ]) fmt.Println( "Balance is: " , userInfo[ "data" ].( map [ string ] interface {})[ "balance" ]) } if cmd == "3" { userInfo, userID := ReadAssetInfo( "USDT" ) fmt.Println( "User ID " ,userID, "'s USDT Address is: " , userInfo[ "data" ].( map [ string ] interface {})[ "public_key" ]) fmt.Println( "Balance is: " , userInfo[ "data" ].( map [ string ] interface {})[ "balance" ]) } 复制代码`

#### 查询ExinCore市场的价格信息 ####

如果来查询ExinCore市场的价格信息呢？你要先了解你交易的基础币是什么，如果你想买比特币，卖出USDT,那么基础货币就是USDT;如果你想买USDT,卖出比特币，那么基础货币就是比特币.

` if cmd == "6" { priceInfo, err := GetMarketPrice(mixin.GetAssetId( "USDT" )) if err != nil { log.Fatal(err) } var marketInfo map [ string ] interface {} err = json.Unmarshal([] byte (priceInfo), &marketInfo) fmt.Println( "Asset | price | min | max | exchanges" ) for _, v := range (marketInfo[ "data" ].( map [ string ] interface {})) { fmt.Println(v.( map [ string ] interface {})[ "exchange_asset_symbol" ], "/" , v.( map [ string ] interface {})[ "base_asset_symbol" ], v.( map [ string ] interface {})[ "price" ], v.( map [ string ] interface {})[ "minimum_amount" ], v.( map [ string ] interface {})[ "maximum_amount" ], v.( map [ string ] interface {})[ "exchanges" ], ) } } func GetMarketPrice (asset_id string ) ([] byte , error) { var body [] byte req, err := http.NewRequest( "GET" , "https://exinone.com/exincore/markets?base_asset=" +asset_id, bytes.NewReader(body)) if err != nil { return nil , err } req.Header.Set( "Content-Type" , "application/json" ) resp, err := httpClient.Do(req) if err != nil { return nil , err } defer resp.Body.Close() // fmt.Println(resp.Body) bt, err := ioutil.ReadAll(resp.Body) if resp.StatusCode != http.StatusOK { var resp struct { Error Error `json:"error"` } err = json.Unmarshal(bt, &resp) if err == nil { err = resp.Error } } return bt, err } 复制代码`

#### 交易前，创建一个Memo! ####

在第二章里, [基于Mixin Network的PHP比特币开发教程: 机器人接受比特币并立即退还用户]( https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fwenewzhang%2Fmixin_labs-php-bot%2Fblob%2Fmaster%2FREADME2-zhchs.md ) , 我们学习过退还用户比特币，在这里，我们除了给ExinCore支付币外，还要告诉他我们想购买的币是什么，即将想购买的币存到memo里。

` packUuid, _ := uuid.FromString(mixin.GetAssetId( "BTC" )) pack, _ := msgpack.Marshal(OrderAction{A: packUuid}) memo := base64.StdEncoding.EncodeToString(pack) // fmt.Println(memo) 复制代码`

#### 币币交易的完整流程 ####

转币给ExinCore时，将memo写入你希望购买的币，否则，ExinCore会直接退币给你！ 如果你想卖出比特币买入USDT,调用方式如下：

` EXIN_BOT = "61103d28-3ac2-44a2-ae34-bd956070dab1" ; packUuid, _ := uuid.FromString(mixin.GetAssetId( "USDT" )) pack, _ := msgpack.Marshal(OrderAction{A: packUuid}) memo := base64.StdEncoding.EncodeToString(pack) // fmt.Println(memo) priKey, pToken, sID, userID, uPIN := GetWalletInfo() bt, err := mixin.Transfer(EXIN_BOT, "0.0001" ,mixin.GetAssetId( "BTC" ),memo, messenger.UuidNewV4().String(), uPIN,pToken,userID,sID,priKey) if err != nil { log.Fatal(err) } fmt.Println( string (bt)) 复制代码`

如果你想卖出USDT买入比特币,调用方式如下：

` packUuid, _ := uuid.FromString(mixin.GetAssetId( "BTC" )) pack, _ := msgpack.Marshal(OrderAction{A: packUuid}) memo := base64.StdEncoding.EncodeToString(pack) // fmt.Println(memo) priKey, pToken, sID, userID, uPIN := GetWalletInfo() bt, err := mixin.Transfer(EXIN_BOT, "0.0001" ,mixin.GetAssetId( "USDT" ),memo, messenger.UuidNewV4().String(), uPIN,pToken,userID,sID,priKey) if err != nil { log.Fatal(err) } fmt.Println( string (bt)) 复制代码`

交易完成后，Exincore会将你需要的币转到你的帐上，同样，会在memo里，记录成交价格，交易费用等信息！你只需要按下面的方式解开即可！

* **readUserSnapshots** 读取钱包的交易记录。

` 复制代码`

一次成功的交易如下：

` ------------MEMO:-coin--exchange-------------- memo: hqFDzQPooVCnMzg3Mi45N6FGqTAuMDAwNzc0NqJGQcQQgVsLGidkNzaPqkLWlPpiCqFUoUahT8QQIbfeL6p5RVOcEP0mLb+t+g== You Get Coins: 815b0b1a-2764-3736-8faa-42d694fa620a 0.3857508 Successful Exchange: Fee asset ID: 815b0b1a-2764-3736-8faa-42d694fa620a fee is :0.0007746 Order ID: 21b7de2f-aa79-4553-9c10-fd262dbfadfa Price is :3872.97 --------------memo-record end--------------- 复制代码`

#### 读取币的余额 ####

通过读取币的余额，来确认交易情况！

` if cmd == "2" { userInfo, userID := ReadAssetInfo( "BTC" ) fmt.Println( "User ID " ,userID, "'s BTC Address is: " , userInfo[ "data" ].( map [ string ] interface {})[ "public_key" ]) fmt.Println( "Balance is: " , userInfo[ "data" ].( map [ string ] interface {})[ "balance" ]) } if cmd == "3" { userInfo, userID := ReadAssetInfo( "USDT" ) fmt.Println( "User ID " ,userID, "'s USDT Address is: " , userInfo[ "data" ].( map [ string ] interface {})[ "public_key" ]) fmt.Println( "Balance is: " , userInfo[ "data" ].( map [ string ] interface {})[ "balance" ]) } 复制代码`

## 源代码执行 ##

执行 **go run coin_exchange.go** 即可开始交易了.

* 1: Create Wallet
* 2: Read Bitcoin balance & Address
* 3: Read USDT balance & Address
* 4: Read EOS balance & address
* 5: pay 0.0001 BTC buy USDT
* 6: Read ExinCore Price(USDT)
* 7: Read ExinCore Price(BTC)
* 8: pay 1 USDT buy BTC
* q: Exit Make your choose:

[完整代码]( https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fwenewzhang%2Fmixin_labs-go-bot%2Fblob%2Fmaster%2Fcoin_exchange%2Fcoin_exchange.go )