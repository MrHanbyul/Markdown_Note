# MakrDown Flow语法
[MarkDown 流程图语法](https://support.typora.io/Draw-Diagrams-With-Markdown/)
[flowchart.js](http://flowchart.js.org/)

```
tag=>type: content:>url[blank]
type: 	start end 
		operation subroutine  
		inputoutput	parallel

```
```flow
st=>start: Start:>http://www.baidu.com[blank]
op=>operation: Your Operation
sub=>subroutine: My Subroutine
cond=>condition: Yes or No?
io=>inputoutput: catch something...
para=>parallel: parallel tasks
e=>end: End

st->op->cond
cond(no)->para
cond(yes, right)->io->e
para(path1, bottom)->sub(right)->op
para(path2, top)->op
```



## 思维导图和MarkDown

[markmap.js](https://markmap.js.org/usage)
[Markdown 与思维导图的相互转换](https://blog.csdn.net/wirelessqa/article/details/105673201)

```markmap-lib
# 毕小烦

## 基本信息

- 年龄：保密
- 性别：男

### 联系方式

* [博客](https://blog.csdn.net/wirelessqa)
* [微博](https://weibo.com/wirelessqa)
* 微信公众号请搜索：毕小烦

## 出版作品

- [了不起的 Markdown](https://item.jd.com/12669274.html)
- 精通 AndroidStudio

## 兴趣爱好

- *啤酒*
- **咖啡**
- **跑步**
---
- 看书
- 写作
```



