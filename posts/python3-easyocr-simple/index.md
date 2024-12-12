# Python3 Easyocr 简单使用识别参数


python3 easyocr 学习，测试。

&lt;!--more--&gt;

```python
import easyocr
import torch

gpu_is_available = torch.cuda.is_available()
reader = easyocr.Reader([&#39;ch_sim&#39;, &#39;en&#39;], gpu=gpu_is_available)


ocr_data = reader.readtext(&#39;/image/path&#39;, detail=0, text_threshold=0.7, low_text=0.1, width_ths=0.3)
```

1. detail = 0 仅输出识别到的文字
2. text_threshold=0.7 文本置信阈值，值越小，文本块内被识别为文字的可能性越高
3. low_text=0.1 值越小，图片内容被识别为文本块的可能性就更高
4. width_ths=0.3 控制横向图片内容是否会被识别为一个或多个文本块，值越小被识别成多个文本块的可能性就更高
5. 待续...

参考：
1. [官方文档](https://www.jaided.ai/easyocr/documentation/ &#34;官方文档&#34;)
2. [Python使用EasyOCR库对行程码图片进行OCR文字识别介绍与实践](https://cloud.tencent.com/developer/article/2016127 &#34;Python使用EasyOCR库对行程码图片进行OCR文字识别介绍与实践&#34;)


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/python3-easyocr-simple/  

