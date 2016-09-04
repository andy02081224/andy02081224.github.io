---
layout: post
category: Programming
published: true
tags: [JavaScript, 工作中]
title: 工作中：Timers in JavaScript
---

這篇文章屬於工作中系列，主要用來記錄我在工作上實際面對到的問題

## 問題

最近在寫Node時要做一個每秒定期會做20至30次的工作，但是用了內建的timer之後發現並不是很精準，比如說，當有某個片段的程式碼如下，我期望的會是每秒會印30上下次log，但實際執行之後發現在十秒過後印出的Count數跟預期的有些落差

```javascript
let count = 0;

let someJob = setInterval(() => {
  console.log(++count);
}, 33)

setTimeout(() => {
  clearInterval(someJob);
}, 10000)
```

## 原因

#### 單執行緒以及Call Stack

在JavaScript當中使用Timer（`setTimeout, setInterval`），尤其是在時間間距較小的情況下，時常會發現他並不完全準確，其原因來自於JavaScript單執行緒的特性。

所謂的單執行緒指的是JavaScript一次只能執行一個Execution Context（一個紀錄目前Function執行環境的物件），而為了記錄整個程式執行的狀況，在JavaScript Interpreter當中會有一個Stack（常被稱作Execution Context Stack或Call Stack）的資料結構專門用來儲存這些Execution Context。

新的Execution Context會隨著Function呼叫被推至Stack的頂端，而那個在頂端的Execution Context就是所謂的Active Execution Context，也就是當前正在執行，也是唯一被執行的Context，Non-blocking的程式碼如timer在被推至Stack上之後即會交由另外的機制去處理、計算Timer上的時間，同時被pop掉，不會阻擋其他Execution Context的執行。

#### Message Queue和Event Loop

非同步程式碼在Call Stack上被pop之後仍然會有機制去持續觀察他的觸發條件，這裡的timer在預設的時間到達之後其處理函式(callback)會被放到一個叫做Message Queue (Message Queue、Task Queue名字很多...)的資料結構(First in, First Out)當中。

JavaScript會有一個叫做Event Loop的機制持續去觀察Call Stack與Queue，當他觀察到Call Stack已經沒有任何正在執行的Execution Context，就會從Message Queue當中取出處理函式（在這邊就是timer的callback function），把它放到Call Stack上去處理。

這就是會什麼timer在JavaScript當中未必會準確的原因，當我們定義的時間到了之後，其處理函式只是被放進Message Queue當中而非直接執行，如果這時Call Stack仍然還有執行的Execution Context或Message Queue當中在timer的處理函式之前還有其他還沒被執行的Callback function時，這個timer的處理函式能做的事就只有等，等到Call Stack空了、Message Queue中他排到第一，他才能被Event Loop放到Call Stack去執行。

這也是為什麼`setTimeout(() => console.log('我未必會立即執行'), 0)`未必會立即執行的原因，而也有人運用這樣的特性讓JavaScript也有類似其他程式語言中wait()的功能。

#### setTimeout與setInterval

根據上面的理解就可以了解以下程式碼的差異

```javascript
setTimeout(function repeat() {
  setTimeout(repeat, 10)
}, 10)

setInterval(function() {/* Some Code */}, 10)
```

第一個setTimeout的意思是在10ms之後幫我把我的Callback Function放到Message Queue上等待執行，所以實際執行Callback的時間一定是大於等於10ms，而在Callback當中又註冊另外一個Timeout，這個Timeout的Callback執行時間又會是10ms後＋在Message Queue當中的等待時間，意思就是前後Timer會互相牽連，時間有可能一直不斷延遲下去。

第二個setInterval的意思則是不管前面的Timer有沒有執行、何時執行，他所做的就是每10ms將Callback Function放到Message Queue當中，前後Timer互不影響。

## 解法

所以在真的需要精準的Timer時，除了用一些別人寫好的現成模組外，我在[Reddit](https://www.reddit.com/r/learnjavascript/comments/3aqtzf/issue_with_setinterval_function_losing_accuracy/)上找到了以下這段比較好理解的解決方式，他基本上就是利用「截長補短」的特性，假如間隔的設定是30ms，但是第一個間隔實際卻花了32ms，他就動態調整下一次的間隔為28ms，以此類推不斷下去。

```javascript
function customSetInterval(func, time){
    var lastTime = Date.now(); // 設定interval時的時間
    var lastDelay = time; // 一開始設定的interval
    var outp = {};

    function tick(){
        func();

        var now = Date.now(); // 實際執行callback時的時間
        var dTime = now - lastTime; // 計算從上一次執行callback到現在實際花了多少時間

        lastTime = now;
      	
        // 下一次設定的間隔是：一開始設定的間隔 + (上一次本來該花多少時間 - 實際花了多少時間)
      	// = 一開始設定的間隔 + 應該調整的誤差
        lastDelay = time + lastDelay - dTime;
        outp.id = setTimeout(tick, lastDelay);
    }
    outp.id = setTimeout(tick, time);

    return outp;
}

var x = customSetInterval(...); 
clearTimeout(x.id)
```



## 參考

1. [What is the Execution Context & Stack in JavaScript?](http://davidshariff.com/blog/what-is-the-execution-context-in-javascript/)
2. [(影片) Philip Roberts: What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
3. (書) JavaScript Ninja Ch.8: Taming threads and timers