---
title: 利用 VITS 生成 VTuber 语音模型
category: 技术
tags: AI
---

[VITS](https://github.com/jaywalnut310/vits) 是一个端到端的语音合成算法。只需提供少量标注好的语料即可合成效果不错的语音。目前在哔哩哔哩上有大量[基于此算法的角色语音模型](https://search.bilibili.com/all?keyword=VITS)。因此我也用了三周时间从头开始制作了一个 VTuber 的[语音模型](https://www.bilibili.com/video/BV1fv4y1r7GD)。

<div style="position: relative; width: 100%; padding-top: 75%;">
<iframe src="https://player.bilibili.com/player.html?bvid=BV1fv4y1r7GD" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" style="position: absolute; top: 0; width: 100%; height: 100%;"></iframe>
</div>

语料获取
----

目标角色在B站有不少直播杂谈录像，这为算法提供了不错的语料。杂谈中只有一个人的语音且大部分时间都在说话。美中不足的是杂谈没有原生的字幕，并且人声和背景音乐混杂，这为后续的数据清洗与标注增加了一些难度。

如果希望减少标注的负担且获得大量语料，可以考虑从 Galgame 中提取角色语音。这些语音发音清晰且有自带的文字标注。

数据预处理
----

数据预处理的部分我采用了 [VTuberTalk](https://github.com/jerryuhoo/VTuberTalk) 框架。

首先用 `tools/cut_source.py` 将下载好的视频切割成每段20分钟的音频文件，并用 [demucs](https://github.com/facebookresearch/demucs) 将每段音频中的人声提取出来，之后用 `tools/audio_to_mono.py` 将人声还原为单通道。这里切割音频的目的是为了方便 demucs 运行以及后续的标注工作。在此过程中推荐全程保持音频的采样率为48000。

用 `tools/split_audio.py` 切割音频为单个句子。此脚本中利用了静音检测算法，针对说话人的停顿自动分隔每一句话。

运行 `tools/gen_text.py` 将每个句子用 PaddleSpeech 做语音识别。语音识别算法我尝试过 [Speech-to-Text](https://cloud.google.com/speech-to-text)，但中文的识别效果差强人意。也尝试过 [Whisper](https://github.com/openai/whisper)，速度比 PaddleSpeech 慢很多，但效果要好一些。由于说话人吐字不清，所有算法都会有很多错字需要后续人工修正。

除了准确率高，Whisper 算法的另一项好处是可以输入整段语音，利用算法自动标注每句话的开始和结束时间。这省去了静音检测分隔句子的步骤。相关尝试可以参考 [Whisper-VITS-Japanese](https://www.bilibili.com/video/BV19e4y167Dx) 中的研究。

语音识别的结果会保存在与 `.wav` 文件并列的 `.txt` 文件中。可以利用脚本将所有文本收集在 VITS 算法所需要的同一个标注文件中。文件的格式为：

~~~~
wavs/01.wav|第一句话的内容。
wavs/02.wav|第二句话的内容。
~~~~

另外 VITS 算法需要单通道22050采样率的音频文件。因此还需要用 FFmpeg 转换音频格式为最终训练所用：`ffmpeg -i <source> -ar 22050 <destination>`。

数据清洗与标注
----

为了提高文本标注的准确率，我在自动识别的基础上对每段音频进行了人工修正与标注。标注的难度根据语音的清晰程度，识别算法的准确率，以及对发言上下文的熟悉程度而有所差异。我此次标注的个人经验是1.5~2小时可以精确标注200+句子。

PaddleSpeech 输出的结果是不带标点符号的，为了让最终的模型在标点处停顿，需要补齐标点。

我所训练的是纯中文语音模型，因此对于语音中出现的整句外语、难以抄录的语气词、歌声可以用 [Audacity](https://www.audacityteam.org/) 做切割或整句删除处理。注意处理完成后仍需保持单通道22050采样率。对于偶尔出现的外语词汇，我替换为了汉字拟声词，例如<ruby>挼破<rt>ruá pò</rt></ruby>。

当时我没有注意到的一点是，单段语音不能超过10秒，否则 VITS 算法会将其丢弃。建议对超时长句子进一步切割处理。

拥有标注好的400~500个句子后就可以训练出可以说话的模型了。但想要更好的效果还需要增加数据量。文章开头的演示视频用了大概900句的精标语料。

模型训练
----

我使用了 [CjangCjeng](https://space.bilibili.com/35285881) 改进版的 VITS [代码](https://github.com/CjangCjengh/vits)做训练。

先将标注好的数据集分隔为训练集和测试集。测试集不需要很多，5~10条足够使用。

修改 `text/symbols.py`，将 `chinese_cleaners` 一节取消注释，并将其余无关的 cleaner 注释掉。也可将 `text/cleaners.py` 中用不到的函数与其对应依赖删除掉。

运行 `preprocess.py --text_cleaners chinese_cleaners --filelists <label files>` 将 `.txt` 的标注文件转换为处理后的 `.txt.cleaned` 文件。这一步会将标签中的中文转换为对应的注音符号。

接下来在 `configs/chinese_base.json` 的基础上修改配置文件。这里需要修改的参数有以下几个：
* `train`
  * `log_interval`: 每隔多少步骤在 TensorBoard 中记录一次 loss 等信息。
  * `eval_interval`: 每隔多少步骤保存一次模型并评测测试集。
  * 上述两个参数可以最初调整为5或10，以便快速确认训练代码可以正确跑起来。正式训练时再根据需要改回去。
  * `epochs`: 总训练 epoch 数量。如果数据不多可以酌情缩减。
  * `batch_size`: 批量训练数据量。调整大小直至 GPU 内存接近占满。
* `data`
  * `training_files`: 训练数据集的 `.txt.cleaned` 文件路径。
  * `validation_files`: 测试数据集的 `.txt.cleaned` 文件路径。
  * `n_speakers`: 单人语音模型可以调整为0。
* `model`
  * `gin_channels`: 单人语音模型可去除此参数。
* `speakers`: 根据说话人调整。

另外 `train.py` 为了节省磁盘空间，会自动删除古老的模型文件。如果不愁磁盘占用，可以删除 `os.remove()` 相关逻辑。

之后就可以运行 `train.py -c <config file> -m <model name>` 开始训练了。训练中途可以挂一个 TensorBoard 查看训练进度。最开始，模型会输出纯杂音，之后变为说话人腔调的嘟囔，最后会是说话人腔调的语音。

此外还可以采用预训练的方法提升模型质量。例如利用[标贝中文标准女声数据集](https://weixinxcxdb.oss-cn-beijing.aliyuncs.com/gwYinPinKu/BZNSYP.rar)训练到模型收敛，再替换训练数据为自己的数据集进一步训练。

模型使用
----

VITS 代码中没有直接提供推理的脚本。但可以参考 `inference.ipynb` 中的内容自己写一个语音生成脚本。或者用 [MoeGoe](https://github.com/CjangCjengh/MoeGoe) 生成。

推荐阅读
----

网络上有不少有关 VITS 的经验分享，下面列举了一些供参考。另外也可以加入 CjangCjengh 的[技术讨论群](https://space.bilibili.com/35285881/dynamic)探讨。

* [VITS中文AI避坑指南](https://www.bilibili.com/read/cv21153903)
* [夏夜有轻风的B站专栏](https://space.bilibili.com/87688497/article)
* [IceKyrin的B站专栏](https://space.bilibili.com/85599969/article)
* [vits单人模型训练](https://colab.research.google.com/drive/1X3Ak70tia6ukmWkc7Vpb_xXun9VtaijV?usp=sharing)
