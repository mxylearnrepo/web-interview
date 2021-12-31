# 图片懒加载
```js
let imgList = [...document.querySelectorAll('img')]
let length = imgList.length

// 自执行是要返回一个 function 哟
const imgLazyLoad = (function() {
  let count = 0 // 用于统计是否加载完全部图片了
    
  return function() {
    let deleteIndexList = []
    imgList.forEach((img, index) => {
      let rect = img.getBoundingClientRect() // 重要的方法
      if (rect.top < window.innerHeight) { // 说明滚进来了
        img.src = img.dataset.src
        // 思考为什么不直接在 forEach 里 imgList.splice ? (因为会漏)
        deleteIndexList.push(index) // 用于从 imgList 摘除已加载的图片索引
        count++
        if (count === length) {
          document.removeEventListener('scroll', imgLazyLoad) // 加载完全部, 移除滚动监听
        }
      }
    })
    imgList = imgList.filter((img, index) => !deleteIndexList.includes(index))
  }
})()

// 这里最好加上防抖处理
document.addEventListener('scroll', imgLazyLoad)
```
