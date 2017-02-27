---
layout: post
category: Programming
published: true
tags: [JavaScript]
title: 使用Reduce method串接Promise
---

最近在看Google Web Fundamental上的[這篇](https://developers.google.com/web/fundamentals/getting-started/primers/promises)複習Promise時，發現一個自己沒怎麼想過的用法：用Array的reduce method串接Promise。

這邊假設一個情境:我現在想透過GitHub API將自己幾個repo的commit紀錄撈回來，parse之後在畫面上照順序顯示repo的名稱、此repo所有commit紀錄的hash與message，使用上述的reduce method將所有promise requests串接在一起:


HTML

```html
<section id="commit-history"></section>
```

JS

```
let myRepos = ['baskeeper', 'exe-andyteki', 'git-memo'];

function getJSON(url) {
  return fetch(url)
  .then(function(response) {
    return response.json()
  }).then(function(json) {
    return json
  }).catch(function(ex) {
    throw ex;
  })
}

function setCommitHistoryToUI(repoName, repoCommits) {
  let domCommitHistoryBlock = document.getElementById('commit-history');
  
  let domTitle = document.createElement('h3');  
  domTitle.textContent = repoName;
  domCommitHistoryBlock.appendChild(domTitle);
  
  repoCommits.forEach((commit) => {
    let sha = commit.sha.substr(0, 7);
    let message = commit.commit.message;
    let domCommitHistory = document.createElement('p');
    
    domCommitHistory.textContent = `${sha}: ${message}`;
    domCommitHistoryBlock.appendChild(domCommitHistory);
  });
}

// 一開始使用一個直接resolve的promise當作reduce的起始值方便
myRepos.reduce((finishedRequests, repoName) => {
  return finishedRequests
  	.then(() => getJSON(`https://api.github.com/repos/andy02081224/${repoName}/commits`))
  	.then((repoCommits) => setCommitHistoryToUI(repoName, repoCommits))
}, Promise.resolve());
```

這時候會面臨到一個問題：我的requests是依序打出去的，也就是說在第一個請求完成之後才會接著發出第二個請求，接著是第三個，以此類推，浪費了瀏覽器提供的可以打parellel requests的特性，這一次先將所有請求發出，再利用Promise.all method來等待所有請求完成：

```
let allRepoCommitsRequests = myRepos.map((repoName) => {   
  return getJSON(`https://api.github.com/repos/andy02081224/${repoName}/commits`);
});

Promise.all(allRepoCommitsRequests)
    .then((allRepoCommits) => {
      allRepoCommits.forEach((repoCommits, i) => {
        setCommitHistoryToUI(myRepos[i], repoCommits)
      });
});
```

但是，又有一種狀況是，我們希望可以同時發送請求，並在結果回來時依照順序顯示。

現在雖然可以同時發送請求，但也因為Promise.all的關係，需要等到所有請求都完成後才處理資料並把他設定到畫面上，為了達成前述的scenario，以下先把所有請求發送出去後，在依序把它串接起來：

```
let allRepoCommitsRequests = myRepos.map((repoName) => {   
  return getJSON(`https://api.github.com/repos/andy02081224/${repoName}/commits`);
});

allRepoCommitsRequests.reduce((finishedRequests, request) => {
  return finishedRequests
  	.then(() => request)
  	.then((repoCommits) => setCommitHistoryToUI(myRepos[i], repoCommits))
}, Promise.resolve());
```

以上三個範例的推進完全是仿照一開始提到的文章，這邊僅是改寫成我自己的例子；個人的理解是，他們之間不是由好到不好的層級差別，而是case by case的應用，端看現在的scenario適合哪個。

