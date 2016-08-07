---
layout: post
category: Programming
published: true
tags: [JavaScript]
title: 同步tab之間的登入狀態
---
之前會需要實作一項功能：如果使用者在同一個瀏覽器下的兩個分頁都開啟已經在登入狀態的應用程式時，如果在其中一個分頁登出，那另外一個分頁需要提醒使用者登入狀態已經更新，需要重新整理。

以前的實作是透過WebSocket從server發送事件給特定的客戶端讓他可以顯示提醒訊息，但是這樣的提醒僅限於「登出」，如果使用者開啟的是兩個尚未登入的應用程式然後從其中一個分頁登入，因為開啟時都是沒有登入的一般狀態，因此server沒辦法針對特定客戶端發出通知。

今天在偶然的情況下發現GitHub上在上述的情況下除了登出，在登入時另外一個分頁也會顯示提醒訊息，看了一下GitHub的resource，發現GitHub在LocalStorage當中有一個logged-in的Key，重複登入登出的操作後可以發現在登入的狀態下此Key的字串值為「true」，而在登出裝態此值為「false」:

###### 登出狀態
[![blog-screenshot.jpg]({{site.baseurl}}/img/2016-07-29-sync-authentication-status-using-localstorage/github-localstorage-logged-in-false.jpg)]({{site.baseurl}}/img/2016-07-29-sync-authentication-status-using-localstorage/github-localstorage-logged-in-false.jpg)

###### 登入狀態
[![blog-screenshot.jpg]({{site.baseurl}}/img/2016-07-29-sync-authentication-status-using-localstorage/github-localstorage-logged-in-true.jpg)]({{site.baseurl}}/img/2016-07-29-sync-authentication-status-using-localstorage/github-localstorage-logged-in-true.jpg)

將使用者目前的登入狀態存在客戶端，就可以透過監聽LocalStorage狀態的改變，來達成在多個分頁開啟應用程式的狀況下，當在其他分頁改變登入狀態時，可以提醒使用者的機制。

以下是利用browser的`storage`事件來監聽LocalStorage的狀態進而達成以上所說的情境：

```javascript
window.addEventListener('storage', (event) => {
  // 在使用者登出/登入時，透過改變LocalStorage的裝態來觸發此事件
  let text = '';
  
  if (event.key == 'logged-in' && event.newValue == 'true') {
	  text = 'logged in';
  }
  else if (event.key == 'logged-in' && event.newValue == 'false') {
	  text = 'logged out';
  }
  else {
	  return;
  }
	
  alert('You have ' + text + ' at another page. Please referesh your browser to update the page');
});
```