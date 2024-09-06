---
layout: post
title: fcitx5-rime + rime-ice
tags: [manjaro]
---

## Package Installation
- `fcitx5` framework
- `fcitx5-gtk` for enabling input method in GTK applications
- `fcitx5-rime` input engine
- `fcitx5-configtool` + `fcitx5-gt` for GUI configuration interface
- `rime-ice` input method

## Configuration Information
- OS: Manjaro Linux x86_64
- Kernel: 6.9.10-1-MANJARO 
- Window System: X11
- DE: Plasma 6.0.5 
- WM: KWin 

## Configuration Steps
1. Download and install the packages
```shell
yay -S fcitx5 fcitx5-gtk fcitx5-rime
```
2. Configure the environment by adding the following lines to `/etc/environment`
```
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```
3. Download the GUI tool for fcitx configuration
```shell
yay -S fcitx-configtool
```
4. Add the Rime input method in the GUI interface
5. Create a file named `default.custom.yaml` in the `~/.local/share/fcitx5/rime/` directory with the following content:
```yaml
patch:
  schema_list:
  - schema: rime_ice
```
6. Restart the system
