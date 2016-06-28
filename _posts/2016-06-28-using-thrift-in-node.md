---
layout: post
category: Programming
published: true
tags: [JavaScript, Node.js]
title: 使用Apache Thrift：以Node.js為例
---

Thrift屬於遠端程序呼叫(RPC，Remote Procedure Call)的一種框架，透過IDL(Interface Description Language)語言去定義Client/Server之間的接口，並自動產生適用於各程式語言的介面，讓使用者毋須擔心客戶端與伺服器端之間的通訊問題，可以在支援的語言下快速開發相關應用，以下以官網的範例簡單說明開發流程，此篇為概述，如需更詳細的資料請參考[Thrift: The Missing Guide](https://diwakergupta.github.io/thrift-missing-guide/)。

## 安裝Thrift

請至[官網](https://thrift.apache.org/download)下載執行檔即可。



## 定義.thrift檔案

#### 型別(Type)

在.thrift檔中內建的型別有：

| Type        | Description                        |
| ----------- | ---------------------------------- |
| bool        | Boolean, one byte                  |
| i8 (byte)   | Signed 8-bit integer               |
| i16         | Signed 16-bit integer              |
| i32         | Signed 32-bit integer              |
| i64         | Signed 64-bit integer              |
| double      | 64-bit floating point valye        |
| string      | String                             |
| binary      | Blob (byte array)                  |
| map<t1, t2> | Map from one type to another       |
| list<t1>    | Ordered list of one type           |
| set<t1>     | Set of unique elements of one type |

#### 引入其他.thrift檔案

可以使用`include`關鍵字在.thrift檔案中引入其他.thrift檔，Thrift會在當前的.thrift檔案所在的資料夾搜尋，也可以使用相對路徑，其會相對於在命令列中使用`-I`flag指定的路徑， 引入成功後即可透過`檔案名稱.檔案物件`的方式去存取被引入進來的檔案其中的物件。

#### 定義名稱空間(Namespace)

就像namespace在其他語言的功用，在.thrift當中可以透過`namespace`關鍵字定義名稱空間，方便管理程式碼並防止衝突，語法為：

```
namespace [language] [namespace]
```

實際範例：

```thrift
namespace cpp mySecretGarden
```

#### 定義客製化型別

使用`typedef`關鍵字定義客製化的型別，型別除了內建型別之外，也可以是一個結構(見下面說明)，語法為：

```
typedef [type | struct] [name]
```

實際範例：

```thrift
typedef double myMagicNumber
typedef someStruct myMagicStructure
```

#### 定義列舉(Enumeration)

使用`enum`關鍵字定義列舉，可以指定也可以不指定預設值，在不指定的情況下預設是從1開始遞增：

```thrift
 enum myEnum {  
	FIELD_1,  
  FIELD_2 = 100,  
  FIELD_3 = 1000
 }
```

#### 定義常數(Constant)

使用`const`關鍵字定義常數，語法為：

```
const [type] [name] = [value]
```

實際範例，複雜結構使用JSON來定義：

```thrift
const map<string, i32> COUNTRY_POPULATION = {"China": 1300000000, "USA": 300000000}
```

#### 定義結構(Structure)

使用`struct`關鍵字定義結構，語法為：

```
struct structName {
  [integer idnetifier]: [type] [name] (optinal)=[default value]
}
```

實際範例如下，如果欄位是選擇性的需要在欄位型別之前加上optional關鍵字：

```thrift
struct Point {
  1: i32 x = 0,
  2: i32 y = 0,
  3: optional string someOtherField = "I can have default value, too"
}
```

#### 定義Service

Service可以說是整個.thrift檔案中最重要的一部分，他類似於Java中的interface，只需定義signature。Thrift透過.thrift檔案產生適用於各語言的介面之後，server端只需專注在此介面的實作；client端則可以參考此介面了解如何對server端做query。

使用`service`關鍵字定義服務，語法為：

```
service [serviceName] (optional)extends [otherService]
```

綜合以上即可快速定義一個服務介面，以官網的[__turotial.thrift__](https://git-wip-us.apache.org/repos/asf?p=thrift.git;a=blob_plain;f=tutorial/tutorial.thrift)為例：

```thrift
//　引入其他.thrift檔案
include "shared.thrift"

// 定義namespace
namespace cpp tutorial
namespace d tutorial
namespace dart tutorial
namespace java tutorial
namespace php tutorial
namespace perl tutorial
namespace haxe tutorial

// 定義type
typedef i32 MyInteger

// 定義constant
const i32 INT32CONSTANT = 9853
const map<string,string> MAPCONSTANT = {'hello':'world', 'goodnight':'moon'}

// 定義enumeration，其本身只是32位元的整數，在不指定的情況下預設從1開始遞增
enum Operation {
  ADD = 1,
  SUBTRACT = 2,
  MULTIPLY = 3,
  DIVIDE = 4
}

// 定義structure
struct Work {
  1: i32 num1 = 0,
  2: i32 num2,
  3: Operation op,
  4: optional string comment,
}

/**
 * Structs can also be exceptions, if they are nasty.
 */
exception InvalidOperation {
  1: i32 whatOp,
  2: string why
}

// 定義service，這邊繼承了一開始引入的shared.thrift當中的SharedService(見下一個Code Block)
service Calculator extends shared.SharedService {
   void ping(),
   i32 add(1:i32 num1, 2:i32 num2),
   i32 calculate(1:i32 logid, 2:Work w) throws (1:InvalidOperation ouch),

   /**
    * This method has a oneway modifier. That means the client only makes
    * a request and does not listen for any response at all. Oneway methods
    * must be void.
    */
   oneway void zip()
}
```

###### shared.thrift

```thrift
/**
 * This Thrift file can be included by other Thrift files that want to share
 * these definitions.
 */
 
namespace cpp shared
namespace d share // "shared" would collide with the eponymous D keyword.
namespace dart shared
namespace java shared
namespace perl shared
namespace php shared
namespace haxe shared

struct SharedStruct {
  1: i32 key
  2: string value
}

service SharedService {
  SharedStruct getStruct(1: i32 key)
}
```



## Code Generation

準備好.thrift檔案後即可透過Thrift產生對應的程式碼，以Node.js為例：

```bash
thrift -r --gen js:node tutorial.thrift
```

執行此命令後，會在當前資料夾新增一個gen-nodejs資料夾，存放產生出來的程式碼。



## 實作Client/Server

接下來要分別實作Client/Server，以下配合上述tutorial.thrift，以官網的[Node.js example](https://thrift.apache.org/tutorial/nodejs)為例，不管是Client還是Server，皆須先從NPM上安裝Thrift的package。

```bash
npm install thrift
```

###### Client實作

```javascript
var thrift = require('thrift');
var Calculator = require('./gen-nodejs/Calculator');
var ttypes = require('./gen-nodejs/tutorial_types');

var transport = thrift.TBufferedTransport()
var protocol = thrift.TBinaryProtocol()

var connection = thrift.createConnection("localhost", 9090, {
  transport : transport,
  protocol : protocol
});

connection.on('error', function(err) {
  assert(false, err);
});

// Create a Calculator client with the connection
var client = thrift.createClient(Calculator, connection);


client.ping(function(err, response) {
  console.log('ping()');
});


client.add(1,1, function(err, response) {
  console.log("1+1=" + response);
});


work = new ttypes.Work();
work.op = ttypes.Operation.DIVIDE;
work.num1 = 1;
work.num2 = 0;

client.calculate(1, work, function(err, message) {
  if (err) {
    console.log("InvalidOperation " + err);
  } else {
    console.log('Whoa? You know how to divide by zero?');
  }
});

work.op = ttypes.Operation.SUBTRACT;
work.num1 = 15;
work.num2 = 10;

client.calculate(1, work, function(err, message) {
  console.log('15-10=' + message);

  client.getStruct(1, function(err, message){
    console.log('Check log: ' + message.value);

    //close the connection once we're done
    connection.end();
  });
});
```

###### Server實作

```javascript
var thrift = require("thrift");
var Calculator = require("./gen-nodejs/Calculator");
var ttypes = require("./gen-nodejs/tutorial_types");
var SharedStruct = require("./gen-nodejs/shared_types").SharedStruct;

var data = {};

var server = thrift.createServer(Calculator, {
  ping: function(result) {
    console.log("ping()");
    result(null);
  },

  add: function(n1, n2, result) {
    console.log("add(", n1, ",", n2, ")");
    result(null, n1 + n2);
  },

  calculate: function(logid, work, result) {
    console.log("calculate(", logid, ",", work, ")");

    var val = 0;
    if (work.op == ttypes.Operation.ADD) {
      val = work.num1 + work.num2;
    } else if (work.op === ttypes.Operation.SUBTRACT) {
      val = work.num1 - work.num2;
    } else if (work.op === ttypes.Operation.MULTIPLY) {
      val = work.num1 * work.num2;
    } else if (work.op === ttypes.Operation.DIVIDE) {
      if (work.num2 === 0) {
        var x = new ttypes.InvalidOperation();
        x.whatOp = work.op;
        x.why = 'Cannot divide by 0';
        result(x);
        return;
      }
      val = work.num1 / work.num2;
    } else {
      var x = new ttypes.InvalidOperation();
      x.whatOp = work.op;
      x.why = 'Invalid operation';
      result(x);
      return;
    }

    var entry = new SharedStruct();
    entry.key = logid;
    entry.value = ""+val;
    data[logid] = entry;

    result(null, val);
  },

  getStruct: function(key, result) {
    console.log("getStruct(", key, ")");
    result(null, data[key]);
  },

  zip: function() {
    console.log("zip()");
    result(null);
  }

});

server.listen(9090);
```



## 測試Client與Server之間的連線

以上皆準備就緒後，就可以啟動Server，接著執行client測試連線與執行結果。Thrift一個比較為人詬病的地方是，官方並沒有為各語言提供一個比較完整的文件，以Node.js為例，我也是在直接印出實際建立的connection物件之後才知道可以註冊什麼event。以上的範例Client/Server端皆為Node.js，但這完全不是必要條件，只要是支援的語言，Client/Server端的語言可以自由的搭配，Thrift會幫忙處理中間的通訊以及相容問題，讓開發者只需專注實作應用程式的邏輯。



## 參考

1. [Thrift: The Missing Guide (Must Read!)](https://diwakergupta.github.io/thrift-missing-guide/)
2. [Apache Thrift](https://thrift.apache.org/)