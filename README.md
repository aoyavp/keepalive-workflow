# 解决GitHub workflow 60天限制问题，推荐API 保活方案
推荐配置：gautamkrishnar/keepalive-workflow
在你的工作流文件中添加以下内容。建议把它放在独立的 job 中，而不是混在业务逻辑里，这样更安全
```
jobs:
  # 你的原有任务...
  main-job:
    name: Main Job
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ... 你原有的代码步骤

  # 新增的保活任务
  keepalive-job:
    name: Keepalive Workflow
    runs-on: ubuntu-latest
    permissions:
      actions: write   # 这是必须的，允许 API 调用
    steps:
      - uses: gautamkrishnar/keepalive-workflow@v2```
```

## 关键配置说明
1、权限 (actions: write)：必须在保活 job 中显式声明，否则 API 调用会失败。

2、独立 Job：即使主任务失败，保活任务也应尽量运行。可以用 needs: main-job 和 if: always() 来控制依赖关系。

3、时间阈值：默认会在仓库 45 天无活动时触发 API 调用，这个时间比 GitHub 的 60 天限制要安全很多。

## 备选方案
如果上面的 Action 出现问题（例如因仓库无活动被 GitHub 误停用），可以换用 liskin/gh-workflow-keepalive。用法完全一样，同样需要 actions: write 权限，且不会产生 dummy commit

如果项目 job 已经声明了 actions: write 权限，保活所需的 API 调用权限已满足，无需改动其他配置，直接在你的 checkin job 末尾追加一个保活步骤即可。
代码如下 ：
```
      # 新增：保活步骤，防止 60 天无活动导致 schedule 被自动禁用
      - name: Keepalive Workflow
        if: always()
        uses: gautamkrishnar/keepalive-workflow@v2
        with:
          gh_token: ${{ github.token }}
```
## 几点说明
1、if: always()：保证即使前面的 Run checkin 或 clear old run jobs 失败，保活步骤依然会执行。这点很重要，否则一旦签到脚本报错，保活就失效了。

2、gh_token 参数：显式传入 github.token，确保 Action 有权限调用 API。虽然该 Action 默认会读取 GITHUB_TOKEN，但显式声明更清晰可靠。

3、clear old run jobs 的潜在冲突：你这段脚本会删除除最新一次之外的所有运行记录。注意保活 Action 本身不会产生新的 workflow run，所以不会和这个清理逻辑冲突。但如果你后续把保活逻辑改成了「空提交」方案，就会产生新的 run，需要留意。

4、保活触发时机：该 Action 默认在仓库 45 天无活动时才调用 API 保活，而不是每次运行都调用，所以你不用担心它每天产生额外操作。

## 备选写法（不依赖第三方 Action）
如果你不想引入外部 Action，也可以用一行 curl 直接调用 API：
```
      - name: Keepalive Workflow
        if: always()
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh api -X PUT /repos/${{ github.repository }}/actions/workflows/${{ github.workflow }}/enable || true
```
不过这种写法只能「重新启用」已停用的工作流，不能预防 60 天倒计时归零。真正有效的保活还是要靠定期制造仓库活动，所以更推荐上面的 keepalive-workflow 方案。
