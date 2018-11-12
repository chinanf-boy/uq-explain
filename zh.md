# uq

`uq`是一个简单,用户友好的,`sort | uniq`替代品.

无论顺序如何,它都会从输出中删除重复的行。不像`sort | uniq`，`uq`不会排序.这允许`uq`也可以在连续串～流上运行.

```bash
$ python -c "while 1: print('a');print('b')" | uq
a
b
```
