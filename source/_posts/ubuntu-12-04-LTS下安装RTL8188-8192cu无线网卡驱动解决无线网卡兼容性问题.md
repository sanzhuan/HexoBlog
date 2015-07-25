title: "ubuntu 12.04 LTS下安装RTL8188/8192cu无线网卡驱动解决无线网卡兼容性问题"
date: 2015-05-21 18:30:16
tags: linux 
---


1.到[官网](http://www.realtek.com.tw/downloads/downloadsView.aspx?Langid=1&PFid=48&Level=5&Conn=4&ProdID=277&DownTypeID=3&GetDown=false&Downloads=true)上下载RTL8192cu的linux版驱动。

2.在`/etc/modules`最后加上`8192cu`，在`/etc/modprobe.d/blacklist.conf`最后加上`blacklist rtl8192cu`。

```bash
echo "8192cu" >> /etc/modules
echo "blacklist rt8192cu" >> /etc/modprobe.d/blacklist.conf
```

3.到下载的驱动根文件夹下,运行`install.sh`即可。

```bash
chmod +x install.sh
./install.sh
```
