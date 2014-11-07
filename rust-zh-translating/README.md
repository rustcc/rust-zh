翻译组工作目录，OmegaT项目目录
=============================

rust-zh-translating 是内部工作目录，仅供翻译组成员使用。

目录结构：

- omegat.project：[OmegaT](http://omegat.org)项目文件
- glossary：词汇表目录
- omegat：内部目录
  - omegat/project_save.tmx：记录翻译中间结果（纯文本格式/XML）
- source：被翻译的英文原文目录（可随时更新、保持与Rust官网同步）
  - source/docs：Rust语言文档
  - source/blogs：Rust官方博客（手工生成的.md文件）
  - source/rfcs：RFC（Rust设计文档）

更新 source 目录内的英文原文，不会导致此前部分翻译完成的文档失效，所以我们可以随时或不定期的更新以保持跟官方文档同步，而不需要担心影响译文。（Liigo注：这是被OmegaT软件的辅助翻译机制保证的，参见下文。）

omegat 目录下的 project_save.tmx 是纯文本文件（XML格式），存储的是翻译中间结果，英文=>中文 逐句对照表。这是本目录内最重要的文件，没有之一。在没有或没有打开 OmegaT 软件的情况下，可手工编译此文件，但务必谨慎处理，不要搞乱XML格式结构。

TODO：介绍 OmegaT 辅助翻译软件，及其翻译流程。
