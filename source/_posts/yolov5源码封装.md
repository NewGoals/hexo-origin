---
title: yolov5源码封装
commends: false
date: 2023-09-06 17:26:42
categories:
tags:
---

</br>

<center>在某些工程中，处于对源代码的保护，需要对相应的工程源码进行封装，尤其是在神经网络的离线训练中。另外，为方便部署，还需将该网络进行训练的环境一并封装即可直接进行部署</center>

<!--more-->

> 封装的方式多种多样，但考虑到封装后的代码迁移能力与封装的繁琐程度，最终选定使用Pyinstaller将yolov5的源代码进行封装，并使用基于pyinstaller的auto-py-to-exe工具。
>
> 参考资料：https://zhuanlan.zhihu.com/p/130328237

## 安装auto-py-to-exe

```shell
pip install auto-py-to-exe
```

使用auto-py-to-exe进行封装时，由于该工具能够将python环境中的相关依赖库一并打包，以保证在部署机器上无需配置环境直接运行，因此，在进行打包前，最好使用anaconda工具新建一个python环境，仅安装该项目的必要依赖库，并在该conda环境下运行 `auto-py-to-exe`命令启动该工具。

![image-20230906093504654](../imgs/yolov5%E6%BA%90%E7%A0%81%E5%B0%81%E8%A3%85/image-20230906093504654.png)

## 配置yolov5的启动文件

一般来说，yolov5常常使用命令行工具（CLI）来启动训练或者推理，不像yolov8一样可以使用ultralytics的model来在python中直接启动训练，但是yolov5中仍提供了python中启动的run函数。

```python
def run(**kwargs):
    # Usage: import train; train.run(data='coco128.yaml', imgsz=320, weights='yolov5m.pt')
    opt = parse_opt(True)
    for k, v in kwargs.items():
        setattr(opt, k, v)
    main(opt)
    return opt
```

因此，可通过在方式，在根目录下新建一个trainv5.py的文件来启动所需要的所有训练工具。

```python
from utils.socket_module import client_socket
from segment import train as segmentTrain
from classify import train as classifyTrain
import train as detectTrain
import export_1 as exp
import yaml
import os
import multiprocessing


if __name__ == '__main__':
    multiprocessing.freeze_support()
    pwd = os.getcwd()
    abs_path = os.path.join(pwd, 'config.yaml')
    # 读取 config.yaml 文件
    with open(abs_path, 'r') as config_file:
        config = yaml.safe_load(config_file)

    # 获取 selectMode 属性的值
    select_mode = config.get('selectMode', None)

    # 根据 selectMode 的不同值，读取相应的第二级属性值
    if select_mode == 'segment':
        segment_properties = config.get('segment', {})
        try:
            segmentTrain.run(**segment_properties)
        except Exception as e:
            error_message = f"An error occurred: {str(e)}"
            print(error_message)
            client_socket.sendall(error_message.encode())
            client_socket.close() 
    elif select_mode == 'detect':
        detect_properties = config.get('detect', {})
        try:
            detectTrain.run(**detect_properties)
        except Exception as e:
            error_message = f"An error occurred: {str(e)}"
            print(error_message)
            client_socket.sendall(error_message.encode())
            client_socket.close()    
    elif select_mode == 'classify':
        classify_properties = config.get('classify', {})
        try:
            classifyTrain.run(**classify_properties)
        except Exception as e:
            error_message = f"An error occurred: {str(e)}"
            print(error_message)
            client_socket.sendall(error_message.encode())
            client_socket.close()  
    elif select_mode == 'export':
        export_properties = config.get('export', {})
        try:
            exp.run(**export_properties)
        except Exception as e:
            error_message = f"An error occurred: {str(e)}"
            print(error_message)
            client_socket.sendall(error_message.encode())
            client_socket.close() 
    client_socket.close() 
    
```

### 运行时与主进程通信

在封装后由于整个代码不可调试，因此需要设计一些用于和主进程进行通信的模块，该项目中使用套接字方式与主进程进行通信，在utils目录下新建`socket_module.py`模块启动套接字客户端，相对应的服务端在后面解释。

```python
import socket

# create TP/ICP socket
server_address = ('localhost', 3334)
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client_socket.connect(server_address)
```

在该方式中，可以将`train.py`中相关数据从训练进程中通过套接字的方式传输出来，主进程中的套接字服务端进行接收。

```python
send_epoch_message = ("当前使用设备：" + '%-11s' + "已迭代轮次：" + '%-11s' + "预计剩余：" + '%-11s' + "单次迭代平均耗时：" + '%-11s' + "验证集精准率：" + '%-11.4f' + "验证集召回率：" + '%-11.4f') % (device, f'{epoch + 1}/{epochs}', f'{int(remain_time // 3600)}h{int((remain_time % 3600) // 60)}m{int(remain_time % 60)}s', f'{int(epoch_time // 3600)}h{int((epoch_time % 3600) // 60)}m{int(epoch_time % 60)}s', metrics_dict['metrics/precision(B)'], metrics_dict['metrics/recall(B)'])
        client_socket.sendall(send_epoch_message.encode())
```

如上述方式中，将使用设备，迭代轮次，预计剩余时间等信息传输到服务端进行训练精度的监控，同时还可在`trainv5.py`中使用套接字传输训练报错日志，如下所示。

```python
# 根据 selectMode 的不同值，读取相应的第二级属性值
    if select_mode == 'segment':
        segment_properties = config.get('segment', {})
        try:
            segmentTrain.run(**segment_properties)
        except Exception as e:
            error_message = f"An error occurred: {str(e)}"
            print(error_message)
            client_socket.sendall(error_message.encode())
            client_socket.close() 
```

### 防止不必要的联网与gitpython出错

- 解决gitpython报错

在运行封装过后的`.exe`文件时，yolov5在运行的过程中会启用git的check检查，并且试图在新部署机器的命令行中运行，但由于yolov5运行时已经将其依赖的环境全部封装在一起，并不会在新机器中再次安装环境，因此会报错。而离线环境中，我们并不关心yolov5的版本如何，也无需进行下载。

下面找到相关训练工具，如`train.py`，`/segment/train.py`等文件，注释以下语句。

```python
# GIT_INFO = check_git_info()
ckpt = {
                    'epoch': epoch,
                    'best_fitness': best_fitness,
                    'model': deepcopy(de_parallel(model)).half(),
                    'ema': deepcopy(ema.ema).half(),
                    'updates': ema.updates,
                    'optimizer': optimizer.state_dict(),
                    'opt': vars(opt),
                    # 'git': GIT_INFO,  # {remote, branch, commit} if a git repo
                    'date': datetime.now().isoformat()}
# check_git_status()
# check_requirements(ROOT / 'requirements.txt')
```

- AMP检查联网

在进行amp检查时，尤其是在网络情况不好时，常常导致启动速度过慢，因此，删除该检查中的下载模块以加快启动速度，需修改的文件为`utils/genenal.py`的check_amp函数，替换一下语句。

```python
# im = f if f.exists() else 'https://ultralytics.com/images/bus.jpg' if check_online() else np.ones((640, 640, 3))
im = f if f.exists() else np.ones((640, 640, 3))
```

### 多进程封装问题

在 Windows 系统上，`multiprocessing` 模块在执行多进程程序时会创建一个新的 Python 进程，这个新的进程会重新导入和执行主程序（包括你的多进程代码），这可能会导致问题，因为在主程序中执行的一些初始化代码可能会再次执行，导致冲突或错误。

为了解决这个问题，`multiprocessing.freeze_support()` 函数应该在主程序的入口点处调用。这个函数的作用是检查当前的进程是否是 "frozen" 进程，如果是，就不再执行初始化代码，从而避免问题。如果不是 "frozen" 进程（通常是在正常运行时），则函数不会执行任何操作，只是简单地返回。

```python
import multiprocessing
if __name__ == '__main__':
    multiprocessing.freeze_support()
```

### 读取训练图像的多进程问题

在使用dataloader时，yolov5会调用utils/dataloaders.py中的loadImagesAndLabels类下的如下代码块。

```python
# Check cache
        self.label_files = img2label_paths(self.im_files)  # labels
        cache_path = (p if p.is_file() else Path(self.label_files[0]).parent).with_suffix('.cache')
        try:
            cache, exists = np.load(cache_path, allow_pickle=True).item(), True  # load dict
            assert cache['version'] == self.cache_version  # matches current version
            assert cache['hash'] == get_hash(self.label_files + self.im_files)  # identical hash
        except Exception:
            cache, exists = self.cache_labels(cache_path, prefix), False  # run cache ops
```

该方法会先读取训练集或验证集中的cache文件，并对version与hash值进行比对，如果无法对应，则会运行cache_labels方法重新生成cache文件。

```python
def cache_labels(self, path=Path('./labels.cache'), prefix=''):
        # Cache dataset labels, check images and read shapes
        x = {}  # dict
        nm, nf, ne, nc, msgs = 0, 0, 0, 0, []  # number missing, found, empty, corrupt, messages
        desc = f'{prefix}Scanning {path.parent / path.stem}...'
        # with Pool(NUM_THREADS) as pool:
        with Pool(1) as pool:
            pbar = tqdm(pool.imap(verify_image_label, zip(self.im_files, self.label_files, repeat(prefix))),
                        desc=desc,
                        total=len(self.im_files),
                        bar_format=TQDM_BAR_FORMAT)
            for im_file, lb, shape, segments, nm_f, nf_f, ne_f, nc_f, msg in pbar:
                nm += nm_f
                nf += nf_f
                ne += ne_f
                nc += nc_f
                if im_file:
                    x[im_file] = [lb, shape, segments]
                if msg:
                    msgs.append(msg)
                pbar.desc = f'{desc} {nf} images, {nm + ne} backgrounds, {nc} corrupt'

        pbar.close()
        if msgs:
            LOGGER.info('\n'.join(msgs))
        if nf == 0:
            LOGGER.warning(f'{prefix}WARNING ⚠️ No labels found in {path}. {HELP_URL}')
        x['hash'] = get_hash(self.label_files + self.im_files)
        x['results'] = nf, nm, ne, nc, len(self.im_files)
        x['msgs'] = msgs  # warnings
        x['version'] = self.cache_version  # cache version
        try:
            np.save(path, x)  # save cache for next time
            path.with_suffix('.cache.npy').rename(path)  # remove .npy suffix
            LOGGER.info(f'{prefix}New cache created: {path}')
        except Exception as e:
            LOGGER.warning(f'{prefix}WARNING ⚠️ Cache directory {path.parent} is not writeable: {e}')  # not writeable
        return x
```

在上述cache_labels的方式实现中，启用了`multiprocessing`的线程池，而在封装后的`.exe`文件执行时，该段代码执行出错，因此在这段代码中，将线程的数量设置为1避免该错误。经测试，在训练集不大的情况下，在加载时并不会导致明显的效率降低。

### 训练执行的多进程问题

在代码的执行过程中，dataloader的numworkers通常用来设置多进程处理的工作线程数，而在封装的过程中，该问题同样会导致程序无法正常运行，该问题和上面的问题类似，应当是会有能够完美运行的方式，但由于时间关系，我仍选择使用最简单的方式，尽快会对效率造成一定的影响。

```python
return loader(dataset,
                  batch_size=batch_size,
                  shuffle=shuffle and sampler is None,
                #   num_workers=nw,
                  num_workers=0,
                  sampler=sampler,
                  pin_memory=PIN_MEMORY,
                  collate_fn=LoadImagesAndLabels.collate_fn4 if quad else LoadImagesAndLabels.collate_fn,
                  worker_init_fn=seed_worker,
                  generator=generator), dataset
```

yolov5源码中全局搜索numworkers，将该项全部置0。

> 值得注意的是，如果涉及到yolov8的封装，启动文件如果在根目录下，其会调用源码中的`dataloader.py`执行，如果启动文件在其他目录下，在训练时其会由于无法检索到相关文件而调用ultralytics库中的`dataloader.py`，因此，如果不在源码根目录下对启动文件打包，那么请务必修改pip安装第三方ultralytics库中的源码。

## 使用auto-py-to-exe进行封装

>Auto PY to EXE 是一个用户友好的图形界面工具，用于将 Python 脚本转换为独立的可执行文件（.exe）或可执行脚本，以便在 Windows 上运行，而无需安装 Python 解释器。以下是 Auto PY to EXE 中的一些主要功能选项：

1. **Script**: 这是指您要转换的 Python 脚本文件的位置。您可以单击 "Browse" 按钮选择您的 Python 脚本文件。
2. **One Directory**: 这个选项允许您将所有生成的文件保存在一个目录中，以便简化项目结构。
3. **Console Window**: 如果您的 Python 脚本需要控制台窗口（命令行界面）来与用户交互，则选择此选项。否则，如果脚本是一个 GUI 应用程序，则取消选择此选项。
4. **Window based (hide the console)**: 如果您的 Python 脚本是一个 GUI 应用程序，但希望在运行时隐藏控制台窗口，可以选择此选项。
5. **Disable console (no output)**: 如果您的 Python 脚本是一个 GUI 应用程序且不会产生任何控制台输出，可以选择此选项以完全禁用控制台窗口。
6. **Add a custom icon**: 这个选项允许您为生成的可执行文件指定一个自定义图标。
7. **Additional Files**: 如果您的 Python 脚本依赖于其他文件，如数据文件、配置文件等，可以在这里添加这些附加文件，以便它们也包含在生成的可执行文件中。
8. **Excluded Modules**: 如果您希望排除某些 Python 模块（库）以减小生成的可执行文件的大小，可以在这里指定这些模块。
9. **Zip Dependencies**: 这个选项允许您将依赖项压缩成一个 Zip 文件，以减小可执行文件的大小。
10. **Advanced**: 在 "Advanced" 选项卡中，您可以进行更高级的配置，如指定自定义 Python 解释器、设置环境变量等。
11. **Logs**: 这个选项允许您启用详细的日志记录，以便在转换过程中出现问题时进行故障排除。
12. **Compile**: 最后，单击 "Compile" 按钮开始将 Python 脚本转换为可执行文件的过程。

以下为我在进行封装时的config文件。

```json
{
 "version": "auto-py-to-exe-configuration_v1",
 "pyinstallerOptions": [
  {
   "optionDest": "noconfirm",
   "value": true
  },
  {
   "optionDest": "filenames",
   "value": "E:/yolov5/trainv5.py"
  },
  {
   "optionDest": "onefile",
   "value": false
  },
  {
   "optionDest": "console",
   "value": true
  },
  {
   "optionDest": "name",
   "value": "trainv5_logE"
  },
  {
   "optionDest": "ascii",
   "value": false
  },
  {
   "optionDest": "clean_build",
   "value": false
  },
  {
   "optionDest": "strip",
   "value": false
  },
  {
   "optionDest": "noupx",
   "value": false
  },
  {
   "optionDest": "disable_windowed_traceback",
   "value": false
  },
  {
   "optionDest": "embed_manifest",
   "value": true
  },
  {
   "optionDest": "uac_admin",
   "value": false
  },
  {
   "optionDest": "uac_uiaccess",
   "value": false
  },
  {
   "optionDest": "win_private_assemblies",
   "value": false
  },
  {
   "optionDest": "win_no_prefer_redirects",
   "value": false
  },
  {
   "optionDest": "bootloader_ignore_signals",
   "value": false
  },
  {
   "optionDest": "argv_emulation",
   "value": false
  },
  {
   "optionDest": "datas",
   "value": "D:/Code/Anaconda3/envs/yolov5env/Lib/site-packages/ultralytics/yolo/cfg/default.yaml;./ultralytics/yolo/cfg"
  },
  {
   "optionDest": "datas",
   "value": "D:/Code/Anaconda3/envs/yolov5env/Lib/site-packages/orderedmultidict/__version__.py;./orderedmultidict"
  },
  {
   "optionDest": "datas",
   "value": "E:/yolov5/data/hyps;./data/hyps/"
  }
 ],
 "nonPyinstallerOptions": {
  "increaseRecursionLimit": true,
  "manualArguments": ""
 }
}
```

在图形化界面中的选项如下图所示。

![image-20230906103456448](../imgs/yolov5%E6%BA%90%E7%A0%81%E5%B0%81%E8%A3%85/image-20230906103456448.png)

> 值得注意的是，在Additional Files部分，所需要添加的文件均为在封装时，根据运行时的报错提示进行追加，其原因是在进行封装时，封装后的根目录发生改变，源码中有些使用绝对路径的文件或者因为其他的原因导致部分文件搜索不到，需要根据提示将其添加到相应的目录下。

该项目将整个工程封装为一个文件夹而不是封装成一个文件，其原因在于封装成一个文件会导致其将相关的动态库和环境全部压缩，在执行该文件时还要先进行解压从而导致启动非常缓慢，因此将其封装为文件夹的形式，动态库均存在于文件夹下，调用方便无需进行解压。

![image-20230906104231256](../imgs/yolov5%E6%BA%90%E7%A0%81%E5%B0%81%E8%A3%85/image-20230906104231256.png)

执行`.exe`文件即可运行。

## 进程调用

### 配置套接字服务端

在调用封装后的yolov5中，由于训练程序和调用程序属于两个不同的进程，仅此接收信息需要用到套接字通信，前面已经在训练的源码中配置完套接字的客户端，现在进行套接字的服务端配置以接收消息。

```C++
void ServerLogThread(){
    WSADATA wsaData;
    WSAStartup(MAKEWORD(2, 2), &wsaData);

    SOCKET serverSocket = socket(AF_INET, SOCK_STREAM, 0);
    
    sockaddr_in serverAddress;
    serverAddress.sin_family = AF_INET;
    serverAddress.sin_addr.s_addr = INADDR_ANY;
    serverAddress.sin_port = htons(3334);

    bind(serverSocket, (struct sockaddr*)&serverAddress, sizeof(serverAddress));
    listen(serverSocket, 1);

    std::cout << "Waiting for a connection..." << std::endl;

    SOCKET clientSocket = accept(serverSocket, NULL, NULL);
    std::cout << "Connected!" << std::endl;

    char previousBuffer[1024];
    char currentBuffer[1024];

    memset(previousBuffer, 0, sizeof(previousBuffer));
    memset(currentBuffer, 0, sizeof(currentBuffer));

    bool clientConnected = true;

    while(clientConnected){
        memset(currentBuffer, 0, sizeof(currentBuffer));
        
        int bytesReceived = recv(clientSocket, currentBuffer, sizeof(currentBuffer), 0);
    
        if (bytesReceived == 0) {
            // Client has closed the connection
            std::cout << "Client has closed the connection." << endl;
            clientConnected = false;
        } else if (bytesReceived > 0 && strcmp(previousBuffer, currentBuffer) != 0) {
            std::cout << currentBuffer << endl;
            strcpy(previousBuffer, currentBuffer);
        }
    }

    closesocket(clientSocket);
    closesocket(serverSocket);
    WSACleanup();
}
```

### yaml文件配置训练参数

为保证在调用进程时不直接向该执行文件传参，在上述源码的启动文件的设计中，即是用yaml文件的读取来获得yolov5的运行参数，因此，在调用该封装文件前，需要对yaml文件的参数进行配置，C++中使用`yaml-cpp`库来完成。

```C++
void updateConfig() {
    try {
        YAML::Node config = YAML::LoadFile("config.yaml");  

        std::string selectedMode = config["selectMode"].as<std::string>();

        std::string modeInput;
        std::getline(std::cin, modeInput);

        if (!modeInput.empty()) {
            config["selectMode"] = modeInput;
            selectedMode = modeInput;
            std::ofstream modifiedConfig("config.yaml");
            modifiedConfig << config;
        } else {
            std::cout << "No mode settings provided" << std::endl;
        }

        if (selectedMode == "segment" || selectedMode == "detect" || selectedMode == "classify") {
            YAML::Node modeSettings = config[selectedMode];

            std::cout << selectedMode << " Settings:" << std::endl;
            for (const auto& keyValue : modeSettings) {
                std::string key = keyValue.first.as<std::string>();
                std::string value = keyValue.second.as<std::string>();
                std::cout << key << ": " << value << std::endl;
            }

            std::cout << "Enter new settings for " << selectedMode << " (key=value pairs separated by space): ";
            std::string modifyInput;
            std::getline(std::cin, modifyInput);

            if (!modifyInput.empty()) {
                std::istringstream iss(modifyInput);
                std::string newKey, newValue;
                while (std::getline(iss, newKey, '=')) {
                    std::getline(iss, newValue, ' ');
                    if (!newValue.empty()) {
                        modeSettings[newKey] = newValue;
                    }
                }

                std::ofstream modifiedConfig("config.yaml");
                modifiedConfig << config;
                std::cout << "Updated configuration saved to config.yaml" << std::endl;
            } else {
                std::cout << "No new settings provided. Configuration remains unchanged." << std::endl;
            }
        } else {
            std::cout << "Invalid selectMode specified." << std::endl;
        }
    } catch (const YAML::Exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
    }
}
```

其中`config.yaml`的配置如下所示，可进行多种形式的训练与文件导出，其具体功能实现参考启动文件中代码部分。

```yaml
selectMode: segment
segment:
  weights: yolov5s-seg.pt
  data: D:/Code/autopy2exeTest/datasets/coco128-seg/coco128-sg.yaml
  epochs: 10
  batch_size: 2
  imgsz: 640
detect:
  weights: yolov5n.pt
  data: D:/Code/autopy2exeTest/datasets/Mask-Wearing-19/data.yaml
  epochs: 20
  batch_size: 2
classify:
  model: yolov5s-cls.pt
  data: cifar100
  epochs: 5
  img: 224
export:
  weights: yolov5n.pt
  include: [onnx]
  device: 0
```

### 调用训练进程

在主进程中新建子进程调用封装后的yolov5进行训练。

```C++
#include <winsock2.h>
#include <Windows.h>
#include <thread>
#include <string>
#include <cstring>
#include <fstream>
#include <yaml-cpp/yaml.h>
#include <iostream>

using namespace std;

#define EXEPATH "D:/Code/autopy2exeTest/trainv5_logE/trainv5_logE.exe"

void ServerLogThread();     // 输出日志
void updateConfig();        // 更新配置文件

int main() {
    // 要启动的 .exe 文件路径
    TCHAR szCommandLine[] = TEXT(EXEPATH);

    // 进程启动信息
    STARTUPINFO startupInfo;
    PROCESS_INFORMATION processInfo;
    memset(&startupInfo, 0, sizeof(startupInfo));
    memset(&processInfo, 0, sizeof(processInfo));
    startupInfo.cb = sizeof(startupInfo);

    // 更新配置文件
    updateConfig();
    
    // 启动server log线程
    thread serverLogThread(ServerLogThread);

    // 启动.exe 文件
    BOOL success = CreateProcess(NULL, szCommandLine, NULL, NULL, FALSE, CREATE_NO_WINDOW, NULL, NULL, &startupInfo, &processInfo);

    if (success) {
        // 进程启动成功
        // 可以使用 processInfo.hProcess 来控制和监视进程
        cout << "Process start success." << endl;
        
        WaitForSingleObject(processInfo.hProcess, INFINITE);

        cout << "Process finished." << endl;
        // 关闭句柄
        CloseHandle(processInfo.hProcess);
        CloseHandle(processInfo.hThread);
    } else {
        // 进程启动失败
        DWORD error = GetLastError();
        // 处理错误
        cout << "Process start failed!" << endl;
        return 2;
    }
    return 0;
}
```

该部分使用windows中的`CreateProcess`来启用一个子进程，并且使用`WaitForSingleObject`方式等待其执行完成。由于在等待时会导致主线程阻塞，因此在启动子进程之前，还需要创建一个线程用来执行套接字的服务端，并不断地接收消息并显示。

