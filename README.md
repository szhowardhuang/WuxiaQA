# WuxiaQA
一个中文武侠小说的问答助手

#### OpenXLab的部署经验:
1)首先既然要部署,代码的逻辑问题都已经不大了,代码和部署有关的部分就要修改。

2)和部署有关的代码具体有:模型下载 ,模型载入 ,数据库生成 , 数据库下载 ,库依赖包

3)模型问题:

由于模型文件从huggingface下载不易,所以改成从OpenXLab模型库下载。

   download的output path,就是模型的path,download不会再添加目录
   
   ```download(model_repo='OpenLMLab/InternLM-chat-7b',output='/home/xlab-app-center/InternLM-chat-7b')```
 
   模型转载path和download output path要保持一致。
   
   ```llm = InternLM_LLM(model_path = "/home/xlab-app-center/InternLM-chat-7b")```
	
4)数据库问题:

由于openXLab 暂时不支持git lfs clone,所以大文件无法自动从github下载。
	针对现状,只能打patch,把大文件切碎,让app跑起来时组装大文件。 具体见unzip_db函数
	
5)库依赖问题:

openXLab是根据github代码库里面的requirements.txt(python依赖) 和 packages.txt(linux库依赖)来构建image,
	比较麻烦的是我们看不到openXLab使用的docker image配置文件,所以只能一点点试着调整库依赖文件。
	其中chromadb有坑,原因是chromadb需要GLIBC 2.29以上,后来提交issue给OpenXLab升级GLIBC解决。 
	接着碰到sqlite3 要大于 3.35.0的问题,正解是让OpenXLab升级sqlite3,补丁解法是修改代码如下:
	
	import pysqlite3
	import sys
	sys.modules["sqlite3"] = sys.modules.pop("pysqlite3")
	import chromadb

同时依赖包添加 pysqlite3-binary



#### 对此次RAG的体会
RAG的数据质量很重要,直接把一份小说向量化存储到数据库里面,然后根据问题检索相应的词条给LLM组织语言,给出答案,这条路基本不可行。

我大体上认为小说的结构复杂,比如:人物之间的关系需要较长的上下文来判断,而RAG这种方式不能送很长的文本给LLM。

RAG的点就是查找TOP K 信息送给LLM。 所以RAG的数据内容需要段落清晰,在几K的文本里面讲清楚某件事情,
其实有点像百科全书,互相之间的内容关联度不高。个人拙见,欢迎拍砖。

将数据库改成百科全书，加上引导词，有所改善。如下图：

![image](https://github.com/szhowardhuang/WuxiaQA/assets/1407300/dc510936-53ce-42e1-82be-b16bc62336f5)

后续打算用结构化数据放进传统数据库， 不把非结构化数据向量化到向量数据库。 然后检索关键字，把检索信息prompt化后送给大模型。
