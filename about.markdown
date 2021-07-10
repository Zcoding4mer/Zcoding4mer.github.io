---
layout: page
title: About
permalink: /about/
---

{% highlight javascript %}
while (live) {
  keepLearning()
  new Promise((resolve, reject) => {
    if (haveTime){
      resolve('travel round the world')
    } else {
      reject('have no time')
    }
  }).then((result) => {
    seeBigWorld()
    meetSomeone()
    console.log('happy')  
  }).catch((reason) => {
    console.error('attention')
    thinkWhy()
    workHarder()
  })
}
{% endhighlight %}
