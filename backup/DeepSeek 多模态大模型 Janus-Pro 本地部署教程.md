# DeepSeek 多模态大模型 Janus-Pro 本地部署教程

## 下载模型仓库

 git clone https://github.com/deepseek-ai/Janus.git下载模型仓库
 
国内下载仓库失败时，可以使用以下代理：

git clone https://github.moeyy.xyz/https://github.com/deepseek-ai/Janus.git
准备 Conda 3.12 虚拟环境
conda create --name deepseek7B python=3.12

conda activate deepseek7B
Janus 安装基础依赖
cd Janus

## 安装基础依赖
pip install -e .

## 安装 gradio ui 界面
pip install gradio

## 启动带有gradio的ui界面，同时下载14GB+的大模型
python demo/app_januspro.py
启动成功后,在浏览器里输入：http://127.0.0.1:7860，即可访问 Gradio 页面。

下载模型时，如果你的服务器无法访问模型地址 huggingface.co，则可以用 modelscope

使用 modelscope 下载模型
官网：https://modelscope.cn/

pip install modelscope

modelscope download --model deepseek-ai/Janus-Pro-7B
默认模型会下载到~/.cache/modelscope/hub目录下，指定 local_dir 可以选择本地下载路径

modelscope download --model deepseek-ai/Janus-Pro-7B --local_dir 'path/to/dir'
打开 ./demo/app_januspro.py 文件，并在15行处，修改为当前本地模型下载地址路径

## 接着就可以启动服务：python demo/app_januspro.py

修改 gradio 地址或端口
打开 ./demo/app_januspro.py 文件，在244行处，进行修改

在这里设置 server_name='0.0.0.0' 以便从任何 IP 访问

demo.launch(share=True, server_name='0.0.0.0', server_port=7860)