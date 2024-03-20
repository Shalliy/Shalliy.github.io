---
title: iOS内购（IAP）那些事
date: 2018-07-31 15:38:11
tags: iOS
---

最近有个项目客户总是反应掉单，于是乎就看了看内购相关的东西，发现坑还真是不少，这里做个总结。

>IAP，即in-App Purchase，是一种智能移动终端应用程序付费的模式，在苹果（Apple）iOS、谷歌安卓（Google Android）、微软WindowsPhone等智能移动终端操作系统中都有相应的实现。  
> -- 百度百科

<!--More-->

### 内购流程

想知道坑在哪里，首先应该了解流程，那么我们首先通过内购的流程，一步步地说坑到底在哪里。
苹果内购的主要流程如下：

```flow
获取商品信息 => 创建交易 => 把交易添加到队列 => 交易成功获取凭证 
=> 拿着凭证做二次验证 => 交易成功
```

1. 通过产品ID获取商品信息（SKProduct）。

    ```objc
    #import <StoreKit/StoreKit.h>
    
    //把商品ID信息放入一个集合中
    NSSet *sets = [NSSet setWithObjects:@"phonicsphase1", nil];
    //请求内购商品信息，只返回你请求的产品（主要用于验证商品的有效性）
    SKProductsRequest *productrequest = [[SKProductsRequest alloc] initWithProductIdentifiers:sets];
    productrequest.delegate = self;
    [productrequest start];
    
    #pragma mark - 获取商品ID成功的代理方法
    - (void)productsRequest:(SKProductsRequest *)request didReceiveResponse:(SKProductsResponse *)response {
        //返回的是SKProduct对象数组
        //如果你上面请求的是多个，那么这里返回的也是多个
        SKProduct *product = [response.products firstObject];
        //查询成功，开始支付
        [self startPaymentWithProduct:product];
    }
    ```

2. 拿到商品信息，创建支付对象`SKMutablePayment`。

    ```objc
    - (void)startPaymentWithProduct:(SKProduct *)product {
        SKMutablePayment *payment = [SKMutablePayment paymentWithProduct:product];
        payment.applicationUsername = @"myOrderID";
        [[SKPaymentQueue defaultQueue] addPayment:payment];
    }
    ```

    这里的`applicationUsername`是一个透传字段，我们在这里传什么参数，在支付成功的时候Apple会原封不动的返回给我们。最常用的就是拿它传递我们自己的订单号，以便跟我们知道是哪个订单支付成功了。
    **这里有个坑：**
    据说我们传递过去的`applicationUsername`有可能返回的时候变成空的，这点我没有遇到，有遇到的可以说一下。
3. 把当前交易添加到交易队列中去（上面的addPayment:方法）。
4. 监听支付结果（paymentQueue:updatedTransactions:），如果支付成功，Apple会把支付成功的凭证（recipt）存到沙盒中。

    ```objc
    //首先我们要在 viewDidLoad 方法中添加监听对象 
    [[SKPaymentQueue defaultQueue] addTransactionObserver:self];
    
    #pragma mark -  监听的代理方法
    - (void)paymentQueue:(SKPaymentQueue *)queue updatedTransactions:(NSArray<SKPaymentTransaction *> *)transactions {
            [transactions enumerateObjectsUsingBlock:^(SKPaymentTransaction * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            SKPaymentTransaction *transation = obj;
            switch (transation.transactionState) {
                case SKPaymentTransactionStatePurchasing:
                    {
                        NSLog(@"购买中");
                    }
                    break;
                    
                case SKPaymentTransactionStatePurchased:
                {
                    NSLog(@"交易完成");
                    //获取透传字段
                    NSString *orderNo = transation.payment.applicationUsername;
                    //transactionIdentifier：相当于Apple的订单号
                    NSString *transationId = transation.transactionIdentifier;
                    NSLog(@"orderNo = %@, 交易ID = %@", orderNo, transationId);
                    //从沙盒中获取交易凭证
                    NSData *reciptData = [NSData dataWithContentsOfURL:[[NSBundle mainBundle] appStoreReceiptURL]];
                    //转化成Base64字符串（用于校验）
                    NSString *reciptString = [reciptData base64EncodedStringWithOptions:NSDataBase64Encoding64CharacterLineLength];
                    //传给后台做二次验证
                    [self checkReceipt:reciptString];                
                }
                    break;
                    
                case SKPaymentTransactionStateFailed:
                {
                    //localizedDescription可以作为提示信息（交易失败无法连接到 iTunes Store）
                    NSLog(@"交易失败%@", transation.error.localizedDescription);
                    [[SKPaymentQueue defaultQueue] finishTransaction:transation];
                }
                    break;
                
                case SKPaymentTransactionStateRestored:
                {
                    NSLog(@"恢复购买完成");
                    //恢复完成（对应restoreCompletedTransactions）方法
                }
                    break;
                
                case SKPaymentTransactionStateDeferred:
                {
                    NSLog(@"交易推迟, 等待外部操作");
                    //交易推迟
                    //官方解释是：交易已经加入队列，但是需要等待外部操作
                    //主要用于儿童模式，需要询问家长同意。这种情况下不能关闭订单（完成交易），否则这类充值将无法处理。                
                    }
                    break;
                    
                default:
                    break;
            }
        }];
    }
    ```

    **注意：**
    1. 透传字段`applicationUsername`可能返回的是`nil`，这也是丢单的原因之一，有些人说遇到过，有的则说没有遇到过。这个尚不清楚。
    2. `[[SKPaymentQueue defaultQueue] finishTransaction:transation]`如果不调用这个方法，那么`transation`就永远不会结束。也就是说每次进来都会重新调用`updatedTransactions`这个方法，**就算你做过二次校验都没用**。
    3. 二次校验只会校验你的凭证是不是有效，不会关心你这个凭证是不是校验过了，所以说，**一个凭证可以被校验多次（这是刷单方法之一）**
    4. 上面说过，如果不`finishTransaction`，每次进入这个界面都会调用监听方法，但是注意了**每次返回的 reciptData都是不一样，即使是同一个订单。**也就是说，同一个订单也有可能有多个`reciptData`，所以它并不能用来确定是哪个订单。
    5. 我们最好是在App启动的时候就去设置监听。把内购封装成一个工具类，这样每次启动App就会调用苹果的补单流程。

5. 我们从沙盒中取到凭证（recipt），发送给我们自己后台进行二次验证，验证成功表示支付成功。

    ```objc
    - (void)checkReceipt:(NSString *)receipt {
        [AntManager postWithPath:CheckReceipt_URL params:params success:^(id response) {
            if ([response[@"status"] integerValue] == 0) {
                //如果支付成功，我们就结束交易
                [[SKPaymentQueue defaultQueue] finishTransaction:transation]
            } else {
                
            }
        } failure:^(id error) {
        
        }];
    }
    ```

    **注意：**
    前面说过，Apple在最初设计IAP的时候就没有想过后台的参与，但是，为了安全起见，我们要把二次验证放到后台做，至于为什么，我们后边会讲到。但是不管怎样，我们至少要了解一下怎么做的二次校验。
    * 我们把recipt经过Base64编码之后，传给Apple的验证服务器进行验证。
    格式如下:
    `{"receipt-data": 你编码过的recipt}`
    * Apple的验证服务器地址有两个  
    `https://sandbox.itunes.apple.com/verifyReceipt` 是沙盒环境的验证地址。
    `https://buy.itunes.apple.com/verifyReceipt` 是正式环境的验证地址
    * 如果你用的是测试账号（就是在iTunes Connect里面设置的，具体请[看这里](https://help.apple.com/app-store-connect/#/dev8b997bee1)）支付的，那么你就需要发送到沙盒环境的验证地址，正式环境应该切换到正式环境的验证地址。
Apple返回的完整数据如下：

    ```json
      {
      "status": 0,
      "environment": "Sandbox"
      "receipt": {
         "receipt_type": "ProductionSandbox",
         "adam_id": 0,
         "app_item_id": 0,
         "bundle_id": "com.BlueMobi.Phonics",
         "application_version": "1.5.0",
         "download_id": 0,
         "version_external_identifier": 0,
         "receipt_creation_date": "2018-06-28 14:08:26 Etc/GMT",
         "receipt_creation_date_ms": "1530194906000",
         "receipt_creation_date_pst": "2018-06-28 07:08:26 America/Los_Angeles",
         "request_date": "2018-08-05 04:50:58 Etc/GMT",
         "request_date_ms": "1533444658147",
         "request_date_pst": "2018-08-04 21:50:58 America/Los_Angeles",
         "original_purchase_date": "2013-08-01 07:00:00 Etc/GMT",
         "original_purchase_date_ms": "1375340400000",
         "original_purchase_date_pst": "2013-08-01 00:00:00 America/Los_Angeles",
         "original_application_version": "1.0",
         "in_app": [
             {
                 "quantity": "1",
                 "product_id": "*******",
                 "transaction_id": "1000000404314890", //这个苹果的交易唯一标识符
                 "original_transaction_id": "1000000404314890",
                 "purchase_date": "2018-06-04 09:58:41 Etc/GMT",
                 "purchase_date_ms": "1528106321000",
                 "purchase_date_pst": "2018-06-04 02:58:41 America/Los_Angeles",
                 "original_purchase_date": "2018-06-04 09:58:41 Etc/GMT",
                 "original_purchase_date_ms": "1528106321000",
                 "original_purchase_date_pst": "2018-06-04 02:58:41 America/Los_Angeles",
                 "is_trial_period": "false"
             },
             {
                 "quantity": "1",
                 "product_id": "*******",
                 "transaction_id": "1000000404523773",
                 "original_transaction_id": "1000000404523773",
                 "purchase_date": "2018-06-05 02:21:26 Etc/GMT",
                 "purchase_date_ms": "1528165286000",
                 "purchase_date_pst": "2018-06-04 19:21:26 America/Los_Angeles",
                 "original_purchase_date": "2018-06-05 02:21:26 Etc/GMT",
                 "original_purchase_date_ms": "1528165286000",
                 "original_purchase_date_pst": "2018-06-04 19:21:26 America/Los_Angeles",
                 "is_trial_period": "false"
             }
           ]
         }
      }
      ```
    * Apple返回的数据也是Json格式的，里面有个字段`status`，当`status == 0`的时候，表示校验成功。但是，我们不能以这个`status`为标准，我们还要判断我们的订单是不是在校验信息里面。
    * 当`status`不为0的时候，是没有其余的Json数据的。

        ```json
        21000    App Store 不能读取你提供的JSON对象
        21002    receipt-data 域的数据有问题
        21003    receipt 无法通过验证
        21004    提供的 shared secret 不匹配你账号中的 shared secret
        21005    receipt 服务器当前不可用
        21006    receipt 合法, 但是订阅已过期. 服务器接收到这个状态码时, receipt 数据仍然会解码并一起发送
        21007    receipt 是 Sandbox receipt, 但却发送至生产系统的验证服务
        21008    receipt 是生产 receipt, 但却发送至 Sandbox 环境的验证服务
        ```

    * `in_app`是一个数组，一条数据对应一条交易。后台需要遍历数组，跟我们传递给后台的`transaction_id`做对比，看看是否存在啊，如果存在就说明本条交易校验成功了。
    * 有人说`in_app`里面的数据是按照时间先后来排序的，只取第一条就可以了，但是经过我的测试发现并不是这样的，所以最好还是逐条对比。

### 掉单问题的探讨

苹果的IAP缺陷还是很明显的，但是没办法，谁让苹果一家独大呢，我们也只能吐吐槽，想尽一切办法防止掉单的发生。
掉单的原因是有多方面的，我们一个个来说。

1. 首先苹果是有补单措施的，如果不finish交易，每次都会请求`updatedTransactions:`方法的。但是前面说了，`applicationUsername`的缺失会导致掉单。`applicationUsername`的丢失会让我们丢失订单号，但是实际的支付仍然是成功的。
    1. 可以传递给后台`未知订单`，但客户投诉的时候，我们从未知订单里面找到符合条件的给恢复订单信息（这样做显然效率很低）
    2. **常用的做法是：**
        * 把每个订单的`recipt`、`orderNumber`、`userId`、`transactionIdentifier`等你需要的信息保存为一个plist文件，每个plist文件对应一个订单。等我们后台返回支付成功的时候就删除相应的plist。
        * 每次启动App的时候去沙盒查找plist文件，如果有，就读取plist文件去请求服务器做二次验证，直到服务器返回成功为止。
        * 但是这样做，就需要在`SKPaymentTransactionStatePurchased`的时候`finishTransaction:`结束交易，因为每次返回的`reciptData`是不同的。除非我们每次都去更新它。并且这样做的话，用户卸载App的时候，保存的订单信息全部都没有了。
    3. 用户支付过后，没有等到验证，就更换Apple账号登录。这种情况我没有试过会不会调用`updatedTransactions:`方法
    4. 其实这个问题始终没有一个很好的解决方案，只能靠苹果的优化了。

2. 由于苹果的服务器在美帝，所以延迟的情况还是很严重的，这时候如果是请求超时，我们服务器又没有收到回调，那么就会产生掉单。比较坑的是，`updatedTransactions:`**在App整个生命周期只会走一次。**其实这种情况只要重启App就会重新走苹果的补单流程了。大佬们如果觉得用户体验不好，可以把信息存本地，隔一段时间请求一次服务器。
其实现在掉单基本就是上面说的那个原因了，只要把它解决了，掉单基本上也就没有了。

**注意：**

1. 掉单问题都是出现在支付完成之后，没有通过二次校验的时候。因为一旦我们得到了`recipt`，就说明苹果已经扣款成功了。但是为什么我们不能以这个作为支付成功的标准呢，请看下面的刷单系列。
2. 针对上面第二种`recipt`存本地的方法，其实没有必要，苹果自己的补单措施基本上是够用了。
3. 如果是你们的App中，用户可以一次性购买多个产品，那么更推荐把`recipt`存本地。因为如果在有多个成功交易未 finish 掉的情况下把应用关闭后再次打开的时候，有些交易的回调会被漏掉（苹果真坑啊！！！）。当然两者结合也是一个很好的办法。

### 刷单的问题

刷单的转自[这篇博客](https://blog.csdn.net/autfish/article/details/52682778)
我在这里简述一下，做个小结。

刷单主要有以下几种方式：

   1. 破解IAP
   2. 重复使用`recipt-data`
   3. 信用卡黑卡
   4. 外币差价
   5. 以及苹果对小额消费不做验证规则的“36技术”

我们这里只讨论 1 和 2 ，因为下面三种跟我们开发者没有关系，想要详细了解的可以去开头的博客上了解一下。

#### 1. 破解IAP

   这个主要是我们不做二次验证引发的漏洞，非法用户可以利用插件模拟扣款成功，有可能是手动调用`updatedTransactions:`（我猜的😂）。就算是我们在App端做了二次验证，结果也有可能是被篡改过的。所以我们需要在自己的服务端进行二次验证。 **任何在App端的数据都是不安全的！！！**

#### 2. 重复使用`recipt-data`

   我们直到，`recipt-data`是支付成功返回给我们的校验凭证，前面说过，苹果校验的时候只负责校验recipt的有效性和真假，并不负责这个凭证是否被用过。如果我们在后台只判断`status == 0`，而不去做其他判断的话，非法用户会保存`recipt-data`，然后多次使用，因为每次都是校验通过的。

   我们也不能判断`recipt-data`是否被使用，因为上面已经说过，同一个订单也会有多个`recipt-data`返回的。

   防范的方法是在确定status值为0后，进一步解析出数据中的transaction_id并存入数据库。每次发货前先检查数据库中是否已经有本次的transaction_id存在，如果已存在则拒绝发货。

还有一种情况需要注意，有些App购买前先有一步创建订单的行为，在服务器端记录购买的商品、时间等，且发货时是按照订单记录中的商品，那么需要比较苹果返回信息中的product_id与订单表中的记录值是否一致。

到了这里IAP基本上已经结束了，小弟只是把IAP的流程以及需要注意的点列出来，以上的方法只是作为总结，并不适用于所有状况，我们在开发中还要具体问题具体分析。希望大家开发中都不会掉单😄。