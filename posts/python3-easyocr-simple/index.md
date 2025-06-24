# Python3 Easyocr 简单使用识别参数


python3 easyocr 学习，测试。

<!--more-->

```python
import easyocr
import torch

gpu_is_available = torch.cuda.is_available()
reader = easyocr.Reader(['ch_sim', 'en'], gpu=gpu_is_available)


ocr_data = reader.readtext('/image/path', detail=0, text_threshold=0.7, low_text=0.1, width_ths=0.3)
```

1. detail = 0 仅输出识别到的文字
2. text_threshold=0.7 文本置信阈值，值越小，文本块内被识别为文字的可能性越高
3. low_text=0.1 值越小，图片内容被识别为文本块的可能性就更高
4. width_ths=0.3 控制横向图片内容是否会被识别为一个或多个文本块，值越小被识别成多个文本块的可能性就更高
5. 待续...

参考：
1. [官方文档](https://www.jaided.ai/easyocr/documentation/ "官方文档")
2. [Python使用EasyOCR库对行程码图片进行OCR文字识别介绍与实践](https://cloud.tencent.com/developer/article/2016127 "Python使用EasyOCR库对行程码图片进行OCR文字识别介绍与实践")


---

> 作者: hpkaiq  
> URL: https://hpk.me/posts/python3-easyocr-simple/  

