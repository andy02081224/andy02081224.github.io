---
layout: post
category: Programming
tags: [JavaScript]
title: 在JavaScript中使用Regex
---

這篇是以前寫在evernote的筆記，以下大部分內容是從MDN上整理而來：

## 宣告Regex

###### 兩種宣告方式

- 使用RegExp實字
   - `var re = /ab+c/gim;`
   - 執行時期已編譯完成，效能較好
- 使用RegExp 物件的建構式
   - `var re = new RegExp("ab+c", "gim");`
   - 執行時期編譯，較有彈性

###### 常用Flags

1. g: global search
2. i: 預設是case sensitive，加上這個flag則為case insensitive
3. m: 預設上如果使用「^」或「$」這類特殊字元，他match的是字串的開始和結束，加上m flag之後「^、$」也會match多行字串每一行的開始與結束

###### 判斷原則

當Regex仍然會異動或還不確定內容時，使用RegExp建構式較為合適，其他情況使用已預先編譯過、效能較好之RegExp實字;另外習慣將正規表示式透過變數去參考，這樣不但可以透過此參考讀取Regex object上的prototype properties，也可重複使用不需重新創建。

## 撰寫Regex

###### 比對字元

- 簡易字元
   - 直接比對
- 特殊字元(或與簡易字元混用)
   - 更有彈性、簡潔的比對
   - [特殊字元整理](https://www.debuggex.com/cheatsheet/regex/javascript) (注意: 使用RegExp建構式時，因為是透過字串去指定pattern，反斜線「\」在字串中是escape character，因此在使用這些特殊字元時反斜線要寫成「\\\\」)

###### 使用小括號(Parentheses)獲取匹配的子字串

在Regex使用的小括號稱為capturing parentheses，可以用他來獲取匹配字串的子字串，另外在宣告Regex、底下介紹到的replace方法中，還可以透過符號搭配數字來參考到他:

1. 宣告Regex: 使用\1, \2,  \n來參考，如`/(foo) (bar) \1 \2/`，後面的\1\2代表的分別是前面捕捉到的「 foo」 、「 bar」 子字串，等同於`/foo bar foo bar/`
2. String.prototype.replace方法的replacement part(第2個參數): 使用$1, $2, $n來參考，如`'Hello World!'.replace( /(Hello) (World)/, '$2, $1' )`，$1, $2分別參考到「Hello」已及「 World」

小括號在Regex當中也有group的意思，如`foo{1,}`當中最後的一到多{1,}只會作用在foo中最後的那個o，如果想要整個foo都可以包括在「一到多」(ex. foofoofoo整個字串都符合)的範圍裡，就可以使用小括號將他圈起如`(foo){1,}`，另外要注意的是，如果僅想要小括號是group而不具有捕捉的功能，可以在小括號開始之後加上「?:」，如`(?:foo){1,}`，這樣這個括號就僅會被當作一個group，這種情況下的小括號又被稱為non-capturing parentheses

## 使用Regex

###### 常用的Regex的方法

- RegExp.prototype

| 方法                                       | 說明                                       | 備註                                       |
| ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| [exec](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec) | `regexObj.exec(str)`：提供Regex object對指定字串做搜尋，並回傳搜尋結果，如果有符合的字串，回傳結果陣列，反之則回傳null | 使用g flag，可以持續呼叫exec搜尋下一筆符合的字串，下一筆搜尋會從紀錄在Regex object上的lastIndex繼續開始， 一般常用的做法是使用while loop檢查exec的回傳結果是否為null，如果不為null則持續進行搜尋；在不使用g flag的情況下，搜尋只會執行一次，並回傳結果陣列(如果結果不為null)，其內容依序為：第一筆符合的字串、他的Captured groups(每個captured group為一個獨立的參數)、符合字串在整個字串中的位置以及整個字串 |
| [test](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/test) | `regexObj.test(str)`：提供Regex object對指定字串做搜尋，並依據是否有符合字串回傳true或false |                                          |

- String.prototype

| 方法                                       | 說明                                       | 備註                                       |
| ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| [match](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/match) | `str.match(regexp)`：提供字串根據指定的Regex object做搜尋，並回傳搜尋結果，如果有符合的字串，回傳結果陣列，反之則回傳null | 如果不使用g flag，回傳與RegExp.prototype.exec相同的搜尋結果；使用g flag則回傳包含所有符合字串的陣列，但是不包含Captured groups |
| [search](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/search) | `str.search(regexp)`：提供字串根據指定的Regex object做搜尋，並回傳搜尋結果，如果有符合的字串，回傳第一個符合結果的index，反之則回傳-1 | 此方法與test方法類似，適合想要知道符合字串位置時使用             |
| [replace](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace) | `str.replace(regexp\|substr, newSubStr\|function)`：提供字串根據指定的Regex object或字串做搜尋，如果第二個參數為字串，會將搜尋結果中符合條件的字串以此參數字串部份或全部取代並回傳新字串(不影響原本的字串)；如果第二個參數是函式，使用函式處理搜尋結果，在此函式中回傳的字串即為取代字串 | 原本可以在第三個參數指定是否做global搜尋的用法已經在部分瀏覽器被deprecated，要做global搜尋的話第一個參數必須使用regex object並加上g flag；在做global搜尋時，當第二個參數是函式時，只要有符合之子字串函式即會被呼叫，有可能執行多次，每次執行會帶有該次符合子字串的相關資訊，因此函式不只是可以透過回傳值來取代搜尋結果，也可以用來處理字串，這個callback函式可以取得的參數與執行exec方法得到的結果相同 |
| [split](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/split) | 基本語法:str.split([regexp\|substr[, limit]])第二個參數限制字串切割的次數；此函式的功能是把seperator(第一個參數)從字串中去除並把被區隔開的子字串一個ㄧ個放進陣列中後回傳此陣列 |                                          |

###### 方法的使用時機(MDN)

> When you want to know whether a pattern is found in a string, use the test or search method; for more information (but slower execution) use the exec or match methods. If you use exec or match and if the match succeeds, these methods return an array and update properties of the associated regular expression object and also of the predefined regular expression object, RegExp. If the match fails, theexec method returns null (which coerces to false).   

## 參考

1. [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp)
2. JavaScript Ninja