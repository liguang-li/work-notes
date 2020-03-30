* Change the resource to aliyu
	1. software & updates
		mirrors.aliyu.com
	Other software
			Canonical partners
	sudo apt update
* Sougou Pinyin
	sudo apt install fcitx-bin
	sudo apt update --fix-missing
	sudo apt install fcitx-bin
	sudo apt install fcitx-table
	Download sogoupinyin
	sudo dpdk -i sogoupinyin

* WPS
	sudo dpkg -i wps*.deb
	由于Linux版权原因，WPS缺少字体，故我们要安装WPS所需要的字体
	sudo mkdir /usr/share/fonts/WPS-Fonts
	**https://pan.baidu.com/s/1QJbyRDP7yu7F3pryKgdrYQ**
	sudo unzip wps_symbol_fonts.zip -d /usr/share/fonts/WPS-Fonts/
	sudo mkfontscale
	sudo mkfontdir
	sudo fc-cache

* 音视频
	sudo apt-get install ubuntu-restricted-extras 
	sudo apt-get install vlc browser-plugin-vlc

	ffmpeg
	sudo add-apt-repository ppa:djcj/hybrid
	sudo apt-get update
	sudo apt-get install ffmpeg
* tweaks
	sudo apt-get install gnome-tweak-tool
	sudo apt-get install gnome-shell-extensions -y 
	sudo apt install chrome-gnome-shell	##为了能在浏览器内安装gnome插件，火狐和谷歌都能用
	sudo apt-get install gtk2-engines-pixbuf #防止GTK2错误
	sudo apt install libxml2-utils

* Nvidia
	sudo apt purge nvidia-*

	sudo add-apt-repository ppa:graphics-drivers/ppa
	sudo apt-get update
	sudo apt-get upgrade
	ubuntu-drivers devices  #查看自己的显卡及可以安装的驱动版本

	sudo apt install nvidia-driver-***

