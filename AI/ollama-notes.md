## 配置docker国内源
查看docker安装方式
```
systemctl status docker
which docker
snap list | grep docker
```
![](../images/AI/ollama_notes_pic_001.png)

修改配置添加docker国内源
```
https://docker.m.daocloud.io
```
![](../images/AI/ollama_notes_pic_002.png)

## 安装nvidia驱动

查看是否有NVIDIA显卡信息
```
lspci | grep -i nvidia
```
![](../images/AI/ollama_notes_pic_003.png)

安装nvidia驱动
```
sudo apt install nvidia-driver-530
```
![](../images/AI/ollama_notes_pic_004.png)

更新内核模块依赖并重建 initramfs
```
sudo update-initramfs -u
sudo reboot
```
![](../images/AI/ollama_notes_pic_005.png)

## ollama拉起并运行模型
运行ollama容器
```
docker run -d --name ollama -p 11434:11434 ollama/ollama:latest
```
![](../images/AI/ollama_notes_pic_006.png)

使用 --runtime=nvidia 参数替换 --gpus all 参数
```
docker run -d --name ollama --runtime=nvidia -p 11434:11434 ollama/ollama:latest
```
![](../images/AI/ollama_notes_pic_007.png)

拉取并测试模型
```
docker exec -it ollama ollama pull nomic-embed-text:latest
docker exec -it ollama ollama list

curl http://localhost:11434/api/embeddings \
  -d '{
    "model": "nomic-embed-text",
    "prompt": "这是一个测试文本"
  }'

```
![](../images/AI/ollama_notes_pic_008.png)

> 测试模型返回的向量

