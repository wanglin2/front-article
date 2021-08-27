众所周知，很多社区都是有内容审核机制的，除了第一次发布，后续的修改也需要审核，最粗暴的方式当然是从头再看一遍，但是编辑肯定想弄死你，显然这样效率比较低，比如就改了一个错别字，再看几遍可能也看不出来，所以如果能知道每次都修改了些什么，就像`git`的`diff`一样，那就方便很多了，本文就来简单实现一个。



# 求最长公共子序列

想要知道两段文本有什么差异，我们可以先求出它们的公共内容，剩下的就是被删除或新增的。在算法中，这是一道经典的题目，力扣上就有这道题[1143. 最长公共子序列](https://leetcode-cn.com/problems/longest-common-subsequence/)，题目描述如下：

![image-20210816195639935.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e4f626b4c1d41afb963dee2ab999e32~tplv-k3u1fbpfcp-watermark.image)

这种求最值的题一般都是使用动态规划来做，动态规划比较像推理题，可以使用递归来自顶向下求解，也可以使用`for`循环自底向上来做，使用`for`循环一般会使用一个叫`dp`的备忘录来存储信息，具体使用几维数组要视题目而定，这道题因为有两个变量（两个字符串的长度）所以我们使用二维数组，我们定义`dp[i][j]`表示`text1`从`0-i`的子串和`text2`从`0-j`的子串的最长公共子序列长度，接下来需要考虑边界情况，首先当`i`为`0`的时候`text1`的子串为空字符串，所以无论`j`为多少最长公共子序列的长度都为`0`，`j`为`0`的情况也是一样，所以我们可以初始化一个初始值全部为`0`的`dp`数组：

```js
let longestCommonSubsequence = function (text1, text2) {
    let m = text1.length
    let n = text2.length
    let dp = new Array(m + 1)
    dp.forEach((item, index) => {
        dp[index] = new Array(n + 1).fill(0)
    })
}
```

当`i`和`j`都不为`0`的情况下，需要分几种情况来看：

1.当`text1[i - 1] === text2[j - 1]`时，说明这两个位置的字符相同，那么它们肯定在最长子序列里，当前最长的子序列就依赖于它们前面的子串，也就是`dp[i][j] = 1 + dp[i - 1][j - 1]`；

2.当`text1[i - 1] !== text2[j - 1]`时，很明显`dp[i][j]`完全取决于之前的情况，一共有三种：`dp[i - 1][j - 1]`、`dp[i][j - 1]`、`dp[i - 1][j]`，不过第一种情况可以排除掉，因为它显然不会有后面两种情况长，因为后面两种都比第一种多了一个字符，所以可能长度会多`1`，那么我们取后面两种情况的最优值即可；

接下来我们只要一个二重循环遍历二维数组的所有情况即可：

```js
let longestCommonSubsequence = function (text1, text2) {
    let m = text1.length
    let n = text2.length
    // 初始化二维数组
    let dp = new Array(m + 1).fill(0)
    dp.forEach((item, index) => {
        dp[index] = new Array(n + 1).fill(0)
    })
    for(let i = 1; i <= m; i++) {
        // 因为i和j都是从1开始的，所以要减1
        let t1 = text1[i - 1]
        for(let j = 1; j <= n; j++) {
            let t2 = text2[j - 1]
            // 情况1
            if (t1 === t2) {
                dp[i][j] = 1 + dp[i - 1][j - 1]
            } else {// 情况2
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1])
            }
        }
    }
}
```

`dp[m][n]`的值就是最长公共子序列的长度，但是只知道长度并没啥用，我们要知道具体哪些位置才行，需要再来一次递归，为什么不能在上述循环里面`t1 === t2`的分支里收集位置呢，因为两个字符串的所有位置都会进行两两比较，当存在多个相同的字符时会存在重复，就像下面这样：

![image-20210817191053130.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6367697e46eb422688950ac6d4ae494f~tplv-k3u1fbpfcp-watermark.image)

我们定义一个`collect`函数，递归判断`i`和`j`位置是否在最长子序列里，比如对于`i`和`j`位置，如果`text1[i - 1] === text2[j - 1]`，那么显然这两个位置在最长子序列内，接下来只要判断`i - 1`和`j - 1`的位置，以此类推，如果当前位置不相同则我们可以根据`dp`数组来判断，因为此时我们已经知道整个`dp`数组的值了：

![image-20210817194735329.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58a3693376644913adae0cdd22d76864~tplv-k3u1fbpfcp-watermark.image)

所以不需要再去尝试每个位置，也就不会造成重复，比如`dp[i - 1] > dp[j]`，那么接下来要判断的就是`i-1`和`j`位置，否则判断`i`和`j-1`位置，递归结束的条件是`i`和`j`有一个已经到达`0`的位置：

```js
let arr1 = []
let arr2 = []
let collect = function (dp, text1, text2, i, j) {
    if (i <= 0 || j <= 0) {
        return
    }
    if (text1[i - 1] === text2[j - 1]) {
        // 收集两个字符串里相同字符的索引
        arr1.push(i - 1)
        arr2.push(j - 1)
        return collect(dp, text1, text2, i - 1, j - 1)
    } else {
        if (dp[i][j - 1] > dp[i - 1][j]) {
            return collect(dp, text1, text2, i, j - 1)
        } else {
            return collect(dp, text1, text2, i - 1, j)
        }
    }
}
```

结果如下：

![image-20210817202220822.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f84140f71644918afe564687ccf3e76~tplv-k3u1fbpfcp-watermark.image)

可以看到是倒序的，如果不喜欢也可以排个序：

```js
arr1.sort((a, b) => {
    return a - b
});
arr2.sort((a, b) => {
    return a - b
});
```

到这里依然没有结束，我们还得根据最长子序列来计算出删除和新增的位置，这个比较简单，直接遍历一下两个字符串，不在`arr1`和`arr2`里的其他位置的字符就是被删掉的或新增的：

```js
let getDiffList = (text1, text2, arr1, arr2) => {
    let delList = []
    let addList = []
    // 遍历旧的字符串
    for (let i = 0; i < text1.length; i++) {
        // 旧字符串里不在公共子序列里的位置的字符代表是被删除的
        if (!arr1.includes(i)) {
            delList.push(i)
        }
    }
    // 遍历新字符串
    for (let i = 0; i < text2.length; i++) {
        // 新字符串里不在公共子序列里的位置的字符代表是新增的
        if (!arr2.includes(i)) {
            addList.push(i)
        }
    }
    return {
        delList,
        addList
    }
}
```

![image-20210818094131108.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/57bde52f7c04467da639390624b2cd86~tplv-k3u1fbpfcp-watermark.image)


# 标注删除和新增

公共子序列和新增删除的索引我们都知道了，那么就可以把它给标注出来，比如删除的用红色背景，新增的用绿色背景，这样一眼就能肯定哪里改变了。

简单起见，我们把新增和删除都在同一段文字上显示出来，就像这样：

![image-20210818094506055.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69b9b845f9a34ff68880eab7d5b2feca~tplv-k3u1fbpfcp-watermark.image)

假设有两段需要比较的文本，每段文本内部都以`\n`分隔来换行，我们先把它们分割成数组，然后再依次两两进行比较，如果新旧文本相等那么直接添加到显示的数组里，否则我们在新文本基础上操作，如果某个位置的字符是新增的那么给它包裹一个新增的标签，被删除的字符也在新文本里找到对应的位置并包裹一个标签再插进去，模板部分是这样的：

```html
<template>
  <div class="content">
    <div class="row" v-for="(item, index) in showTextArr" :key="index">
      <span class="rowIndex">{{ index + 1 }}</span>
      <span class="rowText" v-html="item"></span>
    </div>
  </div>
</template>
```

然后进行两两比较：

```js
export default {
    data () {
        return {
            oldTextArr: [],
            newTextArr: [],
            showTextArr: []
        }
    },
    mounted () {
        this.diff()
    },
    methods: {
        diff () {
            // 新旧文本分割成数组
            this.oldTextArr = oldText.split(/\n+/g)
            this.newTextArr = newText.split(/\n+/g)
            let len = this.newTextArr.length
            for (let row = 0; row < len; row++) {
                // 如果新旧文本完全相同就不用比较了
                if (this.oldTextArr[row] === this.newTextArr[row]) {
                    this.showTextArr.push(this.newTextArr[row])
                    continue
                }
                // 否则计算新旧文本的最长公共子序列的位置
                let [arr1, arr2] = longestCommonSubsequence(
                    this.oldTextArr[row],
                    this.newTextArr[row]
                )
                // 进行标注操作
                this.mark(row, arr1, arr2)
            }
        }
    }
}
```

`mark`方法用来生成最终带差异信息的字符串，先通过上面的`getDiffList`方法获取到删除和新增的索引信息，因为我们是在新文本的基础上进行，所以对于新增的操作比较简单，直接遍历新增的索引，然后找到新字符串里对应位置的字符，前后都拼接上标签元素的字符即可：

```js
/*
oldArr：旧文本的最长公共子序列索引数组
newArr：新文本的最长公共子序列索引数组
*/
mark (row, oldArr, newArr) {
    let oldText = this.oldTextArr[row];
    let newText = this.newTextArr[row];
    // 获取删除和新增的位置索引
    let { delList, addList } = getDiffList(
        oldText,
        newText,
        oldArr,
        newArr
    );
    // 因为添加的span标签也会占位置，所以会导致我们的新增索引发生偏移，需要减去标签所占的长度来修正
    let addTagLength = 0;
    // 遍历新增位置数组
    addList.forEach((index) => {
        let pos = index + addTagLength;
        // 截取当前位置前面的字符串
        let pre = newText.slice(0, pos);
        // 截取后面的字符串
        let post = newText.slice(pos + 1);
        newText = pre + `<span class="add">${newText[pos]}</span>` + post;
        addTagLength += 25;// <span class="add"></span>的长度为25
    });
    this.showTextArr.push(newText);
}
```

效果如下：

![image-20210818181744111.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f63b0efaae2345f595a612fa630bdb9c~tplv-k3u1fbpfcp-watermark.image)

![image-20210818181753703.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38edf2713f874becba1425e88466b480~tplv-k3u1fbpfcp-watermark.image)

删除稍微会麻烦一点，因为显然被删除的字符在新文本里是不存在的，我们要找出如果它没被删的话它应该在哪里，然后在这里再把它插回去，我们画图来看：

![image-20210818183712848.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e63ce1d6b5b241e99064cadbca602226~tplv-k3u1fbpfcp-watermark.image)

先看被删掉的`闪`，它在旧字符串里的位置是`3`，通过最长公共子序列，我们可以找到它前面的字符在新列表里的索引，那么很明显该索引后面就是该被删除字符在新字符串里的位置：

![image-20210818184131840.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99648c3c6a9445f395200d47b59dfebf~tplv-k3u1fbpfcp-watermark.image)

先写一个函数来获取被删除字符在新文本里的索引：

```js
getDelIndexInNewTextIndex (index, oldArr, newArr) {
    for (let i = oldArr.length - 1; i >= 0; i--) {
        if (index > oldArr[i]) {
            return newArr[i] + 1;
        }
    }
    return 0;
}
}
```

接下来就是计算在字符串里具体的位置，对于`闪`来说它的位置计算如下：

![image-20210818185833500.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/46437081cf1044bc97ff8f2a49b7f826~tplv-k3u1fbpfcp-watermark.image)

```js
mark (row, oldArr, newArr) {
    // ...

    // 遍历删除的索引数组
    delList.forEach((index) => {
        let newIndex = this.getDelIndexInNewTextIndex(index, oldArr, newArr);
        // 前面新增的字符数量
        let addLength = addList.filter((item) => {
            return item < newIndex;
        }).length;
        // 前面没有变化的字符数量
        let noChangeLength = newArr.filter((item) => {
            return item < newIndex;
        }).length;
        let pos = addLength * 26 + noChangeLength;
        let pre = newText.slice(0, pos);
        let post = newText.slice(pos);
        newText = pre + `<span class="del">${oldText[index]}</span>` + post;
    });

    this.showTextArr.push(newText);
}
```

到这里`闪`的位置就知道了，看效果：

![image-20210818190258024.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5994bea19f04420980766cf062f67b54~tplv-k3u1fbpfcp-watermark.image)

可以看到后面已经乱了，原因很简单，对于`晶`来说，新插入的`闪`所占的位置我们没有把它加上：

```js
// 插入的字符所占的位置
let insertStrLength = 0;
delList.forEach((index) => {
    let newIndex = this.getDelIndexInNewTextIndex(index, oldArr, newArr);
    let addLength = addList.filter((item) => {
        return item < newIndex;
    }).length;
    let noChangeLength = newArr.filter((item) => {
        return item < newIndex;
    }).length;
    // 加上新插入字符所占的总长度
    let pos = insertStrLength + addLength * 26 + noChangeLength;
    let pre = newText.slice(0, pos);
    let post = newText.slice(pos);
    newText = pre + `<span class="del">${oldText[index]}</span>` + post;
    // <span class="del">x</span>的长度为26
    insertStrLength += 26;
});
```

到这里我们草率的`diff`工具就完成了：

![image-20210818191126696.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/424ce79fcede4a70a22c197a1ea6f3ff~tplv-k3u1fbpfcp-watermark.image)

# 存在的问题

相信聪明的你一定发现上述实现是有问题的，如果我把某行完整的删掉了，或者完整的新增了一行，那么首先新旧的行数就不一样了，先修复一下`diff`函数：

```js
diff () {
    this.oldTextArr = oldText.split(/\n+/g);
    this.newTextArr = newText.split(/\n+/g);
    // 如果新旧行数不一样，用空字符串来补齐
    let oldTextArrLen = this.oldTextArr.length;
    let newTextArrLen = this.newTextArr.length;
    let diffRow = Math.abs(oldTextArrLen - newTextArrLen);
    if (diffRow > 0) {
        let fixArr = oldTextArrLen > newTextArrLen ? this.newTextArr : this.oldTextArr;
        for (let i = 0; i < diffRow; i++) {
            fixArr.push('');
        }
    }
    // ...
}
```

如果我们是最后一行新增了或删除了，那么问题不大：

![image-20210818192107232.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/183544a5f72f482eac5752d0620eec80~tplv-k3u1fbpfcp-watermark.image)

但是，如果是中间的某行新增或删除了，那么该行后面所有的`diff`都会失去意义：

![image-20210818192342057.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd5069f6c1b9453a9e0b641be3367d0a~tplv-k3u1fbpfcp-watermark.image)

原因很简单，删除了某一行，导致后面的两两对比刚好错开了，这咋办呢，一种思路是通过发现某行被删除了或某行是新增的，然后修正对比的行数，另一种方法是不再每一行单独`diff`，而是直接`diff`整个文本，这样就无所谓删除新增行了。

第一种思路反正笔者搞不定，那就只能看第二种了，我们删掉通过换行符分割的逻辑，直接`diff`整个文本：

```js
diff () {
    this.oldTextArr = [oldText];// .split(/\n+/g);
    this.newTextArr = [newText];// .split(/\n+/g);
    // ...
}
```



![image-20210818192825679.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/154be4ae68bd4100988db659de4a4ca0~tplv-k3u1fbpfcp-watermark.image)

看起来好像可以，让我们加大文本的文字数量：

![image-20210818193054909.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fd54571d3974327ad038aede20d86b7~tplv-k3u1fbpfcp-watermark.image)

果然它凉了，显然我们之前简单的求最长公共子序列的算法是无法承受太多文字的，无论是`dp`数组所占的空间过大，还是递归算法的层数过深导致内存溢出。

对于算法渣渣的笔者来说这也搞不定，那怎么办呢，只能使用开源的力量了，当当当当，就是它：[diff-match-patch](https://github.com/google/diff-match-patch)。



# diff-match-patch库

`diff-match-patch`是一个高性能的用来操作文本的库，支持多种编程语言，除了计算两个文本的差异外，它还可以用来进行模糊匹配及打补丁，从名字也能看得出来。

使用很简单，我们先把它引进来，`import`方式引入的话需要修改一下源码文件，源码默认是把类挂到全局环境上的，我们要手动把类给导出来，然后`new`一个实例，调用`diff`方法即可：

```js
import diff_match_patch from './diff_match_patch_uncompressed';

const dmp = new diff_match_patch();

diffAll () {
    let diffList = dmp.diff_main(oldText, newText);
    console.log(diffList);
}
```

返回的结果是这样的：

![image-20210818195940486.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb645cf1c74944419800c0ce1a1e14af~tplv-k3u1fbpfcp-watermark.image)

返回的是一个数组，每一项都代表是一个差异，`0`代表没有差异，`1`代表是新增的，`-1`代表是删除，我们只要遍历这个数组把字符串拼接起来就可以了，非常简单：

```js
diffAll () {
    let diffList = dmp.diff_main(oldText, newText);
    let htmlStr = '';
    diffList.forEach((item) => {
        switch (item[0]) {
            case 0:
                htmlStr += item[1];
                break;
            case 1:
                htmlStr += `<span class="add">${item[1]}</span>`;
                break;
            case -1:
                htmlStr += `<span class="del">${item[1]}</span>`;
                break;
            default:
                break;
        }
    });
    this.showTextArr = htmlStr.split(/\n+/);
}
```

![image-20210818201307191.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5cb14d3ff7c40c3975a3b56b1cadea3~tplv-k3u1fbpfcp-watermark.image)

实测`21432`个字符`diff`耗时为`4ms`左右，还是很快的。

好了，以后编辑都可以愉快的摸鱼了~



# 总结

本文简单做了一道【求最长公共子序列】的算法题，并分析了一下它在文本`diff`上的一个实际用处，但是我们简单的算法终究无法支撑实际的项目，所以如果有相关需求可以使用文中介绍的一个开源库。

完整示例代码：[https://github.com/wanglin2/text_diff_demo](https://github.com/wanglin2/text_diff_demo)
