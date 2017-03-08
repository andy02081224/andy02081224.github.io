---
layout: post
category: Programming
published: true
tags: [JavaScript]
title: DOM insertion效能問題
---
之前面試的時候被問到一個問題：想在短時間內插入大量的DOM到頁面上，有什麼（有效率的）方法？

很多人都聽過，操作DOM是很「昂貴」的，這主要肇因於DOM操作時常會伴隨[browser reflow](https://dev.opera.com/articles/efficient-javascript/?page=3#reflow)，也就是當DOM的內容不停變動，browser就需要持續去計算新的layout位置並repaint。

插入DOM的方式有百百種，但是在複雜度差不多的情況下，不同方式的寫法可能會造成百倍以上的效能差距，這邊有一個很有趣的performance測試：[DOM insertion benchmarking](http://jsperf.com/render-in-memory-vs-direct-dom-insertion/7)，其中分別使用jQuery與DOM API來達成DOM insertion，可以發現在相對簡單易懂的幾種方法間，最差的case每秒執行效率尚不及最好的百分之一（測試環境：OS X 10.12, Chrome 56）。

比較差的幾個版本，不管是使用jQuery還是DOM API，無二致的都是採用一個一個append上去的方式，這種逐次更新的方式會在短時間內引起大量的browser reflow，影響效能：

```javascript
var i = 20;
var cnt = document.getElementById('container');
while (i--) {
  cnt.innerHTML += "<div>a new div</div>";
};
```

而有趣的是，效能最好的版本同樣是使用如上的innerHTML property，差別在於他先把所有的HTML string放到一個array，在一次把這個array內所有的內容塞給innerHTML：

```javascript
var i = 20;
var cnt = document.getElementById('container');
var aHTML = [];
while (i--) {
  aHTML.push("<div>a new div</div>");
};
cnt.innerHTML = aHTML.join("");
```

由此可知，要提升DOM insertion的效能，減少DOM的更動次數是一個關鍵。

這個benchmarking的test case當中還有用到另外兩個有趣的API：`Document.createDocumentFragment()`以及`Node.cloneNode()`，透過這兩個API，我們可以將DOM node當成一個「片段」，一個一個append到由`Document.createDocumentFragment()`創造出來的DocumentFragment物件上，並把這個Fragment物件上的所有片段一次append至真正的DOM element：

```javascript
var i = 20;
var frag = document.createDocumentFragment();
var div = document.createElement('div');
div.textContent = 'a new div';

while (i--) {
  frag.appendChild(div.cloneNode(true));
}

var cnt = document.getElementById('container');
cnt.appendChild(frag);
```

這邊提高效率的原因來自於DocumentFragment上的DOM片段並不屬於DOM Tree而是保存在記憶體當中，因此append新的DOM到DocumentFragment當中並不會造成browser reflow。

要特別注意的是上面`cloneNode()`API的使用，這邊必須寫成`frag.appendChild(div.cloneNode(true))`而不是`frag.appendChild(div)`的原因是因為不管執行後者多少次，因為是同一個element，不會重複加入，到最後DocumentFragment還是只有一個div，而前者透過cloneNode()的方式可以不斷複製出內容相同的全新DOM Node讓DocumentFragment上的片段持續增加。

##### 補充：innerHTML +=與appendChild()的差異

基本上Stackoverflow上的[這篇](http://stackoverflow.com/questions/2305654/innerhtml-vs-appendchildtxtnode)已經講得很好了，基本上來說因為innerHTML即使看起來是用「+=」把新的HTML string append上去，但是實際上還是把目標的內容全部再打掉重練，相對的，appendChild()只會造成局部的重建，因此如果是要在已經有內容的目標物件上append新內容，推薦使用appendChild()。

##### Reference

1.[Minimizing browser reflow](https://developers.google.com/speed/articles/reflow?csw=1)

2.[Repaints and Reflows: Manipulating the DOM responsibly](http://blog.letitialew.com/post/30425074101/repaints-and-reflows-manipulating-the-dom)

3.[Efficient JavaScript - Repaint and reflow](https://dev.opera.com/articles/efficient-javascript/?page=3#reflow)

4.[MDN - Document.createDocumentFragment()](https://developer.mozilla.org/zh-TW/docs/Web/API/Document/createDocumentFragment)

5.[MDN - Node.cloneNode()](https://developer.mozilla.org/zh-TW/docs/Web/API/Node/cloneNode)

   ​