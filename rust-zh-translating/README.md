翻译组工作目录，OmegaT项目目录
=============================

rust-zh-translating 是内部工作目录，仅供翻译组成员使用。

# 目录结构：

- omegat.project：[OmegaT](http://omegat.org)项目文件
- glossary：词汇表目录
- omegat：内部目录
  - omegat/**project_save.tmx**：记录翻译中间结果（纯文本格式/XML）
- **source**：被翻译的英文原文目录（可随时更新、保持与Rust官网同步）
  - source/docs：Rust语言文档
  - source/blogs：Rust官方博客（手工生成的.md文件）
  - source/rfcs：RFC（Rust设计文档）

更新 source 目录内的英文原文，不会导致此前部分翻译完成的文档失效，所以我们可以随时或不定期的更新以保持跟官方文档同步，而不需要担心影响译文。（Liigo注：这是被OmegaT软件的辅助翻译机制保证的，参见下文。）

omegat 目录下的 project_save.tmx 是纯文本文件（XML格式），存储的是翻译中间结果，英文=>中文 逐句对照表。这是本目录内最重要的文件，没有之一。在没有或没有打开 OmegaT 软件的情况下，可手工编译此文件，但务必谨慎处理，不要搞乱XML格式结构。

# 计算机辅助翻译软件OmegaT

OmegaT是辅助翻译领域做的相当好的免费开源软件。看下面的知乎链接，国内用户胡杨实际使用过OmegaT，并给出了高度评价。（注：有比OmegaT更好的收费软件，但不适合开源社区使用。）

译者通过OmegaT逐句翻译文档（软件自动拆分句子）。**当原文档更新之后，未改变的句子不需要再次翻译，部分改变的句子可以参考之前的翻译（通过模糊匹配找到）**。此外，OmegaT还支持多人协作翻译（通过Git/Svn），支持术语表（Glossary）。对于Rust这类文档变化很快的情况，对于Rust中文圈社区协同翻译，对于术语较多的系统编程领域，恰好适合OmegaT发挥自身优势。这是我们选用OmegaT的理由。

- [OmegaT官网](http://www.omegat.org/) 和 [维基百科条目](http://zh.wikipedia.org/wiki/OmegaT)
- [OmegaT这款开源的计算机辅助翻译软件（CAT）如何？](http://www.zhihu.com/question/22738692)，知乎提问，译者胡杨现身说法；
- [OmegaT，一款免费又给力的翻译集成工具](http://knewalker.com/?p=666)，作者Corinne McKay，[英文原文](http://thoughtsontranslation.com/2008/04/11/omegat-a-free-and-very-useful-tent/)；
- [OmegaT - 开源跨平台的电脑辅助翻译工具软件入门与下载](http://www.iplaysoft.com/omegat.html)，来自异次元网站；

TODO：介绍使用OmegaT辅助翻译的操作流程。
