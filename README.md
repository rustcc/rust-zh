Rust编程语言官方文档中文化
=========================

本项目由 [Rust中文社区](http://www.rust.cc) 发起和运作，致力于把 [Rust](http://www.rust-lang.org) 编程语言官方文档全部翻译为中文。

我们热忱欢迎所有Rust爱好者加入我们的行列，为Rust在中文社区的推广尽一份力量。Rust中文社区官方QQ群：`144605258`。

本项目拟在 `Creative Commons — 署名-相同方式共享 4.0` （`CC BY-SA 4.0`） 协议下开源。（征求所有成员同意中...）

本项目的主要目录结构介绍：

- docs：Rust官方文档和教程中文版（优先翻译部分，原文来自 https://github.com/rust-lang/rust/tree/master/src/doc ）
- rfcs：Rust官方设计文档中文版（RFC：征求修正意见书，原文来自 https://github.com/rust-lang/rfcs/tree/master/text ）
- blogs：Rust官方博客中文版（原文来自 http://blog.rust-lang.org ）
- other：Rust相关的第三方文章和博客中文版（必要时自建子目录）
- rust-zh-translating：翻译组内部工作目录，OmegaT项目目录（初步试用）

其中前四个目录（docs, rfcs, blogs, other）是文档发布目录，最后一个 rust-zh-translating 目录是翻译组内部工作目录。
发布在 other 目录的内文章，请在前面注明原文作者、写作日期、原始链接，当然还有译者信息。

在前期翻译中，可以直接提交翻译后的中文版到 docs, rfcs, blogs, other 这四个目录。
但是后期我（Liigo）希望除 other 目录以外的其他所有翻译工作最终转换到 OmegaT 辅助翻译软件中进行（前期转换工作由我或/和其他志愿者完成）。
目的是尽可能使我们平滑地跟进官方最新文档，不做无谓的重复翻译和反复检查工作。

愿意加入和我们一起翻译的朋友，请先Fork，并充分利用本项目的[Issues](https://github.com/rustcc/rust-zh/issues)，选定一个别人还没有（开始）翻译的文章或段落，创建一个新的Issue，翻译后提交Pull Request，被合并后Close该Issue。请在Issue标题和说明中写清楚要翻译的内容，并在Issue评论中随时公布进度，跟其他网友共同讨论。也可在QQ群中咨询别人的翻译范围，避免重复劳动。

（本项目欲采用的翻译流程和辅助工具还没有最终确定，需所有成员共同协商一致后确定。重点需要考虑到Rust官方文档目前仍处于频繁的更新中，我们要想办法随时并轻松地跟进到官方最新版，否则之前翻译的中文版很快就陈旧了。）
