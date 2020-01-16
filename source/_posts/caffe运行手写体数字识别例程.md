---
title: caffe运行手写体数字识别例程
commends: false
date: 2020-01-09 18:04:43
categories: caffe
tags: caffe
---

<center>如果说还是不确定自己的caffe安装是否成功，完成一个简单的小项目来进行测试。我们使用MNIST数据集来作为训练和测试样本，使用LeNet-5作为训练模型</center>

<!--more-->

## 下载MNIST数据集
假设我们处在根目录(之后的文件夹操作我们都假定自己处于根目录下)，进入data/mnist，可以看到一个get_mnist.sh的脚本
``` bash
#!/usr/bin/env sh
# This scripts downloads the mnist data and unzips it.

DIR="$( cd "$(dirname "$0")" ; pwd -P )"
cd "$DIR"

echo "Downloading..."

for fname in train-images-idx3-ubyte train-labels-idx1-ubyte t10k-images-idx3-ubyte t10k-labels-idx1-ubyte
do
    if [ ! -e $fname ]; then
        wget --no-check-certificate http://yann.lecun.com/exdb/mnist/${fname}.gz
        gunzip ${fname}.gz
    fi
done
```
可见该脚本实现从下载地址下载mnist数据集，`bash get_mnist.sh`执行该脚本，会将该数据集下载到当前目录下，该数据集的格式可以在相关的网站查询得到。

**TRAINING SET LABEL FILE (train-labels-idx1-ubyte):**


| [offset] | [type]     |     [value]    |      [description] |
| ------ | -------- | ----- | ----- |
|0000 |    32 bit integer | 0x00000801(2049) |magic number (MSB first)
|0004 |    32 bit integer | 60000            |number of items
|0008 |    unsigned byte  |??               |label
|0009 |    unsigned byte  | ??              | label
|........|
|xxxx|     unsigned byte|   ??|               label


**TRAINING SET IMAGE FILE (train-images-idx3-ubyte):**


|[offset]| [type]|          [value]|          [description]
| ---- | ---- | ---- | ---- |
|0000|     32 bit integer|  0x00000803(2051)| magic number
|0004|     32 bit integer|  60000|            number of images
0008|     32 bit integer|  28|               number of rows
0012|    32 bit integer|  28|               number of columns
0016|     unsigned byte|   ??|               pixel
0017|     unsigned byte|   ??|               pixel
........|
xxxx|     unsigned byte|   ??|               pixel


测试集格式同样类似

## 转换格式为LMDB
caffe对mnist样例写好了转换格式的代码，将原二进制格式转化为LMDB，在caffe根目录下执行`./examples/mnist/create_mnist.sh`，会发现在examples/mnist/mnist_train_lmdb和examples/mnist/mnist_test_lmdb两个目录，每个目录下都有两个文件：data.lmdb和lock.lmdb，由名称就可以知道一个为LMDB格式的训练集，一个为LMDB格式的测试集。

可以分析以下脚本了解是如何转换的。用vim打开create_mnist.sh
``` bash
#!/usr/bin/env sh
# This script converts the mnist data into lmdb/leveldb format,
# depending on the value assigned to $BACKEND.
set -e

EXAMPLE=examples/mnist
DATA=data/mnist
BUILD=build/examples/mnist

BACKEND="lmdb"

echo "Creating ${BACKEND}..."

# 将原本examples/mnist中有关train和test的lmdb或leveldb格式的文件删除，重新转换
rm -rf $EXAMPLE/mnist_train_${BACKEND}
rm -rf $EXAMPLE/mnist_test_${BACKEND}

# 使用已经写好的转换工具进行类型转换
$BUILD/convert_mnist_data.bin $DATA/train-images-idx3-ubyte \
  $DATA/train-labels-idx1-ubyte $EXAMPLE/mnist_train_${BACKEND} --backend=${BACKEND}
$BUILD/convert_mnist_data.bin $DATA/t10k-images-idx3-ubyte \
  $DATA/t10k-labels-idx1-ubyte $EXAMPLE/mnist_test_${BACKEND} --backend=${BACKEND}

echo "Done."
```
由上面的脚本可知调用了build/examples/mnist下的convert_mnist_data.bin的编译成功的二进制工具，之前我们已经知道了make编译后的文件都存放在build目录下，所以我们根据目录的提示可以知道源码应在examples/mnist中，其源码应为convert_mnist_data.cpp
``` Cpp
// This script converts the MNIST dataset to a lmdb (default) or
// leveldb (--backend=leveldb) format used by caffe to load data.
// Usage:
//    convert_mnist_data [FLAGS] input_image_file input_label_file
//                        output_db_file
// The MNIST dataset could be downloaded at
//    http://yann.lecun.com/exdb/mnist/

#include <gflags/gflags.h>
#include <glog/logging.h>
#include <google/protobuf/text_format.h>

#if defined(USE_LEVELDB) && defined(USE_LMDB)
#include <leveldb/db.h>
#include <leveldb/write_batch.h>
#include <lmdb.h>
#endif

#include <stdint.h>
#include <sys/stat.h>

#include <fstream>  // NOLINT(readability/streams)
#include <string>

#include "boost/scoped_ptr.hpp"
#include "caffe/proto/caffe.pb.h"
#include "caffe/util/db.hpp"
#include "caffe/util/format.hpp"

#if defined(USE_LEVELDB) && defined(USE_LMDB)

using namespace caffe;  // NOLINT(build/namespaces)
using boost::scoped_ptr;
using std::string;

// GFALGS工具定义命令行选项，默认值为lmdb，定义方式为: --backend=lmdb
DEFINE_string(backend, "lmdb", "The backend for storing the result");

// 大小端转换。MNIST原始数据文件中32为的整型值为大端存储，C/C++变量为小端存储，因此需要加入转换机制，MNIST文件中的说明为魔数
uint32_t swap_endian(uint32_t val) {
    val = ((val << 8) & 0xFF00FF00) | ((val >> 8) & 0xFF00FF);
    return (val << 16) | (val >> 16);
}

void convert_dataset(const char* image_filename, const char* label_filename,
        const char* db_path, const string& db_backend) {
  // Open files
  // 用C++输入文件流以二进制方式打开文件
  std::ifstream image_file(image_filename, std::ios::in | std::ios::binary);
  std::ifstream label_file(label_filename, std::ios::in | std::ios::binary);
  // CHECK宏来自于google的glog库，类似于标准库中的assert宏
  // 给定条件不满时终止程序，条件不满时输出后面程序
  CHECK(image_file) << "Unable to open file " << image_filename;
  CHECK(label_file) << "Unable to open file " << label_filename;
  // Read the magic and the meta data
  uint32_t magic;
  uint32_t num_items;
  uint32_t num_labels;
  uint32_t rows;
  uint32_t cols;

  // read()是以字符的类型读取，而MNIST的魔数由32位整型组成，并且使用大端存储
  // 应读4个字符共32位，read()将4个字符读取到magic地址的缓存中
  // 必要时进行强制类型转换，并将其转换为小端存储。总条目数，行数，列数同样如此
  image_file.read(reinterpret_cast<char*>(&magic), 4);
  magic = swap_endian(magic);
  // 将大端存储转换为小端存储后
  // 使用同样来自于Glog的CHECK_EQ来判断其是否与2051相等
  // 标签文件则要判断其是否与2049相等
  CHECK_EQ(magic, 2051) << "Incorrect image file magic.";
  label_file.read(reinterpret_cast<char*>(&magic), 4);
  magic = swap_endian(magic);
  CHECK_EQ(magic, 2049) << "Incorrect label file magic.";
  image_file.read(reinterpret_cast<char*>(&num_items), 4);
  num_items = swap_endian(num_items);
  label_file.read(reinterpret_cast<char*>(&num_labels), 4);
  num_labels = swap_endian(num_labels);
  CHECK_EQ(num_items, num_labels);
  image_file.read(reinterpret_cast<char*>(&rows), 4);
  rows = swap_endian(rows);
  image_file.read(reinterpret_cast<char*>(&cols), 4);
  cols = swap_endian(cols);


  scoped_ptr<db::DB> db(db::GetDB(db_backend));
  db->Open(db_path, db::NEW);
  scoped_ptr<db::Transaction> txn(db->NewTransaction());

  // Storing to db
  // 将数据存储到db中
  char label;
  char* pixels = new char[rows * cols];
  int count = 0;
  string value;

  // Datum是caffe.proto定义的message，这里用来存储图片
  // 由于image_file是采用unsigned byte类型，Datum中也有相似的bytes类型
  // 因此这里采用data来进行存取，Datum也支持float类型的数据
  // C++里没有byte类型，但是有与之相似的char类型，用其保存byte
  Datum datum;
  datum.set_channels(1);
  datum.set_height(rows);
  datum.set_width(cols);
  LOG(INFO) << "A total of " << num_items << " items.";
  LOG(INFO) << "Rows: " << rows << " Cols: " << cols;
  for (int item_id = 0; item_id < num_items; ++item_id) {
    image_file.read(pixels, rows * cols);
    label_file.read(&label, 1);
    datum.set_data(pixels, rows*cols);
    datum.set_label(label);
    string key_str = caffe::format_int(item_id, 8);
    datum.SerializeToString(&value);

    txn->Put(key_str, value);

    if (++count % 1000 == 0) {
      txn->Commit();
    }
  }
  // write the last batch
  if (count % 1000 != 0) {
      txn->Commit();
  }
  LOG(INFO) << "Processed " << count << " files.";
  delete[] pixels;
  db->Close();
}

int main(int argc, char** argv) {
#ifndef GFLAGS_GFLAGS_H_
  namespace gflags = google;
#endif

  FLAGS_alsologtostderr = 1;

  gflags::SetUsageMessage("This script converts the MNIST dataset to\n"
        "the lmdb/leveldb format used by Caffe to load data.\n"
        "Usage:\n"
        "    convert_mnist_data [FLAGS] input_image_file input_label_file "
        "output_db_file\n"
        "The MNIST dataset could be downloaded at\n"
        "    http://yann.lecun.com/exdb/mnist/\n"
        "You should gunzip them after downloading,"
        "or directly use data/mnist/get_mnist.sh\n");
  gflags::ParseCommandLineFlags(&argc, &argv, true);

  // FLAGS_backend在前面通过DEFINE_string定义，是字符串类型
  const string& db_backend = FLAGS_backend;

  if (argc != 4) {
    gflags::ShowUsageWithFlagsRestrict(argv[0],
        "examples/mnist/convert_mnist_data");
  } else {
    google::InitGoogleLogging(argv[0]);
    convert_dataset(argv[1], argv[2], argv[3], db_backend);
  }
  return 0;
}
#else
int main(int argc, char** argv) {
  LOG(FATAL) << "This example requires LevelDB and LMDB; " <<
  "compile with USE_LEVELDB and USE_LMDB.";
}
#endif  // USE_LEVELDB and USE_LMDB
```
数据类型多种多样，不可能用一套代码实现所有类型输入数据的读取。转化成统一格式可以简化数据读取层的实现，另一方面，使用LMDB可以提高磁盘的IO利用率。

接下来我们用LeNet-5模型来进行训练，原版的LeNet-5模型稍有不同，比如将激活函数由Sigmoid改为ReLU，该模型存放在examples/mnist/lenet_train_test.prototxt中。使用python/draw_net.py可以查看网络结构。

## 训练超参数
caffe也为该例程提供了训练的超参数和脚本，查看examples/mnist/train_lenet.sh脚本。
``` bash
#!/usr/bin/env sh
set -e

./build/tools/caffe train --solver=examples/mnist/lenet_solver.prototxt $@
```
该脚本使用了build/tools/caffe.bin的二进制程序(caffe是指向caffe.bin的)，并且使用examples/mnist/lenet_solver.prototxt作为求解器。
``` protobuf
# The train/test net protocol buffer definition
net: "examples/mnist/lenet_train_test.prototxt"
# test_iter specifies how many forward passes the test should carry out.
# In the case of MNIST, we have test batch size 100 and 100 test iterations,
# covering the full 10,000 testing images.
test_iter: 100
# Carry out testing every 500 training iterations.
test_interval: 500
# The base learning rate, momentum and the weight decay of the network.
base_lr: 0.01
momentum: 0.9
weight_decay: 0.0005
# The learning rate policy|学习速率的衰减策略
lr_policy: "inv"
gamma: 0.0001
power: 0.75
# Display every 100 iterations|每经过100次迭代，在屏幕上打印一次运行log
display: 100
# The maximum number of iterations
max_iter: 10000
# snapshot intermediate results|每5000次迭代打印一次快照
snapshot: 5000
snapshot_prefix: "examples/mnist/lenet"
# solver mode: CPU or GPU|Caffe求解为CPU模式。可以改为GPU模式
solver_mode: GPU
```
执行脚本后会在examples/mnist下生成四个文件，因为每5000次迭代会生成一次快照。具体caffe的运作机制可以在tools/caffe.cpp源代码中查看。
```
lenet_iter_10000.caffemodel
lenet_iter_10000.solverstate
lenet_iter_5000.caffemodel
lenet_iter_5000.solverstate
```
在测试中我们选择的模型依旧为lenet_train_test.prototxt，它既包括训练的模型，也包括测试的模型，权重选择迭代10000次后的模型lenet_iter_10000.caffemodel，选择100次迭代为一个batch，100个batch刚好可以测试完10000张图片，并且使用指定的gpu进行测试。
```
./build/tools/caffe test -model examples/mnist/lenet_train_test.prototxt -weights examples/mnist/lenet_iter_10000.caffemodel -iterations 100 -gpu 0
```
效果如下所示，正确率达到了99.07%。
```
I0116 17:55:27.196736 13017 caffe.cpp:309] Loss: 0.0284669
I0116 17:55:27.196743 13017 caffe.cpp:321] accuracy = 0.9907
I0116 17:55:27.196750 13017 caffe.cpp:321] loss = 0.0284669 (* 1 = 0.0284669 loss)
```