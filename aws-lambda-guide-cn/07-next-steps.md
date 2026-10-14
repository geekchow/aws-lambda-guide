# 07 —— 后续：自测题、动手实验、延伸阅读

> **你在这里：** 完整走查之后。
> **读完本文你将知道：** 如何在沙箱账号中亲手验证每个概念，以及去哪里深入学习。

---

## 1. 检验你的心智模型（答案在文末）

1. 某函数平均运行 300 毫秒，接收 1,500 RPS（Requests Per Second，每秒请求数）。它需要多少并发？如果某个依赖把它拖慢到 900 毫秒会怎样？
2. 为什么不能在 `$LATEST` 上配置预置并发？
3. 你设置预留并发 = 200、PC（Provisioned Concurrency，预置并发）= 200。流量需要 250。多出的 50 个请求经历的是冷启动还是限流？
4. 在别名 PC = 20 的 10% 金丝雀期间，新版本出现溢出，而总利用率只有 40%。为什么？
5. 你的 Init 代码读取 `Date.now()` 来计算 5 分钟后过期的令牌。为什么这只在 PC 环境中出问题？
6. 你删除了版本 3。如果最新版本是 9，下一次 `PublishVersion` 会产生几号？

## 2. 动手实验（沙箱账号，约 30 分钟，花费几美分）

```bash
FN=mm-lab
ROLE_ARN=arn:aws:iam::<account-id>:role/<lambda-basic-execution-role>

cat > index.mjs <<'EOF'
const initAt = Date.now();
const initType = process.env.AWS_LAMBDA_INITIALIZATION_TYPE;
await new Promise(r => setTimeout(r, 800));          // 模拟缓慢的 Init
export const handler = async () => ({
  version: process.env.AWS_LAMBDA_FUNCTION_VERSION,
  initType, envAgeMs: Date.now() - initAt
});
EOF
zip fn.zip index.mjs

# 1. 函数 + 两个版本
aws lambda create-function --function-name $FN --runtime nodejs22.x \
  --handler index.handler --role $ROLE_ARN --zip-file fileb://fn.zip
aws lambda wait function-active-v2 --function-name $FN
aws lambda publish-version --function-name $FN            # → 1
aws lambda update-function-configuration --function-name $FN --memory-size 256
aws lambda wait function-updated-v2 --function-name $FN
aws lambda publish-version --function-name $FN            # → 2

# 2. 加权别名：主版本 1，版本 2 占 50%
aws lambda create-alias --function-name $FN --name live --function-version 1 \
  --routing-config 'AdditionalVersionWeights={"2"=0.5}'
for i in $(seq 10); do
  aws lambda invoke --function-name $FN:live out.json >/dev/null && cat out.json; echo
done                                                       # 观察版本 1 与 2 的分布

# 3. 冷与热：在 REPORT 行中寻找 "Init Duration"
aws lambda invoke --function-name $FN:2 --log-type Tail out.json \
  --query LogResult --output text | base64 --decode | grep REPORT

# 4. 在别名上配置预置并发（先让它只指向一个版本）
aws lambda update-alias --function-name $FN --name live --function-version 2 --routing-config '{}'
aws lambda put-provisioned-concurrency-config --function-name $FN --qualifier live \
  --provisioned-concurrent-executions 2
aws lambda get-provisioned-concurrency-config --function-name $FN --qualifier live   # 等待 READY
aws lambda invoke --function-name $FN:live out.json && cat out.json   # initType=provisioned-concurrency，envAgeMs 很大

# 5. 预留并发 = 0 就是关闭开关
aws lambda put-function-concurrency --function-name $FN --reserved-concurrent-executions 0
aws lambda invoke --function-name $FN:live out.json        # → TooManyRequestsException

# 清理（PC 只要配置着就在计费！）
aws lambda delete-provisioned-concurrency-config --function-name $FN --qualifier live
aws lambda delete-function --function-name $FN
```

观察要点：版本限定的调用是稳定的；别名随机分流；`Init Duration` 只出现在冷启动中；PC 环境报告 `initType=provisioned-concurrency`，且 `envAgeMs` 远大于请求耗时（Init 早已完成）；RC（Reserved Concurrency，预留并发）= 0 立即限流。

## 3. 延伸阅读

| 主题 | 来源 |
|---|---|
| 执行环境生命周期 | AWS Lambda 开发者指南——"Understanding the Lambda execution environment lifecycle" |
| 扩缩与并发 | 开发者指南——"Lambda scaling behavior"、"Configuring reserved concurrency"、"Configuring provisioned concurrency" |
| 版本与别名 | 开发者指南——"Manage Lambda function versions"、"Create an alias"、"Implement canary deployments using a weighted alias" |
| 内部实现 | re:Invent 演讲 "A closer look at AWS Lambda"（SVS404/SVS407 系列）；论文 *Firecracker: Lightweight Virtualization for Serverless Applications*（NSDI（Networked Systems Design and Implementation，网络系统设计与实现研讨会）2020）；github.com/firecracker-microvm/firecracker |
| 冷启动替代方案 | 开发者指南——"Improving startup performance with Lambda SnapStart" |
| 部署自动化 | AWS SAM（Serverless Application Model，无服务器应用模型）`AutoPublishAlias` + `DeploymentPreference`；CodeDeploy 的 Lambda 部署配置 |

## 答案

1. 1,500 × 0.3 = **450**。900 毫秒时：**1,350**——变为三倍，可能在账号限额处被限流。
2. PC 从固定制品预初始化环境；`$LATEST` 随时可能变化，预热池会因此过时。
3. **限流**（429）。RC 把总并发限制在 200，而 PC 用掉了全部；没有剩余的按需余量可供溢出。
4. PC 按权重拆分（10% 的版本约 2 个环境），且路由是概率性的；新版本的一阵短时突发就超过了它的 2 个环境。
5. PC 的 Init 可能在首个请求前数分钟或数小时运行，所以"5 分钟后"的过期时间早已过去。
6. **10**——编号永不复用。

---

**返回** [00-overview.md](00-overview.md) · [03-concept-map.md](03-concept-map.md)
