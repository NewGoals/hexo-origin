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
  // CHECK宏来自于google的glog库，类似于标准库中的assert宏，给定条件不满时终止程序，条件不满时输出后面程序
  CHECK(image_file) << "Unable to open file " << image_filename;
  CHECK(label_file) << "Unable to open file " << label_filename;
  // Read the magic and the meta data
  uint32_t magic;
  uint32_t num_items;
  uint32_t num_labels;
  uint32_t rows;
  uint32_t cols;

  // read()是以字符的类型读取，而MNIST的魔数由32位整型组成，并且使用大端存储，应读4个字符共32位，read()将4个字符读取到magic地址的缓存中，必要时进行强制类型转换，并将其转换为小端存储。总条目数，行数，列数同样如此
  image_file.read(reinterpret_cast<char*>(&magic), 4);
  magic = swap_endian(magic);
  // 将大端存储转换为小端存储后即可使用同样来自于Glog的CHECK_EQ来判断其是否与2051相等，标签文件则要判断其是否与2049相等
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
  char label;
  char* pixels = new char[rows * cols];
  int count = 0;
  string value;

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
```

