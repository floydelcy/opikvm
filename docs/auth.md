# 默认密码以及2FA验证

!!! warning "PiKVM默认密码:"

    * **Linux admin** (SSH, console, 等.): 用户名 `root`和`orangepi`, 密码都为 `orangepi`。
    * **PiKVM Web登录页面** ([API](https://github.com/pikvm/pikvm/blob/master/docs/api.md), [VNC](https://github.com/pikvm/pikvm/blob/master/docs/vnc.md)...): 用户名 `admin`, 密码 `admin`, 默认未启用2FA code。

    **这是两个独立的实体，拥有独立的账户。**

!!! note "还有另一个特殊的Linux用户:`kvmd-webterm`"
    它不能用于登录或远程访问PiKVM OS，并且在操作系统中没有特权权限。
    密码访问和`sudo`被禁用。它仅用于启动Web Terminal终端。
    设置这些限制是出于安全原因。

*参考文档更改[VNCAuth密钥](https://github.com/pikvm/pikvm/blob/master/docs/vnc.md)和[IPMI密码](https://github.com/pikvm/pikvm/blob/master/docs/ipmi.md)，这些服务在默认情况下是禁用的*。

-----
## Web Terminal终端中的root访问权限

如上所述，Web Terminal终端使用`kvmd-webterm`账户运行，禁用`sudo`和密码访问。
但是，大多数PiKVM管理命令都需要`root`访问权限。要在Web Terminal终端中获取权限，请键入`su -`，然后输入`root`密码：

```console
[kvmd-webterm@orangepizero3:~$] su -
...
[root@orangepizero3:~#]
```

??? example "如何禁用Web Terminal终端"

    某些情况下，PiKVM设备的实际所有者和被允许使用它的用户是不同的人。
    因此您可以使用以下命令禁用Web中的控制台访问，即禁用Web Terminal终端：

    ```console
    [kvmd-webterm@orangepizero3:~$] su -
    [root@orangepizero3:~#] systemctl disable --now kvmd-webterm
    [root@orangepizero3:~#]
    ```

    对于您自己对PiKVM OS的SSH访问将仍然存在。


-----
## 更改Linux admin密码

此密码用于PiKVM OS主机SSH/串口登录

```console
[root@orangepizero3:~#] passwd
New password:
Retype new password:
passwd: password updated successfully
[root@orangepizero3:~#]
```


-----
## 更改PiKVM Web密码

此密码用于Web UI登录，用于访问[API](https://github.com/pikvm/pikvm/blob/master/docs/api.md)、[VNC](https://github.com/pikvm/pikvm/blob/master/docs/vnc.md)(如果启用)
以及其他与操作系统shell无关的功能。

默认情况下，配置了类似于Apache Server的身份验证方法：用户和密码
以加密方式存储在`/etc/kvmd/htpasswd`文件中。为了管理它们，有一个实用程序`kvmd-htpasswd`。

```console
[root@orangepizero3:~#]kvmd-htpasswd set admin   #修改web密码
Password:
Repeat:
[root@orangepizero3:~#]
```

注意，`admin`是默认用户的名称。你可以创建多个不同的用户，使用不同的密码访问Web UI，但请记住，它们都具有相同的权限：

```console
[root@orangepizero3:~#]kvmd-htpasswd set <user> # 添加新用户
[root@orangepizero3:~#]kvmd-htpasswd list # 显示所有web ui账户
[root@orangepizero3:~#]kvmd-htpasswd del <user> # 移除一个账户
```

目前无法为不同的PiKVM Web用户创建任何ACL。

-----
## 2FA双因素验证(Two-factor authentication)

这是一种强力保护PiKVM的新方法，从`KVM >= 3.196`开始可用。
如果您需要将PiKVM暴露在互联网中，强烈建议启用它。

!!! warning
    使用2FA排除了使用[IPMI](https://github.com/pikvm/pikvm/blob/master/docs/ipmi.md)和[VNC with vncauth](https://github.com/pikvm/pikvm/blob/master/docs/vnc.md)(默认情况下均禁用)的可能性。
    它还略微影响了[API](https://github.com/pikvm/pikvm/blob/master/docs/api.md)和带有用户/密码的常规VNC的使用，下文会介绍。

    请注意，2FA不涉及`root`用户的Linux操作系统访问，因此请注意设置强密码用于SSH访问(或设置[密钥访问](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server))。

??? example "在PiKVM上启用2FA验证"

    1. **确保NTP服务正在运行(`timedatectl`命令)，否则您将无法访问**。时区无关紧要。

    2. 将 **Google Authenticator** 应用安装到您的移动设备
        ([iOS](https://apps.apple.com/us/app/google-authenticator/id388497605),
        [Android](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2))。 用来生成一次性访问代码。

    3. 在PiKVM上为一次性密码创建密钥，此命令会生成一个二维码：
       ```console
       [root@orangepizero3:~#]
       [root@orangepizero3:~#] kvmd-totp init
       [root@orangepizero3:~#]
       ```

    4. 运行Google Authenticator并扫描二维码。

    5. 现在在PiKVM登录页面上，您需要在“2FA验证”后输入6位2FA验证码进行登录。

    *如果你无法使用google服务，你同样可以使用*
    ***[Microsoft Authenticator](https://www.microsoft.com/zh-cn/security/mobile-authenticator-app)***
    *来实现2FA功能，步骤与上述一致*

所有Web UI用户都需要在登录时输入一次性密码。
换言之，**密钥对所有用户都是相同的**。

!!! note
    对于开启了2FA的API或VNC身份验证，您需要将一次性密码附加到密码中，不带空格。
    例如：密码是“foobar”，一次性密码是“123456”，那么你需要使用“foobar123456”作为密码。

要查看密钥的当前二维码，请使用命令`kvmd-totp show`。

要禁用2FA并删除密钥，请使用命令`kvmd-totp del`。