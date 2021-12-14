---
title: Java中if else替代方案
date: 2021-11-08 14:13:40
tags:
- java 
- if
- else
---

在我们平时的开发过程中，经常可能会出现大量If else的场景，代码显的很臃肿，非常不优雅，且难以维护。那我们有没有办法处理呢？

## 工厂模式+枚举
工厂模式
将操作进行抽象给出一个操作接口
```
public interface IOperation {
    int apply(int a, int b);
}
```
```
然后实现加减乘除四个方法
```
```
public class Addition implements IOperation {
    
    @Override
    public int apply(int a, int b) {
        return a + b;
    }

    
    public enum OperationEnum {
        // 唯一枚举:
        INSTANCE;

        private Addition type = new Addition();

        public Addition getType() {
            return this.type;
        }

        public void setType(Addition type) {
            this.type = type;
        }
    }
}
```

使用枚举
```
public enum OperatorEnum {
    ADD(1, "加法", Addition.OperatorEnum.INSTANCE.getType())
    
    private final int code;

    private final String remark;

    private final Operation operation;
}
```

使用
```
int code= 1
IOperation operation = OperatorEnum.valueOf(1).getOperation();
result = shelfType.handleRefundSuccess(reqDTO);

```



## 责任链模式
https://www.cnblogs.com/xrq730/p/10633761.html
https://www.cnblogs.com/java-my-life/archive/2012/05/28/2516865.html

## 命令模式
https://cloud.tencent.com/developer/article/1559667

## 规则引擎
https://cloud.tencent.com/developer/article/1559667