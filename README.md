# 基于Orangepi的大语言模型的简化工具链

/

* ## 首先

  让我们假设你已经有了一块orange pi5Plus --这好像是废话--
  >对了 6b以上模型最好16G以上
  
/

/

/

* ## 安装系统

  你需要下载并安装orangepi官方的 "Orangepi5plus_1.0.10_ubuntu_jammy_linux6.1.43"系统

  地址:https://pan.baidu.com/s/1cQR1pcca0P-xuQbTrnGXAw?pwd=pjhv#list/path=%2F

  建议选择server版本 系统占用内存相对小 方便我们运行大模型

/

/

/

* ## 编译内核
 
  下载并编译orangepi官方的6.1内核源码
  >这一步是为了更新一些驱动

  地址:https://github.com/orangepi-xunlong/linux-orangepi/tree/orange-pi-6.1-rk35xx
  检查是否是 "orange-pi-6.1-rk35xx" 分支

  将下载的包传至你的开发板 解压 

&

      如果你使用的是orangepi官方的散热器 是不是对官方那傻X散热策略感同深受?
      所以你可以使用我优化过的风扇驱动 就项目里那个 "rk3588-orangepi-5-plus.dts" 
      将其放入下载的内核源码内的 "/arch/arm64/boot/dts/rockchip/" 覆盖掉原来的同名驱动文件即可
      我实测即便高负载下风扇依然可以做到几乎无声 而温度却并没多大变化

&

  然后再下载项目文件里的 "Compiling_kernel.docx" 打开文档一步步进行编译操作

/

/

/

* ## 转换模型

  到这一步代表你已经编译完成了
  >如未成功 自行对照文档debug

  用: cat /sys/kernel/debug/rknpu/versioncat 确认一下npu驱动是否已经更新成功

  确保开发板输出大致为:
  >RKNPU driver: v0.9.6

&

  接着就是转换并量化模型了
  >注意下面转换这一步最好在另一台x86_64的linux主机上操作 转换是需要一定性能的 特别是大一点的模型 且内存最好大点 起码32GB+

  >要转换为rk芯片可识别的格式 量化主要是为了优化模型性能 不然就3588那个身板...

&

  首先安装好docker

  然后这里建议下载b站up主 "kaylordut" 的docker:

  >docker pull kaylor/rk3588_llm

  里面转换环境已经配置好了

&

  将 rknn-llm-main 文件夹传入x86主机

  之后去 Hugging Face 找一个兼容的模型
  >兼容什么模型 自行参阅 rknn-llm-main/doc/下的 "Rockchip_RKLLM_SDK_CN.pdf" 文档

  最好将整个模型下到 /rknn-llm-main/rkllm-toolkit/examples/huggingface/ 下 方便后续修改识别路径
  >可以在 Hugging Face网站 先选择下载一部分文件

  >然后进入对应模型文件夹用 git lfs pull 将剩下的大文件全下下来

&

  下载完毕后 进入 /examples/huggingface 里面有个test.py

  用文档编辑的方式打开 修改 modelpath 一项为你下载的模型的完整路径 保存

  然后将之前传入的 rknn-llm-main 映射入docker 顺便进入docker 比如:
  >docker run -it -v  /usr/local/AI/rknn-llm-main/rkllm-toolkit/examples/huggingface:/usr/local/AI/rknn-llm-main/rkllm-toolkit/examples/huggingface  kaylor/rk3588_llm
  
  >中间的路径自行修改为自己的

  cd 到/examples/huggingface 内

  用python运行 huggingface/ 下的 test.py
  >python test.py

  等它运行完成后看到这个就代表转换完成:
  >Model has been saved to ./qwen.rkllm!

  >并且输入 ls 能看到 huggingface/ 下有个 qwen.rkllm

  >如不是 请自行debug

&

  现在 下载项目内 "ai.tar.gz" 传至你的开发板并解压 这是包含命令行运行和web网页运行两种大模型运行方式的启动包
  >如你要调整自定义参数 请自行研究rknn-llm-main/doc/下的 "Rockchip_RKLLM_SDK_CN.pdf" 文档 当然 直接使用我调整好的也可

  x86主机退出docker
  
  然后在x86主机上将模型文件传入开发板的刚刚解压好的 ai/ 文件夹内:
  >scp ./qwen.rkllm 你的开发板用户@开发板ip:/ai/

  现在 转到开发板并检查 ai/ 下是否有个 qwen.rkllm 你也可以自行修改文件名

&

  自此 模型转换结束

/
  
/

/

* ## 运行模型

     在运行之前 你需要先安装OpenCL
     具体参考:
     >https://www.roselladb.com/install-opencl-orangepi5-debian-ubuntu.htm
     或者:
     >https://clehaxze.tw/gemlog/2023/06-17-setting-up-opencl-on-rk3588-using-libmali.gmi

     ### 控制台版本:
     用文件编辑器打开 ai/run-ai.sh

     taskset一栏需要更改:
     >taskset f0 ./启动程序 ./你转换的模型文件

     >两个启动程序: llm_demo-gpu-2k 是2k上下文和token数量上限 同理 llm_demo-gpu-32k 是32k

     修改完成保存退出 并运行:
     >./run-ai.sh

     看到这行即运行成功:
     >rkllm init success

     恭喜!

     ### web网页版本:
     用文件编辑器打开 ai/server/rkllm_server/run-ai-server.sh
     >python3 gradio_server.py --target_platform rk3588 --rkllm_model_path /你的模型文件路径

     保存退出并运行:
     >./run-ai-server.sh

     看到这些代表运行成功:
     >RKLLM初始化成功！
     
     >==============================
     
     >Running on local URL:  http://0.0.0.0:8080

     >To create a public link, set `share=True` in `launch()`.

     之后 你可以在web浏览器和你的模型对话 地址是:
     >你的开发板ip:8080

     恭喜!
     
  
     



