---
layout: mypost
title: Full NixOS Guide Everything You Need to Know in One Place!
categories: [NixOS]
---

## 起因

日常作为一个长期使用种Linux发行版的开发者，之前一直主力的Linux发行版是Ubuntu LTS版本，从最开始的Ubuntu 16.04 LTS使用，随着版本迭代到了目前最新的Ubuntu 24.04.02 LTS 并开启Ubuntu Pro 长期支持。
在Ubuntu 的使用和升级过程中也存在踩过各种坑，Ubuntu的更新机制也是频繁推送，外加可能夹杂的推广信息，每次的升级和迁移都是比较痛苦，所以有了换Linux发行版的念头。

在日常浏览各种新闻的时候，碰到NixOS的相关介绍，一下子感觉像是碰到新鲜的事物，迫不及待想去了解这个NixOS的特性。

NixOS核心特性中，最吸引人的一点是其声明式配置管理方式，熟悉Kubernetes的朋友一定不陌生这种管理方式，Kubernetes对于Pod和deployment之类的资源管理方式也是声明式的。基于声明式配置管理方式，你可以：

- 完整描述系统的所有配置
- 实现配置的版本控制
- 轻松完成系统环境的迁移
- 支持系统配置的回滚
- 确保系统配置的可重现性

以上的方式，可以让系统维护、迁移更加可控，声明式的配置大大降低了心智。

## 快速入门

对于初次接触的NixOS的用户，推荐在本地拉起虚拟机的方式，在本地虚拟机环境体验。在体验的过程中慢慢完善声明式的配置，等时机成熟，完成基于完善的配置文件的系统迁移。

### 系统安装

NixOS 提供了 Minimal ISO 和 带GUI的Desktop Evironment的镜像用于安装。虽然安装过程比传统 Linux 发行版略显复杂，但相比 Arch Linux 要简单得多。详细的安装指南请参考：

[NixOS Installation Guide](https://nixos.org/manual/nixos/stable/#ch-installation)


### 基础配置

NixOS的配置 不想其他各类Linux发行版，各个软件都默认在`/etc/`下生成自己配置文件，各种管理配置文件，NixOS所有的系统级、软件包级的配置都在`/etc/nixos/`这个目录下面，其基础配置就是`/etc/nixos/configuration.nix`
在开始阶段我们就是不断在编辑这个系统配置文件。

> 在系统完成初始化安装的情况下，系统不默认按照Vim, 可以通过以下命令 临时安装`vim`,具备vim编辑器

```shell
nix-shell -p vim
```

对于日常使用的情况下，丰富的软件包和好用的中文输入法 是决定Linux发行版是否好用的关键,对于这2点 NixOS都能满足要求


### 高级特性Home Manager和Flake

### NixOS的Home Manager

####  什么是 Home Manager？
`Home Manager` 是一个基于 Nix 的用户环境管理工具，允许你用声明式的配置文件统一管理：

- shell 配置（如 zsh、bash、fish）
- 编辑器和开发工具
- Git、GPG、SSH 等个性化设置
- 甚至是 GUI 应用的配置文件

与系统级 nixosConfiguration 不同，Home Manager 管理的是用户级配置，而不是整个操作系统。这意味着：

不需要 root 权限即可更新配置
配置可以方便地移植到其他机器
可以和非 NixOS 系统（如 macOS, Ubuntu）配合使用


### NixOS的Flakes

Flakes 是 `Nix 2.4+` 引入的新特性，它解决了传统 `Nix` 配置中的以下问题：

- 依赖不固定（版本漂移）
- 缺乏标准化结构（各种配置文件散乱）
- 无法方便地复用和分发

一个 Flakes 基本上就是一个带有 `flake.nix` 文件的 `Git` 仓库，它定义了：

- 配置项（nixosConfigurations、homeConfigurations 等）
- 输入依赖（inputs）
- 输出内容（outputs）

Flakes 带来：

可复现性（所有依赖和配置都固定版本）
可共享性（别人可以直接 nix flake clone 使用你的配置）
统一入口（`nix run / nix develop / nixos-rebuild switch --flake`）

### 用 Flake 管理 Home Manager 配置的好处

把系统级和用户级配置放在同一个仓库统一版本管理
系统和用户更新可以一次性完成（例如`nixos-rebuild switch --flake .`)
多主机、多用户的配置可以方便管理和复用



### 完整的配置文件

`configuration.nix`

```
# Edit this configuration file to define what should be installed on
# your system.  Help is available in the configuration.nix(5) man page
# and in the NixOS manual (accessible by running ‘nixos-help’).

{ config, pkgs, lib, ... }:

{  
  imports =
    [ # Include the results of the hardware scan.
      ./hardware-configuration.nix
    ];
  
  # Bootloader.
  boot.loader.grub.enable = true;
  boot.loader.grub.device = "/dev/nvme0n1";
  boot.loader.grub.useOSProber = true;

  networking.hostName = "nixos"; # Define your hostname.
  # networking.wireless.enable = true;  # Enables wireless support via wpa_supplicant.

  # Configure network proxy if necessary
  # networking.proxy.default = "http://user:password@proxy:port/";
  # networking.proxy.noProxy = "127.0.0.1,localhost,internal.domain";

  # Enable networking
  networking.networkmanager.enable = true;
  

  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  # Set your time zone.
  time.timeZone = "Asia/Shanghai";

  # Select internationalisation properties.
  i18n.defaultLocale = "en_US.UTF-8";

  i18n.extraLocaleSettings = {
    LC_ADDRESS = "zh_CN.UTF-8";
    LC_IDENTIFICATION = "zh_CN.UTF-8";
    LC_MEASUREMENT = "zh_CN.UTF-8";
    LC_MONETARY = "zh_CN.UTF-8";
    LC_NAME = "zh_CN.UTF-8";
    LC_NUMERIC = "zh_CN.UTF-8";
    LC_PAPER = "zh_CN.UTF-8";
    LC_TELEPHONE = "zh_CN.UTF-8";
    LC_TIME = "zh_CN.UTF-8";
  };

  # Enable the X11 windowing system.
  services.xserver.enable = true;

  # Enable the GNOME Desktop Environment.
  # services.xserver.displayManager.gdm.enable = true;
  # services.xserver.desktopManager.gnome.enable = true;
  # services.xserver.desktopManager.gnome.extraGSettingsOverrides = ''
  # [org.gnome.shell]
  # enabled-extensions=['dash-to-dock@micxgx.gmail.com']
  # '';


  # Enable the KDE Plasma Desktop Environment.
  services.displayManager.sddm.enable = true;
  services.desktopManager.plasma6.enable = true;



  # Configure keymap in X11
  services.xserver.xkb = {
    layout = "us";
    variant = "";
  };

  # Enable CUPS to print documents.
  services.printing.enable = true;

  # Enable sound with pipewire.
  services.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire = {
    enable = true;
    alsa.enable = true;
    alsa.support32Bit = true;
    pulse.enable = true;
    # If you want to use JACK applications, uncomment this
    #jack.enable = true;

    # use the example session manager (no others are packaged yet so this is enabled by default,
    # no need to redefine it in your config for now)
    #media-session.enable = true;
  };

  # Enable touchpad support (enabled default in most desktopManager).
  # services.xserver.libinput.enable = true;

  users.defaultUserShell=pkgs.zsh; 

  # Define a user account. Don't forget to set a password with ‘passwd’.
  users.users.Joe = {
    isNormalUser = true;
    description = "Joe";
    shell = pkgs.zsh;
    extraGroups = [ "networkmanager" "wheel" ];
    packages = with pkgs; [
    #  thunderbird
    vim
    ];
  };

  security.sudo.extraConfig = ''
    Joe ALL=(ALL) NOPASSWD: ALL
  '';

  environment.sessionVariables = {
    GTK_IM_MODULE = "fcitx";
    QT_IM_MODULE = "fcitx";
    XMODIFIERS = "@im=fcitx";
  };


  # Install firefox.
  programs.firefox.enable = true;
  
  # Install zsh
  programs.zsh.enable = true;

  # Allow unfree packages
  nixpkgs.config.allowUnfree = true;
  nixpkgs.config.allowUnfreePredicate = pkg: builtins.elem (lib.getName pkg) [
    "wechat"
  ];
 
  # List packages installed in system profile. To search, run:
  # $ nix search wget
  environment.systemPackages = with pkgs; [
  # Do not forget to add an editor to edit configuration.nix! The Nano editor is also installed by default.
  # wget
  (import ./tabby.nix { inherit pkgs; })
  wget
  vim
  google-chrome
  remmina
  telegram-desktop
  noto-fonts
  noto-fonts-cjk-sans
  noto-fonts-emoji
  gedit
  bat
  git
  wechat
  neofetch
  oh-my-zsh
  marktext
  ];

  fonts = {
    fontconfig = {
      defaultFonts = {
        emoji = [ "Noto Color Emoji" ];
        monospace = [ "Noto Sans Mono CJK SC" "Noto Sans Mono" ];
        sansSerif = [ "Noto Sans CJK SC" "Noto Sans" ];
        serif = [ "Noto Serif CJK SC" "Noto Serif" ];
      };
    };
  };


  i18n.inputMethod = {
    enable = true;
      type = "fcitx5";
      fcitx5.addons = with pkgs; [
      fcitx5-rime
      fcitx5-chinese-addons
    ];

  };
  

  # Some programs need SUID wrappers, can be configured further or are
  # started in user sessions.
  # programs.mtr.enable = true;
  # programs.gnupg.agent = {
  #   enable = true;
  #   enableSSHSupport = true;
  # };

  # List services that you want to enable:

  # Enable the OpenSSH daemon.
  services.openssh.enable = true;
  services.openssh.settings.PermitRootLogin = "no";
  # Open ports in the firewall.
  # networking.firewall.allowedTCPPorts = [ ... ];
  # networking.firewall.allowedUDPPorts = [ ... ];
  # Or disable the firewall altogether.
  # networking.firewall.enable = false;

  # This value determines the NixOS release from which the default
  # settings for stateful data, like file locations and database versions
  # on your system were taken. It‘s perfectly fine and recommended to leave
  # this value at the release version of the first install of this system.
  # Before changing this value read the documentation for this option
  # (e.g. man configuration.nix or on https://nixos.org/nixos/options.html).
  system.stateVersion = "25.05"; # Did you read the comment?

}

```


`flake.nix`

```
{
  description = "My NixOS config with Home Manager";
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.05";
    home-manager.url = "github:nix-community/home-manager/release-25.05";
    home-manager.inputs.nixpkgs.follows = "nixpkgs";
  };
  outputs = { self, nixpkgs, home-manager, ... }:
    {
      nixosConfigurations.nixos = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [
          ./configuration.nix
          home-manager.nixosModules.home-manager
          {
            home-manager.users.Joe = import ./home.nix;
          }
        ];
      };
    };
}

```


`home.nix`

```
{ config, pkgs, ...}:

{
  home.username = "Joe";
  home.homeDirectory = "/home/Joe";
  home.stateVersion = "25.05";

  # Install zsh
  programs.zsh = {
    enable = true;

    plugins = [
      {
        name = "zsh-autosuggestions";
        src = pkgs.zsh-autosuggestions;
      }
  ];

    oh-my-zsh = {
      enable = true;
      plugins = [ "git" "z" "sudo" ];
      theme = "powerlevel10k/powerlevel10k";
      extraConfig = ''
        source ${pkgs.zsh-powerlevel10k}/share/zsh-powerlevel10k/powerlevel10k.zsh-theme
      '';
    };
    };

  home.packages = with pkgs; [
    bat
  ];
}

```
### 更新配置

```
构建并切换配置：
sudo nixos-rebuild switch --flake .#<hostname>

检查配置：
nixos-rebuild dry-run --flake .#<hostname>

更新 flake 输入：
nix flake update
```


> 以上配置文件的内容可以在这个[github repo](https://github.com/JaeGerW2016/nixos-flakes-btw) 查看

### 参考文档

[NixOS  Documents](https://nixos.org/manual/nixos/stable/)

[NixOS Wiki: Flakes](https://nixos.wiki/wiki/Flakes)

[How to install and Customize NixOS Linux (2026 Edition)](https://www.youtube.com/watch?v=lUB2rwDUm5A&t=1380s)

[What's on my Home Server 2025-NixOS Edition](https://www.youtube.com/watch?v=f-x5cB6qCzA&t=2s)
