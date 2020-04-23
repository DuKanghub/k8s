<font size=4>

# kubectl命令自动补全方法
正常情况下，系统只能自动补全`kubectl`这个命令，但`kubectl get`这样的就没法补全了。需进行以下设置即可。

`centos`

```bash
yum install -y epel-release bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
# 生效命令方式二
echo "source <(kubectl completion bash)" >> /etc/profile
```

`ubuntu`

```bash
apt install bash-completion
# 生效命令方式一
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
# 生效命令方式二
echo "source <(kubectl completion bash)" >> /etc/profile
```

`mac`

```bash
brew install bash-completion
source $(brew --prefix)/etc/bash_completion
source <(kubectl completion bash)
```

