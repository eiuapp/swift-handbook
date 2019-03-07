

openstack swift benchmarking 工具 getput 使用

## env

如前，不再重复。

## step

根据 https://docs.openstack.org/swift/queens/associated_projects.html 我们知道有3个 benchmarking 工具

- getput - getput tool suite
- COSbench - COSbench tool suite
- ssbench - ssbench tool suite

其中后面 COSbench ， ssbench 暂时都没有很好的支持 keystone v3, 并经过一定的尝试后，放弃了。

下面我们只讲 [getput](https://github.com/markseger/getput)

### 文档

https://github.com/markseger/getput/blob/master/getting-started.txt   这里面有介绍

![getput-command-1](https://res.cloudinary.com/dmtixvmgt/image/upload/v1550741954/getput-command-1_kizm2m.png)

另一个 Introduction.pdf 中也有类似的

![getput-command-2](https://res.cloudinary.com/dmtixvmgt/image/upload/v1550741954/getput-command-2_dpdcig.png)

看起来，能展示的东西，可以满足我们的需求

### 安装


箭头位置报错了。

![getput-install-1](https://res.cloudinary.com/dmtixvmgt/image/upload/v1550741954/getput-install-1_spvqba.png)

看了下代码，加一个括号呀。

![getput-install-2](https://res.cloudinary.com/dmtixvmgt/image/upload/v1550741955/getput-install-2_tkvjre.png)

```
git clone https://github.com/markseger/getput.git && cd getput/
vim setup.py 
pip install python-swiftclient
pip3 install python-swiftclient
python setup.py install
```

命令示例如下

```
getput
getput -cc -oo -n1 -s1k -tp,d
getput -cc -oo -n1 -s10k -tp,g,d
getput -cc -oo -n1 -s10k -tp,g,d --procs 2
getput -cc -oo -n1 -s100k -tp,g,d --procs 2
getput -cc -oo -n1 -s1000k -tp,g,d --procs 2
getput -cc -oo -n1 -s100000k -tp,g,d --procs 2
getput -cc -oo -n1 -s100000k -tp,g,d --procs 1
getput -cc -oo -n1 -s10000k -tp,g,d --procs 1
getput -cc -oo -n1 -s10000k -tp,g,d --procs 2
getput -cc -oo -n1 -s10000k -tp,g,d --procs 3
getput -cc -oo -n1 -s1m -tp,g,d --procs 3
```

参数具体含义，直接看文档吧，也很简单。

## ref
- https://docs.openstack.org/swift/queens/associated_projects.html
- https://github.com/markseger/getput
