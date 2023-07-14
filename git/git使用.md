## 设置用户名和邮箱

```shell
git config --global user.name <your username for Github>
git config --global user.email <your email address for Github>
```

## 从命令行操作需要登录时

1. `git config --global credential.helper store`执行这条命令，之后的操作就不用每次都输入账户密码了
2. 账户为邮箱，密码为token

