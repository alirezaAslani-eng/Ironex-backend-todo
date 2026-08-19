# مشکلات Backend و جزئیات تست


## ۱۰. نیاز به API برای دریافت قوانین KYC

نیاز است Backend یک API مشابه API مربوط به `Loyalty Rules` در اختیار Frontend قرار دهد تا لیست قوانین و مقررات مربوط به KYC را دریافت کنیم.

### نیازمندی

* ایجاد یک API برای دریافت لیست قوانین KYC
* ساختار Response مشابه `Loyalty Rules` باشد.
* هر قانون باید حداقل شامل متن قانون و اطلاعات لازم برای شناسایی آن باشد.
* API باید امکان دریافت قوانین فعال و قابل نمایش به کاربر را فراهم کند.





## ۹. مشکل اشتباه بودن `currentLevel` در وضعیت KYC

پاسخ فعلی Backend برای وضعیت KYC به شکل زیر است:

```json
{
  "json": {
    "currentLevel": "Level2_Advanced",
    "level1": {
      "identity": {
        "stepKey": "Level1_Identity",
        "status": "Approved",
        "comment": null
      }
    },
    "level2": {
      "address": {
        "stepKey": "Level2_Address",
        "status": "Approved",
        "comment": null
      },
      "document": {
        "stepKey": "Level2_Document",
        "status": "NotStarted",
        "comment": null
      }
    },
    "level3": {
      "liveness": {
        "stepKey": "Level3_Liveness",
        "status": "NotStarted",
        "comment": null
      }
    }
  }
}
```

### نتیجه تست

در پاسخ، مقدار `currentLevel` برابر با `Level2_Advanced` است، در حالی که مرحله `Level2_Document` هنوز وضعیت `NotStarted` دارد.

بنابراین کاربر هنوز مرحله Document مربوط به سطح دوم را تکمیل و تأیید نکرده است، اما Backend مقدار `currentLevel` را `Level2_Advanced` برمی‌گرداند.

نیاز است منطق تعیین `currentLevel` بررسی شود تا این مقدار بر اساس وضعیت واقعی مراحل KYC تعیین شود و زمانی که یک Step ضروری هنوز تکمیل یا تأیید نشده است، سطح کاربر به‌صورت اشتباه `Level2_Advanced` اعلام نشود.

## ۱. مشکل آپدیت نشدن کیف پول

هنگام انجام معامله، کیف پول کاربر از طریق SignalR به‌صورت Real-time آپدیت نمی‌شود.

همچنین هنگام خرید یک نماد، خود نماد وارد کیف پول نمی‌شود و حتی بعد از Refresh نیز همچنان در کیف پول نمایش داده نمی‌شود.

نکته مهم این است که موجودی ریالی کیف پول بعد از Refresh آپدیت می‌شود، اما دارایی خریداری‌شده به کیف پول اضافه نمی‌شود.

### درخواست خرید

```json
{
  "ProductCode": "REBAR",
  "OrderSide": 0,
  "OrderType": 1,
  "Weight": 1,
  "Price": 750000,
  "IsDemo": false,
  "settlementMode": 0
}
```

### پاسخ کیف پول قبل از خرید

```json
{
  "isSuccess": true,
  "data": {
    "totalPortfolioValueIrt": 1513000000,
    "totalProfitLoss24hIrt": 0,
    "totalProfitLoss24hPercentage": 0,
    "availableCash": 1513000000,
    "marginCredit": 6000000,
    "buyingPower": 1519000000,
    "assets": [
      {
        "assetSymbol": "IRT",
        "availableBalance": 1513000000.000,
        "lockedBalance": 0.000,
        "livePrice": 1,
        "change24h": 0,
        "totalValueInIrt": 1513000000.000,
        "profitLoss24hInIrt": 0,
        "lockedDetails": {
          "totalLocked": 0.000,
          "lockedInOrders": 0.000,
          "lockedForDebt": 0.000,
          "lockedByAdmin": 0.000
        }
      }
    ]
  }
}
```

### پاسخ کیف پول بعد از خرید

```json
{
  "isSuccess": true,
  "data": {
    "totalPortfolioValueIrt": 1513000000,
    "totalProfitLoss24hIrt": 0,
    "totalProfitLoss24hPercentage": 0,
    "availableCash": 1512259260,
    "marginCredit": 6000000,
    "buyingPower": 1518259260,
    "assets": [
      {
        "assetSymbol": "IRT",
        "availableBalance": 1512259260.000,
        "lockedBalance": 740740.000,
        "livePrice": 1,
        "change24h": 0,
        "totalValueInIrt": 1513000000.000,
        "lockedDetails": {
          "totalLocked": 740740.000,
          "lockedInOrders": 740740.000,
          "lockedForDebt": 0.000,
          "lockedByAdmin": 0.000
        }
      }
    ]
  }
}
```

### وضعیت SignalR

لیسنرهای زیر در Frontend فعال هستند:

```ts
walletHub.start(con);

con.on(onPortfolioUpdate, updatePortfolio);
con.on(onPriceUpdate, updatePrice);
con.on(onRefreshWallet, refreshPortfolio);
```

اما هیچ‌کدام از Eventهای زیر اجرا نمی‌شوند (**فقط برای لغو سفارش ها و واریز به کیف پول اپدیت انجام میشه**):

* `onPortfolioUpdate`
* `onPriceUpdate`
* `onRefreshWallet`

بنابراین آپدیت Real-time کیف پول از طریق SignalR انجام نمی‌شود.

## ۲. مشکل آپدیت نشدن کیف پول بعد از Fill شدن سفارش فروش

هنگام فروش یک دارایی و Fill شدن سفارش، موجودی کیف پول کاربر باید افزایش پیدا کند؛ اما بعد از Fill شدن سفارش فروش، کیف پول آپدیت نمی‌شود.

### وضعیت تست

این مورد فعلاً قابل تست نیست.

دلیل این است که دارایی خریداری‌شده در کیف پول اضافه نمی‌شود و در نتیجه امکان فروش آن دارایی و بررسی رفتار کیف پول بعد از Fill شدن سفارش فروش وجود ندارد.

## ۳. مشکل معامله فوری در حالت Demo

هنگام انجام معامله در حالت Demo، با وجود اینکه موجودی کافی وجود دارد، معامله فوری انجام نمی‌شود و با خطای `Order.HoldFailed` مواجه می‌شود.

### درخواست ثبت سفارش

```json
{
  "ProductCode": "REBAR",
  "OrderSide": 0,
  "OrderType": 1,
  "Weight": 1,
  "Price": 750000,
  "IsDemo": true,
  "settlementMode": 0
}
```

### پاسخ سرور

```json
{
  "isSuccess": false,
  "message": "ثبت سفارش یا مسدودسازی دارایی با خطا مواجه شد. موجودی خود را بررسی کنید.",
  "errorCode": "Order.HoldFailed"
}
```

### موجودی کیف پول Demo

```json
{
  "isSuccess": true,
  "data": {
    "totalPortfolioValueIrt": 1000000000,
    "totalProfitLoss24hIrt": 0,
    "totalProfitLoss24hPercentage": 0,
    "availableCash": 1000000000,
    "marginCredit": 6000000,
    "buyingPower": 1006000000,
    "assets": [
      {
        "assetSymbol": "IRT",
        "availableBalance": 1000000000.00,
        "lockedBalance": 0.00,
        "livePrice": 1,
        "change24h": 0,
        "totalValueInIrt": 1000000000.00,
        "profitLoss24hInIrt": 0,
        "lockedDetails": {
          "totalLocked": 0.00,
          "lockedInOrders": 0.00,
          "lockedForDebt": 0.00,
          "lockedByAdmin": 0.00
        }
      }
    ]
  }
}
```

## ۴. مشکل آپدیت نشدن موجودی Trading Credit / Margin

اعتبار معاملاتی در محاسبات معامله در نظر گرفته می‌شود و مقدار `marginCredit` در کیف پول وجود دارد.

اما بعد از انجام معامله، موجودی `Margin / Trading Credit` آپدیت نمی‌شود، حتی بعد از Refresh.

همچنین این مقدار باید از طریق SignalR به‌صورت Real-time نیز آپدیت شود.

### وضعیت فعلی

مقدار `marginCredit` قبل و بعد از معامله همچنان `6,000,000` باقی می‌ماند.

نیاز است بعد از انجام معامله، مقدار اعتبار معاملاتی و Buying Power متناسب با معامله به‌درستی به‌روزرسانی شود.

## ۵. مشکل بررسی Locked Details برای دارایی‌های خریداری‌شده

به دلیل اینکه بعد از خرید، نماد وارد کیف پول نمی‌شود، امکان بررسی `lockedDetails` برای دارایی خریداری‌شده وجود ندارد.

این اطلاعات برای مشخص کردن دارایی‌هایی که نیاز به پرداخت بدهی دارند استفاده می‌شود.

### وضعیت تست

این مورد فعلاً قابل تست نیست.

ابتدا باید مشکل اضافه نشدن دارایی خریداری‌شده به کیف پول برطرف شود تا بتوان `lockedDetails` مربوط به آن دارایی را بررسی کرد.

## ۶. تفکیک نشدن Order Book در معامله ۱۰ درصد

سرویس Order Book با وجود دریافت Query مربوط به `settlementMode`، بین Order Book حالت ۱۰ درصد و حالت عادی تفاوتی ایجاد نمی‌کند. حتا در اپدیت سیگنال ار هم تفاوتی اعمال نمیکنه من چک کردم که حتما  unsub و sub  به حالت جدید اتفاق بیوفته و درست عمل میکنه ولی اپدیت ها اشتباه ارسال میشن.

### درخواست حالت ۱۰ درصد

```text
/api/v1/market/depth/REBAR?isDemo=false&setllementMode=1
```

### پاسخ

```json
{
  "isSuccess": true,
  "data": {
    "bids": [
      [
        100000,
        10
      ]
    ],
    "asks": [
      [
        740000,
        18586.8
      ]
    ]
  },
  "message": "عملیات با موفقیت انجام شد.",
  "errorCode": null
}
```

### درخواست حالت عادی

```text
/api/v1/market/depth/REBAR?isDemo=false&setllementMode=0
```

### نتیجه تست

پاسخ هر دو درخواست دقیقاً یکسان است.

یعنی مقدار `setllementMode=1` هیچ تفاوتی در Order Book ایجاد نمی‌کند و سفارش‌های حالت ۱۰ درصد و حالت معمولی از یکدیگر تفکیک نمی‌شوند.

## ۷. کسر شدن ۱۰۰ درصد در معامله ۱۰ درصد

هنگام انجام معامله با `settlementMode: 1`، به جای کسر ۱۰ درصد مبلغ معامله، ۱۰۰ درصد مبلغ از کیف پول کاربر کسر می‌شود.

### درخواست ثبت سفارش ۱۰ درصد

```json
{
  "ProductCode": "REBAR",
  "OrderSide": 0,
  "OrderType": 1,
  "Weight": 1,
  "Price": 750000,
  "IsDemo": true,
  "settlementMode": 1
}
```

### درخواست بررسی کیف پول

```text
/api/v1/wallet/portfolio?isdemo=false
```

### نتیجه تست

با وجود اینکه سفارش با `settlementMode: 1` ثبت می‌شود، در زمان انجام معامله **۱۰۰ درصد مبلغ معامله از کیف پول کسر می‌شود** و نه ۱۰ درصد.

بنابراین منطق کسر موجودی برای معاملات ۱۰ درصد همچنان صحیح نیست.

## ۸. ارسال نشدن Eventهای مربوط به معامله

برای دریافت آپدیت‌های Market، ابتدا Subscribe به Market انجام می‌شود.

### اینوکر Market

```ts
con.invoke(
  "SubscribeToMarket",
  symbol,
  isdemo,
  settlementMode,
)
```

در این درخواست مقادیر زیر به Server ارسال می‌شوند:

* `symbol`
* `isdemo`
* `settlementMode`

### Eventهای مورد انتظار

```ts
const onTradeExecuted = "OnTradeExecuted";
const orderBookUpdated = "OrderBookUpdated";
const OnMarketPriceChanged = "OnMarketPriceChanged";
```

Eventهایی که Frontend برای دریافت تغییرات Market به آن‌ها گوش می‌دهد:

* `OnTradeExecuted`
* `OrderBookUpdated`
* `OnMarketPriceChanged`

### نتیجه تست

Eventهای مربوط به معامله و تغییرات Market از سمت Server به‌درستی دریافت نمی‌شوند و در نتیجه آپدیت‌های Real-time مربوط به معامله، Order Book و قیمت در Frontend انجام نمی‌شود.
