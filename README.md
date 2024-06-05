<!--
 * @Author: wzdnzd
 * @Date: 2022-03-06 14:51:29
 * @Description:
 * Copyright (c) 2022 by wzdnzd, All Rights Reserved.
-->

# 代理池

## 功能

> 打造免费代理池，爬一切可爬节点。拥有插件系统，如果目标网站特殊，现有功能未能覆盖，可针对性地通过插件实现

## 使用方法

### 本地使用

```shell
pip3 install pyYAML tqdm
python3 -u subscribe/collect.py -s
```

然后使用你的`clash`客户端导入`data/clash.yml`即可

### Gist 使用

1. 创建一个`GIST`
2. 创建`PAT`

   - 登录到您的 GitHub 帐户。
   - 点击页面右上角的您的头像，然后选择 “Settings”（设置）。
   - 在左侧导航栏中，选择 “Developer settings”（开发者设置）。
   - 在 “Developer settings” 页面上，选择 “Personal access tokens”（个人访问令牌）。
   - 点击 “Generate new token”（生成新令牌）按钮。
   - 在 “Note”（注释）字段中输入一个描述令牌用途的名称。
   - 在 “Select scopes”（选择范围）部分，选择您想要该令牌具有的权限（例如，读取存储库、写入存储库等）。
   - 单击页面底部的 “Generate token”（生成令牌）按钮。

3. 创建`secrets`变量

   创建`GIST`和`PAT`变量，以下是创建过程。

   - 打开项目的设置页面：

     进入 GitHub 仓库主页。
     点击仓库名称右侧的 Settings（设置）选项卡。

   - 导航到 Secrets 页面：

     在设置页面的左侧边栏中，点击 Secrets and variables。
     然后选择 Actions，进入 Actions secrets 页面。

   - 添加新 secret：

     点击 New repository secret 按钮。
     在 Name 字段中输入 secret 的名称（例如：MY_SECRET）。
     在 Secret 字段中输入 secret 的值。
     点击 Add secret 按钮保存。

4. 创建 Action

   ```yaml
   name: Get Subscription Weekly

   on:
   schedule:
       - cron: "0 16 * * 6" # 北京时间每周日凌晨执行
   workflow_dispatch: # 允许手动触发

   permissions:
   contents: write

   jobs:
   execute-script:
       runs-on: ubuntu-latest

       steps:
       - uses: actions/checkout@v3

       - name: Set up Python 3.10
           uses: actions/setup-python@v3
           with:
           python-version: "3.10"

       - name: Install dependencies
           run: |
           pip3 install pyYAML requests
           if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

       - name: Collect Subscribe
           id: collect
           run: |
           python -u subscribe/collect.py

       - name: Create/Update Gist
           env:
           GH_TOKEN: ${{ secrets.PAT}}
           run: |
           cd data
           gh gist edit ${{ secrets.GIST_ID }} --add "clash.yaml"
           gh gist edit ${{ secrets.GIST_ID }} --add "subscribes.txt"
           gh gist edit ${{ secrets.GIST_ID }} --add "domains.txt"
           gh gist edit ${{ secrets.GIST_ID }} --add "valid-domains.txt"
   ```

5. 制作成订阅链接

   ```shell
   https://sub.xeton.dev/sub?target=clash&new_name=true&url=你的gist订阅链接&insert=false&config=https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/config/ACL4SSR_Online_Full.ini

   ```

## 免责申明

- 本项目仅用作学习爬虫技术，请勿滥用，不要通过此工具做任何违法乱纪或有损国家利益之事
- 禁止使用该项目进行任何盈利活动，对一切非法使用所产生的后果，本人概不负责
