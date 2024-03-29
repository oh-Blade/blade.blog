Elasticsearch 使用一种称为 倒排索引 的结构，它适用于快速的全文搜索。一个倒排索引由文档中所有不重复词的列表构成，对于其中每个词，有一个包含它的文档列表。

例如，假设我们有两个文档，每个文档的 content 域包含如下内容：

The quick brown fox jumped over the lazy dog
Quick brown foxes leap over lazy dogs in summer
为了创建倒排索引，我们首先将每个文档的 content 域拆分成单独的 词（我们称它为 词条 或 tokens ），创建一个包含所有不重复词条的排序列表，然后列出每个词条出现在哪个文档。结果如下所示：
![image.png](https://www.hounk.world/upload/2021/08/image-b5487b17b14c45f793006ad2171014f5.png)

现在，如果我们想搜索 quick brown ，我们只需要查找包含每个词条的文档：

![image.png](https://www.hounk.world/upload/2021/08/image-ef854d66419c4641af1ced5b1d9953ea.png)

两个文档都匹配，但是第一个文档比第二个匹配度更高。如果我们使用仅计算匹配词条数量的简单 相似性算法 ，那么，我们可以说，对于我们查询的相关性来讲，第一个文档比第二个文档更佳。

但是，我们目前的倒排索引有一些问题：

* Quick 和 quick 以独立的词条出现，然而用户可能认为它们是相同的词。
* fox 和 foxes 非常相似, 就像 dog 和 dogs ；他们有相同的词根。
* jumped 和 leap, 尽管没有相同的词根，但他们的意思很相近。他们是同义词。

使用前面的索引搜索 +Quick +fox 不会得到任何匹配文档。（记住，+ 前缀表明这个词必须存在。）只有同时出现 Quick 和 fox 的文档才满足这个查询条件，但是第一个文档包含 quick fox ，第二个文档包含 Quick foxes 。

我们的用户可以合理的期望两个文档与查询匹配。我们可以做的更好。

如果我们将词条规范为标准模式，那么我们可以找到与用户搜索的词条不完全一致，但具有足够相关性的文档。例如：

Quick 可以小写化为 quick 。
foxes 可以 词干提取 --变为词根的格式-- 为 fox 。类似的， dogs 可以为提取为 dog 。
jumped 和 leap 是同义词，可以索引为相同的单词 jump 。
现在索引看上去像这样：

![image.png](https://www.hounk.world/upload/2021/08/image-cb1d593259e045c59f22db11a42a53b6.png)
这还远远不够。我们搜索 +Quick +fox 仍然 会失败，因为在我们的索引中，已经没有 Quick 了。但是，如果我们对搜索的字符串使用与 content 域相同的标准化规则，会变成查询 +quick +fox ，这样两个文档都会匹配！

[倒排索引](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inverted-index.html#inverted-index)
