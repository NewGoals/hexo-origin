---
title: Google_Protocol_Buffer基本使用
commends: false
date: 2020-01-14 17:01:49
categories: caffe
tags: protocol
---

<center>Protocol是Google公司内部的混合语言数据标准。</center>
<center>Protocol Buffers是一种轻便高效的结构化数据存储格式，可以用于结构化数据穿串行化，或者说序列化。它很适合做数据存储或RPC数据交换格式。可用于通讯协议，数据存储等领域的语言无关，平台无关，可扩展的序列化结构数据格式。</center>
<center>目前提供了C++，Java，Python三种语言的API。</center>

<!--more-->

------

> 我们主要来探究protocol关于C++部分的API，了解这方面知识可能对理解caffe的源码有很大的帮助，因为protocol在caffe中的使用实在是普遍，最常见的namespace caffe即是protoc编译生成的。在这篇文章中我不会做详细的说明，因为在下面的教程和文档中已经说的非常明白了，请自行了解。
参考资料：  
https://www.ibm.com/developerworks/cn/linux/l-cn-gpb/
https://developers.google.com/protocol-buffers/docs/cpptutorial


## 举个栗子
开发一个简单的helloworld程序，该程序分为两部分，一部分为Writer，第二部分为Reader。Writer负责将一些结构化的数据写入一个磁盘文件，Reader则负责从该磁盘问价中读取结构化数据并打印到屏幕上，实现磁盘的读写操作。  
### 定义你的Protocol格式
创建一个helloworld.proto文件，并编辑
``` protobuf
package lm;
message helloworld{
    required int32 id = 1;
    required string str = 2;
    optional int32 opt = 3;
}
```
### 编译.proto文件
写好了proto文件是不能直接拿来使用的，需要protoc编译生成头文件和cc文件。由于我安装caffe时已经顺便安装了protocol，如果你没有安装，可以去网上自行搜索安装方式。
```
protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/helloworld.proto
```
***-I***: 表示编译源文件的目录  
***--cpp_out***: 输入C++可用的文件  
这里我们不写这么麻烦，直接将其生成在当前目录下即可。
``` 
 protoc helloworld.proto --cpp_out=./
```
现在目录下就有了三个文件，一个为自己写的helloworld.proto，另外两个为protoc编译生成的两个文件，.proto文件改动时，需要重新编译生成文件才会更新。
```
helloworld.pb.cc  helloworld.pb.h  helloworld.proto
```
### 编写writer和reader
protoc生成的文件只是提供给我们方法而已，真正的读写操作还是要自己来实现，至于究竟给我们提供那些操作，可以自行去头文件中查询。

创建writer.cpp，并编辑
``` cpp
#include "helloworld.pb.h"
#include <fstream>
#include <iostream>
using lm::helloworld;
using std::fstream;
using std::ios;

int main(){
    helloworld msg1;
    msg1.set_id(101);
    msg1.set_str("hello");

    // 创建一个fstream流
    fstream output("./hello", ios::out | ios::trunc | ios::binary);

    // 将msg1序列化后写入fsteam流
    if(!msg1.SerializeToOstream(&output)){
        std::cerr << "Failed to write msg. "<<std::endl;
        return -1;
    }
    return 0;
}
```
创建reader.cpp，并编辑
``` cpp
#include "helloworld.pb.h"
#include <iostream>
#include <fstream>
using lm::helloworld;
using std::fstream;
using std::ios;

void ListMsg(const helloworld &msg){
    std::cout<<msg.id()<<std::endl;
    std::cout<<msg.str()<<std::endl;
}

int main(){
    helloworld msg1;

    // 创建一个输入流
    fstream input("./hello", ios::in | ios::binary);

    // 解析输入流
    if(!msg1.ParseFromIstream(&input)){
        std::cerr<<"Failed to parse hello"<<std::endl;
        return -1;
    }

    ListMsg(msg1);
    return 0;
```

### 编译writer.cpp与reader.cpp
写一个简单的Makefile文件
``` makefile
all: writer reader
.PHONY: all

writer:writer.o helloworld.pb.o
	g++ -o writer writer.o helloworld.pb.o -lprotobuf
reader:reader.o helloworld.pb.o
	g++ -o reader reader.o helloworld.pb.o -lprotobuf

helloworld.pb.o:helloworld.pb.cc
	g++ -c helloworld.pb.cc
writer.o:writer.cpp helloworld.pb.h
	g++ -c writer.cpp
reader.o:reader.cpp helloworld.pb.h
	g++ -c reader.cpp

clean:
	rm writer reader *.o
```
使用`make all`执行一下Makefile，可以看到生成了两个可执行文件writer和reader。

./writer运行后产生一个hello文件，该文件为储存磁盘上的二进制文件，再执行./reader输出从hello中读取的值。
```
./reader
>101
>hello
```

## 官方指南的栗子
### Define Your Protocol Format
> [Protocol Buffer Basics:C++](https://developers.google.com/protocol-buffers/docs/cpptutorial)有详细的说明

创建一个联系人手册的应用，首先应创建一个.proto的文件，为我们想要序列化的数据结构增加message，为message的每一个成员声明一个名字和类型。我们创建一个`addressbook.proto`的文件。创建一个联系人**Person** message，当中应该包含联系人的信息**name**, **id**还有可选项**email**，写一个枚举类型，将**PhoneType**限制为**MOBILE**, **HOME**和**WORK**，创建一个嵌套的**PhoneNumber** message，允许一个联系人有多种联系方式，用number记录电话号码，**PhoneType**指定电话类型，最后将定义**phone**表示联系方式的数目，将其设为repeated表示可以设置多个。最后设置一个主角**AddressBook** message，让其可以设置多个联系人。
``` protobuf
syntax = "proto2";

package tutorial;

message Person{
    required string name = 1;       //name
    required int32 id = 2;          //no.
    optional string email = 3;      //email

    enum PhoneType{
        MOBILE = 0;
        HOME = 1;
        WORK = 2;
    }
    message PhoneNumber{
        required string number = 1;
        optional PhoneType type = 2 [default = HOME];
    }

    repeated PhoneNumber phones = 4; //number of phone
}

message AdderssBook{
    repeated Person people = 1;     //number of person
}
```
### Compiling Your Protocol Buffers
和上面的简单栗子一样，先编译生成C++可以使用的头文件和.cc文件，同样将其放到当前目录下。
```
protoc addressbook.proto --cpp_out=./
```

### The Protocol Buffer API
如果查看生成的`addressbook.pb.h`文件的话，你可以看到其为每一个你定义的message都生成了一个类，比如查看看Person类，每一个成员都有一些存取的操作方法。
``` cpp
  // name
  inline bool has_name() const;
  inline void clear_name();
  inline const ::std::string& name() const;
  inline void set_name(const ::std::string& value);
  inline void set_name(const char* value);
  inline ::std::string* mutable_name();

  // id
  inline bool has_id() const;
  inline void clear_id();
  inline int32_t id() const;
  inline void set_id(int32_t value);

  // email
  inline bool has_email() const;
  inline void clear_email();
  inline const ::std::string& email() const;
  inline void set_email(const ::std::string& value);
  inline void set_email(const char* value);
  inline ::std::string* mutable_email();

  // phones
  inline int phones_size() const;
  inline void clear_phones();
  inline const ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >& phones() const;
  inline ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >* mutable_phones();
  inline const ::tutorial::Person_PhoneNumber& phones(int index) const;
  inline ::tutorial::Person_PhoneNumber* mutable_phones(int index);
  inline ::tutorial::Person_PhoneNumber* add_phones();
```
至于Writing和Reading部分，感兴趣的可以去官方文档了解，这里不详细赘述了(实在是不想写了，官网上都有...)