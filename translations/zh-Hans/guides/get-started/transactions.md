---
title: 付款和收款
---

现在你拥有了一个账户，你可以通过 Stellar 网络发送和接收资金。如果你还没有创建一个账户的话，[请阅读入门指南的第二步](./create-account.md)。

大多数时候，你会把资产发送给有另外一个拥有账户的人。然而在本教程中，您应当再创建一个账户，以此来模拟付款操作。

## 付款

在 Stellar 中，付款、改变你的账户或交易各种货币等会改变总账的行为被称为**操作**。[^1] 为了实际执行一项操作，你需要创建一个事务，它只是一组附带一些额外信息的操作，比如这个事务是哪个账户发起的，以及一个用于验证事务真实性的加密签名。[^2]

如果事务中的任何操作失败，则所有操作都会失败。例如，假设你有 100 Lumens，你做了两次支付操作，每次支付 60 Lumens。如果您进行两个事务(每个事务有一个操作) ，第一个事务将成功，第二个事务将失败，因为您没有足够的流明。你会剩下 40 Lumens。然而，如果你将这两笔支付放在一个事务中，它们都会失败，并且你的账户中还会剩下 100 Lumens。

最后，每笔交易都要支付一小笔费用。 就像账户的最低余额一样，这项费用可以让系统避免因大量的恶意交易而超负荷运转。这笔费用被称为**基本费用**，数额很小——每次操作的费用为 100 stroops (即 0.00001 XLM，当表示极小数额的时候 stroops 比 lumen 更方便)。一笔包含了两次操作的事务要花费 200 stroops。[^3]

### 构建事务

Stellar 以 XDR 编码的二进制格式存储和传输事务数据。[^4] Stellar SDK 提供了一些工具来简化这些工作。下面是将 10 Lumens 发送到另一个账户的示例：

<code-example name="提交事务">

```js
var StellarSdk = require('stellar-sdk');
StellarSdk.Network.useTestNetwork();
var server = new StellarSdk.Server('https://horizon-testnet.stellar.org');
var sourceKeys = StellarSdk.Keypair
  .fromSecret('SCZANGBA5YHTNYVVV4C3U252E2B6P6F5T3U6MM63WBSBZATAQI3EBTQ4');
var destinationId = 'GA2C5RFPE6GCKMY3US5PAB6UZLKIGSPIUKSLRB6Q723BM2OARMDUYEJ5';
// Transaction will hold a built transaction we can resubmit if the result is unknown.
var transaction;

// First, check to make sure that the destination account exists.
// You could skip this, but if the account does not exist, you will be charged
// the transaction fee when the transaction fails.
server.loadAccount(destinationId)
  // If the account is not found, surface a nicer error message for logging.
  .catch(StellarSdk.NotFoundError, function (error) {
    throw new Error('The destination account does not exist!');
  })
  // If there was no error, load up-to-date information on your account.
  .then(function() {
    return server.loadAccount(sourceKeys.publicKey());
  })
  .then(function(sourceAccount) {
    // Start building the transaction.
    transaction = new StellarSdk.TransactionBuilder(sourceAccount)
      .addOperation(StellarSdk.Operation.payment({
        destination: destinationId,
        // Because Stellar allows transaction in many currencies, you must
        // specify the asset type. The special "native" asset represents Lumens.
        asset: StellarSdk.Asset.native(),
        amount: "10"
      }))
      // A memo allows you to add your own metadata to a transaction. It's
      // optional and does not affect how Stellar treats the transaction.
      .addMemo(StellarSdk.Memo.text('Test Transaction'))
      .build();
    // Sign the transaction to prove you are actually the person sending it.
    transaction.sign(sourceKeys);
    // And finally, send it off to Stellar!
    return server.submitTransaction(transaction);
  })
  .then(function(result) {
    console.log('Success! Results:', result);
  })
  .catch(function(error) {
    console.error('Something went wrong!', error);
    // If the result is unknown (no response body, timeout etc.) we simply resubmit
    // already built transaction:
    // server.submitTransaction(transaction);
  });
```

```java
Network.useTestNetwork();
Server server = new Server("https://horizon-testnet.stellar.org");

KeyPair source = KeyPair.fromSecretSeed("SCZANGBA5YHTNYVVV4C3U252E2B6P6F5T3U6MM63WBSBZATAQI3EBTQ4");
KeyPair destination = KeyPair.fromAccountId("GA2C5RFPE6GCKMY3US5PAB6UZLKIGSPIUKSLRB6Q723BM2OARMDUYEJ5");

// First, check to make sure that the destination account exists.
// You could skip this, but if the account does not exist, you will be charged
// the transaction fee when the transaction fails.
// It will throw HttpResponseException if account does not exist or there was another error.
server.accounts().account(destination);

// If there was no error, load up-to-date information on your account.
AccountResponse sourceAccount = server.accounts().account(source);

// Start building the transaction.
Transaction transaction = new Transaction.Builder(sourceAccount)
        .addOperation(new PaymentOperation.Builder(destination, new AssetTypeNative(), "10").build())
        // A memo allows you to add your own metadata to a transaction. It's
        // optional and does not affect how Stellar treats the transaction.
        .addMemo(Memo.text("Test Transaction"))
        .build();
// Sign the transaction to prove you are actually the person sending it.
transaction.sign(source);

// And finally, send it off to Stellar!
try {
  SubmitTransactionResponse response = server.submitTransaction(transaction);
  System.out.println("Success!");
  System.out.println(response);
} catch (Exception e) {
  System.out.println("Something went wrong!");
  System.out.println(e.getMessage());
  // If the result is unknown (no response body, timeout etc.) we simply resubmit
  // already built transaction:
  // SubmitTransactionResponse response = server.submitTransaction(transaction);
}
```

```go
package main

import (
	"github.com/stellar/go/build"
    "github.com/stellar/go/clients/horizon"
    "fmt"
)

func main () {
	source := "SCZANGBA5YHTNYVVV4C3U252E2B6P6F5T3U6MM63WBSBZATAQI3EBTQ4"
	destination := "GA2C5RFPE6GCKMY3US5PAB6UZLKIGSPIUKSLRB6Q723BM2OARMDUYEJ5"

	// Make sure destination account exists
	if _, err := horizon.DefaultTestNetClient.LoadAccount(destination); err != nil {
		panic(err)
	}

	passphrase := network.TestNetworkPassphrase

	tx, err := build.Transaction(
		build.TestNetwork,
		build.SourceAccount{source},
		build.AutoSequence{horizon.DefaultTestNetClient},
		build.Payment(
			build.Destination{destination},
			build.NativeAmount{"10"},
		),
	)

	if err != nil {
		panic(err)
	}

	// Sign the transaction to prove you are actually the person sending it.
	txe, err := tx.Sign(source)
	if err != nil {
		panic(err)
	}

	txeB64, err := txe.Base64()
	if err != nil {
		panic(err)
	}

	// And finally, send it off to Stellar!
	resp, err := horizon.DefaultTestNetClient.SubmitTransaction(txeB64)
	if err != nil {
		panic(err)
	}

	fmt.Println("Successful Transaction:")
	fmt.Println("Ledger:", resp.Ledger)
	fmt.Println("Hash:", resp.Hash)
}
```

</code-example>

到底发生了什么？ 让我们来逐步的分析一下。

1. 通过从 Stellar 网络加载收款帐户 ID 的数据，确认该账户确实存在于 Stellar 网络中。如果你跳过这个步骤，代码仍然可以正常运行，但是载入账户数据可以让你避免提交一个失败的事务。您还可以在此这里检查您可能要在目标帐户上执行的任何其他验证。 例如，如果您正在编写银行软件，您可以在这里插入监管合规性检查和 KYC 验证。

    <code-example name="载入收款账户">

    ```js
    server.loadAccount(destinationId)
      .then(function(account) { /* validate the account */ })
    ```

    ```java
    server.accounts().account(destination);
    ```

    ```go
    if _, err := horizon.DefaultTestNetClient.LoadAccount(destination); err != nil {
    	panic(err)
    }
    ```

    </code-example>

2. 加载要付款帐户的数据。 一个帐户一次只能执行一个事务[^5]，每个账户都有一个[**序列号**](../concepts/accounts.md#sequence-number)，它这个序号可以帮助 Stellar 验证事务的顺序。事务的序列号需要与帐户的序列号相匹配，因此您需要从网络中获取帐户的当前序列号。

    <code-example name="载入付款账户">

    ```js
    .then(function() {
    return server.loadAccount(sourceKeys.publicKey());
    })
    ```

    ```java
    AccountResponse sourceAccount = server.accounts().account(source);
    ```

    </code-example>

    当您构建一个事务时，SDK 会自动增加帐户的序列号，因此如果您想执行第二个事务，无需再次查询序列号。

3. 开始建立一个交易。这需要一个帐户对象，而不仅仅是一个帐户 ID，因为这个操作将使帐户的序列号增加。

    <code-example name="构建一个事务">

    ```js
    var transaction = new StellarSdk.TransactionBuilder(sourceAccount)
    ```

    ```java
    Transaction transaction = new Transaction.Builder(sourceAccount)
    ```

    ```go
    tx, err := build.Transaction(
    // ...
    )
    ```

    </code-example>

4. 添加一个支付操作。值得注意的是，你需要指定你发送的资产的类型——Stellar 的"原生"资产是 Lumen，但你可以发送任何类型的资产或货币，可以是美元或比特币，也可以是你信任的发行人发行的任何类型的资产[（这里介绍了更多的细节）](#transacting-in-other-currencies)。 但是在这里我们还是使用 Lumen，它在 SDK 中被称为 "原生 (native)" 资产：

    <code-example name="添加操作">

    ```js
    .addOperation(StellarSdk.Operation.payment({
      destination: destinationId,
      asset: StellarSdk.Asset.native(),
      amount: "10"
    }))
    ```

    ```java
    .addOperation(new PaymentOperation.Builder(destination, new AssetTypeNative(), "10").build())
    ```

    ```go
    tx, err := build.Transaction(
    	build.Network{passphrase},
    	build.SourceAccount{from},
    	build.AutoSequence{horizon.DefaultTestNetClient},
    	build.MemoText{"Test Transaction"},
    	build.Payment(
    		build.Destination{to},
    		build.NativeAmount{"10"},
    	),
    )
    ```

    </code-example>

    您应该注意到金额是一个字符串而不是一个数字。当处理极小或极大的值时，[浮点数可能会引入误差](https://en.wikipedia.org/wiki/Floating_point#Accuracy_problems)。 由于并非所有的系统都有一个原生的方法来精确地表示极小或极大的小数，Stellar 使用字符串作为一种可靠的方法来表示精确的值。

5. 您还可以将元数据(在 Stellar 中被称为 "[**memo**](../concepts/transactions.md#memo)")添加到事务中。Stellar 不会对这些数据做任何处理，但您可以将其用于任何您想要的目的。例如，如果您是一家代其他人收款或付款的银行，您可能会在这里填写收付款的实际对象。

    <code-example name="添加 Memo">

    ```js
    .addMemo(StellarSdk.Memo.text('Test Transaction'))
    ```

    ```java
    .addMemo(Memo.text("Test Transaction"));
    ```

    ```go
    build.MemoText{"Test Transaction"},
    ```

    </code-example>

6. 现在事务已经拥有了它所需要的所有数据，随后您必须使用您的私密种子对它进行加密签名。 签名可以证明数据是由您提供的，而不是有人冒充您。

    <code-example name="签署事务">

    ```js
    transaction.sign(sourceKeys);
    ```

    ```java
    transaction.sign(source);
    ```

    ```go
    txe, err := tx.Sign(from)
    ```

    </code-example>

7. 最后让我们把它提交到 Stellar 网络上。

    <code-example name="提交事务">

    ```js
    server.submitTransaction(transaction);
    ```

    ```java
    server.submitTransaction(transaction);
    ```

    ```go
    resp, err := horizon.DefaultTestNetClient.SubmitTransaction(txeB64)
    ```

    </code-example>

**重要** 由于 bug、网络状况等原因，您可能不会收到 Horizon 服务器的响应。 在这种情况下，你将无法确定事务的状态。 因此我们应该始终将构建的事务(或以 XDR 格式编码的事务)保存在变量或数据库中，并在不知道其状态时重新提交它。 如果交易已经成功地并入到总账中，Horizon 将直接返回事务的结果，并且不会再次尝试提交该事务。 只有在交易状态未知（并且有机会被并入总账）的情况下，才会重新提交到网络。

## 收款

你无需做任何事情就可以接收 Stellar 账户的付款——如果付款人成功地提交了将资产发送到你的账户的事务，那么这些资产将自动添加你的账户。

然而，你会想知道某人是否真的付了钱给你。如果您是一家代表他人接受付款的银行，您需要查明付款人给你支付了多少钱，以便您可以向指定的收件人支付资金。 如果你经营的是零售业务，你需要在把商品交给他们之前确认他们是否已经付过钱了。 如果你是一辆拥有 Stellar 账户的自动出租汽车，你可能会想要在前排座位上的顾客启动引擎之前确认他是否已经付了款。

下面是一个简单的程序，通过它你可以看到支付信息：

<code-example name="Receive Payments">

```js
var StellarSdk = require('stellar-sdk');

var server = new StellarSdk.Server('https://horizon-testnet.stellar.org');
var accountId = 'GC2BKLYOOYPDEFJKLKY6FNNRQMGFLVHJKQRGNSSRRGSMPGF32LHCQVGF';

// Create an API call to query payments involving the account.
var payments = server.payments().forAccount(accountId);

// If some payments have already been handled, start the results from the
// last seen payment. (See below in `handlePayment` where it gets saved.)
var lastToken = loadLastPagingToken();
if (lastToken) {
  payments.cursor(lastToken);
}

// `stream` will send each recorded payment, one by one, then keep the
// connection open and continue to send you new payments as they occur.
payments.stream({
  onmessage: function(payment) {
    // Record the paging token so we can start from here next time.
    savePagingToken(payment.paging_token);

    // The payments stream includes both sent and received payments. We only
    // want to process received payments here.
    if (payment.to !== accountId) {
      return;
    }

    // In Stellar’s API, Lumens are referred to as the “native” type. Other
    // asset types have more detailed information.
    var asset;
    if (payment.asset_type === 'native') {
      asset = 'lumens';
    }
    else {
      asset = payment.asset_code + ':' + payment.asset_issuer;
    }

    console.log(payment.amount + ' ' + asset + ' from ' + payment.from);
  },

  onerror: function(error) {
    console.error('Error in payment stream');
  }
});

function savePagingToken(token) {
  // In most cases, you should save this to a local database or file so that
  // you can load it next time you stream new payments.
}

function loadLastPagingToken() {
  // Get the last paging token from a local database or file
}
```

```java
Server server = new Server("https://horizon-testnet.stellar.org");
KeyPair account = KeyPair.fromAccountId("GC2BKLYOOYPDEFJKLKY6FNNRQMGFLVHJKQRGNSSRRGSMPGF32LHCQVGF");

// Create an API call to query payments involving the account.
PaymentsRequestBuilder paymentsRequest = server.payments().forAccount(account);

// If some payments have already been handled, start the results from the
// last seen payment. (See below in `handlePayment` where it gets saved.)
String lastToken = loadLastPagingToken();
if (lastToken != null) {
  paymentsRequest.cursor(lastToken);
}

// `stream` will send each recorded payment, one by one, then keep the
// connection open and continue to send you new payments as they occur.
paymentsRequest.stream(new EventListener<OperationResponse>() {
  @Override
  public void onEvent(OperationResponse payment) {
    // Record the paging token so we can start from here next time.
    savePagingToken(payment.getPagingToken());

    // The payments stream includes both sent and received payments. We only
    // want to process received payments here.
    if (payment instanceof PaymentOperationResponse) {
      if (((PaymentOperationResponse) payment).getTo().equals(account)) {
        return;
      }

      String amount = ((PaymentOperationResponse) payment).getAmount();

      Asset asset = ((PaymentOperationResponse) payment).getAsset();
      String assetName;
      if (asset.equals(new AssetTypeNative())) {
        assetName = "lumens";
      } else {
        StringBuilder assetNameBuilder = new StringBuilder();
        assetNameBuilder.append(((AssetTypeCreditAlphaNum) asset).getCode());
        assetNameBuilder.append(":");
        assetNameBuilder.append(((AssetTypeCreditAlphaNum) asset).getIssuer().getAccountId());
        assetName = assetNameBuilder.toString();
      }

      StringBuilder output = new StringBuilder();
      output.append(amount);
      output.append(" ");
      output.append(assetName);
      output.append(" from ");
      output.append(((PaymentOperationResponse) payment).getFrom().getAccountId());
      System.out.println(output.toString());
    }

  }
});
​````

​```go
package main

import (
	"context"
	"fmt"
	"github.com/stellar/go/clients/horizon"
)

func main() {
	const address = "GC2BKLYOOYPDEFJKLKY6FNNRQMGFLVHJKQRGNSSRRGSMPGF32LHCQVGF"
	ctx := context.Background()

	cursor := horizon.Cursor("now")

	fmt.Println("Waiting for a payment...")

	err := horizon.DefaultTestNetClient.StreamPayments(ctx, address, &cursor, func(payment horizon.Payment) {
		fmt.Println("Payment type", payment.Type)
		fmt.Println("Payment Paging Token", payment.PagingToken)
		fmt.Println("Payment From", payment.From)
		fmt.Println("Payment To", payment.To)
		fmt.Println("Payment Asset Type", payment.AssetType)
		fmt.Println("Payment Asset Code", payment.AssetCode)
		fmt.Println("Payment Asset Issuer", payment.AssetIssuer)
		fmt.Println("Payment Amount", payment.Amount)
		fmt.Println("Payment Memo Type", payment.Memo.Type)
		fmt.Println("Payment Memo", payment.Memo.Value)
	})

	if err != nil {
		panic(err)
	}

}
```

</code-example>

这个程序有两个主要部分。首先，为涉及给定账户的支付创建一个查询。 与 Stellar 中的大多数查询一样，这可能返回大量的条目，因此 API 返回分页信息，您可以稍后使用这些信息从之前中断的地方开始查询。 在上面的例子中，保存和加载分页信息的函数是空白的，但是在一个真正的应用程序中，你需要将分页信息保存到一个文件或者数据库中，这样你就可以在程序崩溃或者用户关闭它的情况下使用分页信息从中断的地方继续查询。

<code-example name="构建一个支付信息查询">

```js
var payments = server.payments().forAccount(accountId);
var lastToken = loadLastPagingToken();
if (lastToken) {
  payments.cursor(lastToken);
}
```

```java
PaymentsRequestBuilder paymentsRequest = server.payments().forAccount(account)
String lastToken = loadLastPagingToken();
if (lastToken != null) {
  paymentsRequest.cursor(lastToken);
}
```

</code-example>

其次，查询的结果是以 Stream 流提供的，这是监听支付信息最简单的方式。Stream 会推送所有已存在的支付信息，当所有信息推送完成后，Stream 会保持开启的状态，一旦有新的支付行为，Stream 便会推送该支付信息。

尝试一下: 运行这个程序，然后在另一个窗口中创建并提交付款。你应该看到这个程序打印出支付记录。

<code-example name="使用 Stream 查询支付信息">

```js
payments.stream({
  onmessage: function(payment) {
    // handle a payment
  }
});
```

```java
paymentsRequest.stream(new EventListener<OperationResponse>() {
  @Override
  public void onEvent(OperationResponse payment) {
    // Handle a payment
  }
});
```

</code-example>

你也可以分组或分页查询支付信息。当你处理完一页付款信息后，你可以获取并处理下一页。

<code-example name="通过翻页信息查询支付信息">

```js
payments.call().then(function handlePage(paymentsPage) {
  paymentsPage.records.forEach(function(payment) {
    // handle a payment
  });
  return paymentsPage.next().then(handlePage);
});
```

```java
Page<OperationResponse> page = payments.execute();

for (OperationResponse operation : page.getRecords()) {
	// handle a payment
}

page = page.getNextPage();
```

</code-example>


## 以其他货币进行交易

Stellar 网络最让人惊叹的一点便是你可以收发多种类型的资产，比如美元、尼日利亚奈拉、比特币等数字资产，甚至是你自己发行的新资产。

Stellar 的原生资产是 Lumens，所有其他资产可以被认为是某个特定账户发行的信贷。 事实上，当你在 Stellar 网络上交易美元时，你并不是在交易真正的美元，而是在交易某个特定账户发行的美元信贷。这就是为什么上面例子中的资产既有 `code` 又有 `issuer`。`issuer` 是创建资产的帐户的 ID。在您信任某个资产前请先确保您对这个账户有所了解，确保它有能够将您在 Stellar 网络中的资产兑换为真实世界的资产。正因为如此，你通常只会信任由大型金融机构发行的国家货币。

Stellar 还支持让发送者发送一种货币，而接收者收到另外一种货币。你可以发送奈拉（尼日利亚的货币单位）给在德国的朋友，而他们却能够收到欧元。这种货币兑换是通过一种内置的交易市场机制实现的，在这种机制下，人们可以购买和出售不同类型的资产。Stellar 会自动找到最佳的兑换模式，以便将您的奈拉兑换成欧元。这个系统被称为[分布式交易所](../concepts/exchange.md)

您可以在[资产概述](../concepts/assets.md)中阅读有关资产详细信息的更多内容。

## 接下来做什么？

现在你已经能够使用 Stellar 的 API 进行付款和收款了，你可以开始编写各种令人惊叹的财务软件了。你也可以开始尝试 Stellar 的其它 API，然后阅读一些更加深入的主题：

- [Become an anchor](../anchor/)
- [Security](../security.md)
- [Federation](../concepts/federation.md)
- [Compliance](../compliance-protocol.md)

<div class="sequence-navigation">
  <a class="button button--previous" href="create-account.html">上一页：创建账户</a>
</div>



[^1]: 所有可用操作的列表可以在[操作页面](../concepts/operations.md)上找到。

[^2]: 有关事务的详细信息可以在[事务页面](../concepts/transactions.md)上找到。

[^3]: 这 100 stroops 被称为恒星的**基本费用**。 基本费用是可以调整的，但是它可能多年才会调整一次。你可以通过[检查最新的总账信息](https://www.stellar.org/developers/horizon/reference/endpoints/ledgers-single.html)查询当前的费用。

[^4]: 尽管 Horizon REST API 的大部分响应都使用 JSON，但 Stellar 中的大部分数据实际上是以一种称为 XDR(External Data Representation) 的格式存储的。使用 XDR 编码的数据比用 JSON 编码的数据更小，并且以更规范的方式存储数据，这使得签名和验证 XDR 编码的消息更加容易。您可以在此[获取更多有关 XDR 的信息](https://www.stellar.org/developers/horizon/reference/xdr.html)。

[^5]: 在需要在短时间内执行大量事务的情况下(例如，一家银行可能使用一个 Stellar 账户代表多个客户执行事务) ，您可以创建多个 Stellar 账户同时工作。 在 [channels 指南](../channels.md)中了解更多信息。