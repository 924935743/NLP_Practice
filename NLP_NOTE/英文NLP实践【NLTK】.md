# 英文NLP实践【NLTK】

在**海外舆情监控**项目的契机下，通过不断的遇到问题，再解决问题的过程中，发现了一些好玩的文本处理方法和一些实用的工具。所以写在这里当作分享的同时，也当作是自己的一个备忘录。 由于游戏聊天是一个多语种的环境，针对不同语言的处理逻辑和技巧也不尽相同。今天先总结的是**英文**，后续会陆陆续续更新**中文、日文、韩语**等其他语种。选择的分析工具是`NLTK`（ps：`NLTK`不支持中文），下文中展示的代码例子也是通过`NLTK`来实践的。

- [x] 🧐 **文本挖掘的思路**
- [x] 🛠**文本预处理**；
- [x] 🚮**语种判别**；
- [x] 🔪 **分词**；
- [x] 🌟**词性标注**；
- [x] 🔷 **分块切词**；
- [x] 👶**词形还原**  & **词干提取**
- [x] ⛳️  **结果过滤**

## 文本挖掘的思路

在分析之前，应当有意识的维护一套自己的**用户词典**，可以让运营同事提供游戏专业名词，这些词也往往是运营同事比较关注的重点词汇，这些词汇在市面上的分词工具中是比较难准确切割的。同时，寻找合适的*stopwords***停用词词典**，对后续的分词结果再做一层过滤。字典最好有一个**标准化**的格式，后续后新的用途可以灵活的使用。目前我的字典按照下面这个格式进行整理：

![](./md_png/Picture1.png)

```mermaid
graph TD;
 文本-->文本预处理;
 文本预处理-->语种判别; 
语种判别-->分词-词性标注;
 分词-词性标注-->分块切词 & 词形还原 & 词干提取;
分块切词 & 词形还原 & 词干提取-->结果过滤;
```

## 文本预处理

去除表情和多余的空格，大小写转化等一系列常规操作。

```python
import emoji
import re

def emoji_remove(text):
    text = emoji.demojize(text)
    res = re.sub(':\S+?:', ' ', text)
    return res

def text_clean(text):
    text = re.sub(r"[ ]+", " ", text)
    text = self.emoji_remove(text)
    text = text.strip().lower()
    return text
```

## 语种判别

使用`langid`工具进行语种判别，`langid`支持56种语言识别，和概率输出，整体识别效果还不错，也比较稳定。但是需要注意的是标点符号（半角，全角）会影响判别，所以在判别的时候可以先将标点符号替换成空格。

```python
from langid.langid import LanguageIdentifier, model
identifier = LanguageIdentifier.from_modelstring(model,norm_probs=True)
identifier.classify('我爱你中国')
```

> ('zh', 0.9999998421995415)

## 分句 & 分词

`nltk`默认用`PunktSentenceTokenizer`分句，用基于宾州树库分词规范的`TreebankWordTokenizer`分词。简单对比`nltk.word_tokenize`和`nltk.wordpunct_tokenize`的区别：

* word_tokenize（默认）：
  
  * 缩写(contraction)]()：Isn’t会分成Is和n’t
  * 符号粘连：19+，hello～这种伴有符号的词会默认将他们划分成一个整体
  
* wordpunct_tokenize：
  
  * 缩写(contraction)：Isn’t会分成Isn和t
  * 符号粘连： 19+，hello～这种伴有符号的词会划分成19,+,hello,~


```python
import nltk
print(nltk.word_tokenize("isn't"))
print(nltk.wordpunct_tokenize("isn't"))
print(nltk.word_tokenize('19~'))
print(nltk.wordpunct_tokenize('19~'))
```

> ['is', "n't"]
>
> ['isn', "'", 't']
>
> ['19~']
>
> ['19', '~']

总的来说，`wordpunct_tokenize`的颗粒度更细，但是也容易分错词。	`word_tokenize`除了符号粘连处理不好之外，效果更好。同时`NLTK`工具支持选择指定分词器，可以通过`dir(nltk.tokenize)` 进行选择。

## 词性标注

`nltk.pos_tag`支持对分好的token进行词性标注。词性大类如下图总结，重点关注名词**NN***，形容词**JJ***，动词**V***，副词**RB***。

![](./md_png/Picture2.png)

## 分块切词

基于上述的分词和词性标注，结合句法分析编写**语法表达式**，进行`chunk`划分。`nltk.regexpParser`解析语法表达式，并以`语法树`进行分析展示。下面的例子使用了四个英文的语法结构：

* Adj-N：形容词+名词
* Adv-V：副词+名词
* V-N：动词+（冠词）+名词
* Noun：名词+（介词|所有格）+名词


代码中详细举例，大家也可自行编写自己关注的语法结构表达式。

```python
from nltk import RegexpParser
from nltk import word_tokenize, pos_tag

def chunk_split(tokens): # --> [(word,tag),....]
    # chunk + fitter
    # POS tag a sentence
    res = []
    grammar = """
    Adj-N: {<JJ.*>+<NN.*>+}
    Adv-V: {<RB.*>+<V.*>+}
    V-N: {<V.*>+<DT>?<NN.*>+} # want some water/want apple --> verb + determiner + noun/verb + noun
    Noun: {<NN.*>+<IN|POS>?<NN.*>+} # --> apple of eva/office lady --> noun + in + noun/noun + noun
    """
parser = RegexpParser(grammar)
chunks = parser.parse(tokens)
```

```python
str0 = "Hello World! Isn't it good to see you? Thanks for buying this book."
sent_cut = nltk.pos_seg(nltk.word_tokenize(str0))
chunk_res = chunk_split(sent_cut)
print(chunk_res)
```

> [('hello world', 'cu'), ('buy this book', 'cu')]

![](./md_png/Picture3.png)

可以看到`hello world`和'buy this book`短语有划分出来。

## 词形还原 & 词干提取

英文在不同的语法环境中，词有各种各样的变形：单复数，比较级，过去分词，动名词等等。为了精确统计高频词，我们需要将词进行词形还原或者是词干提取。

a.     词形还原`nltk.stem. WordNetLemmatizer`：可以根据指定的词性进行还原

```python
from nltk.stem import WordNetLemmatizer
lemmatizer = WordNetLemmatizer()

print(lemmatizer.lemmatize('cooking', pos='v'))
print(lemmatizer.lemmatize('cookbooks',pos=’n’))
```

> cook
>
> cookbook

b.     词干提取`nltk.stem.PorterStemmer`:

```
from nltk.stem import PorterStemmer
stemmer = PorterStemmer()

print(stemmer.stem('working'))
print(stemmer.stem('worked'))
```

> work
> 
> work

## 结果过滤

在最后输出分析结果之前，再结合以下方式过滤结果：

a.     **词性过滤**：过滤一些不关注的词性

b.    **单词过滤**：长度为1的词

c.     **停用词过滤**

## 附件

![](./md_png/Picture4.png)

## 参考

[自然语言处理工具包之NLTK](https://www.biaodianfu.com/nltk.html)

[ Natural Language Toolkit](https://www.nltk.org)

最新更新于 2021.12.29
