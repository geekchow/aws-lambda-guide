# 07 — Next Steps: Exercises, Lab, Further Reading

> **Where you are:** after the walkthrough.
> **After this file you will know:** how to verify each concept yourself in a sandbox account, and where to go deeper.

---

## 1. Check your mental model (answers at the bottom)

1. A function runs 300 ms on average and receives 1,500 RPS (Requests Per Second). What concurrency does it need? What happens if a dependency slows it to 900 ms?
2. Why can't you put provisioned concurrency on `$LATEST`?
3. You set reserved concurrency = 200 and PC (Provisioned Concurrency) = 200. Traffic needs 250. What do the extra 50 requests experience — cold starts or throttles?
4. During a 10% canary with PC = 20 on the alias, the new version shows spillover even though total utilization is 40%. Why?
5. Your Init code reads `Date.now()` to compute a token expiry 5 minutes ahead. Why does this break only on PC environments?
6. You delete version 3. What number does the next `PublishVersion` produce if the latest was 9?

## 2. Hands-on lab (sandbox account, ~30 minutes, cents of cost)

```bash
FN=mm-lab
ROLE_ARN=arn:aws:iam::<account-id>:role/<lambda-basic-execution-role>

cat > index.mjs <<'EOF'
const initAt = Date.now();
const initType = process.env.AWS_LAMBDA_INITIALIZATION_TYPE;
await new Promise(r => setTimeout(r, 800));          // simulate slow Init
export const handler = async () => ({
  version: process.env.AWS_LAMBDA_FUNCTION_VERSION,
  initType, envAgeMs: Date.now() - initAt
});
EOF
zip fn.zip index.mjs

# 1. Function + two versions
aws lambda create-function --function-name $FN --runtime nodejs22.x \
  --handler index.handler --role $ROLE_ARN --zip-file fileb://fn.zip
aws lambda wait function-active-v2 --function-name $FN
aws lambda publish-version --function-name $FN            # → 1
aws lambda update-function-configuration --function-name $FN --memory-size 256
aws lambda wait function-updated-v2 --function-name $FN
aws lambda publish-version --function-name $FN            # → 2

# 2. Weighted alias: 1 primary, 2 at 50%
aws lambda create-alias --function-name $FN --name live --function-version 1 \
  --routing-config 'AdditionalVersionWeights={"2"=0.5}'
for i in $(seq 10); do
  aws lambda invoke --function-name $FN:live out.json >/dev/null && cat out.json; echo
done                                                       # observe version 1 vs 2

# 3. Cold vs warm: look for "Init Duration" in REPORT lines
aws lambda invoke --function-name $FN:2 --log-type Tail out.json \
  --query LogResult --output text | base64 --decode | grep REPORT

# 4. Provisioned concurrency on the alias (point it at one version first)
aws lambda update-alias --function-name $FN --name live --function-version 2 --routing-config '{}'
aws lambda put-provisioned-concurrency-config --function-name $FN --qualifier live \
  --provisioned-concurrent-executions 2
aws lambda get-provisioned-concurrency-config --function-name $FN --qualifier live   # wait for READY
aws lambda invoke --function-name $FN:live out.json && cat out.json   # initType=provisioned-concurrency, large envAgeMs

# 5. Reserved concurrency = 0 is an off switch
aws lambda put-function-concurrency --function-name $FN --reserved-concurrent-executions 0
aws lambda invoke --function-name $FN:live out.json        # → TooManyRequestsException

# Clean up (PC bills while configured!)
aws lambda delete-provisioned-concurrency-config --function-name $FN --qualifier live
aws lambda delete-function --function-name $FN
```

What to observe: version-qualified invocations are stable; the alias splits traffic randomly; `Init Duration` appears only on cold starts; PC environments report `initType=provisioned-concurrency` and an `envAgeMs` far larger than the request time (Init happened earlier); RC = 0 throttles instantly.

## 3. Further reading

| Topic | Source |
|---|---|
| Execution environment lifecycle | AWS Lambda Developer Guide — "Understanding the Lambda execution environment lifecycle" |
| Scaling & concurrency | Developer Guide — "Lambda scaling behavior", "Configuring reserved concurrency", "Configuring provisioned concurrency" |
| Versions & aliases | Developer Guide — "Manage Lambda function versions", "Create an alias", "Implement canary deployments using a weighted alias" |
| Internals | re:Invent talks "A closer look at AWS Lambda" (SVS404/SVS407 series); paper *Firecracker: Lightweight Virtualization for Serverless Applications* (NSDI (Networked Systems Design and Implementation) 2020); github.com/firecracker-microvm/firecracker |
| Cold-start alternatives | Developer Guide — "Improving startup performance with Lambda SnapStart" |
| Deploy automation | AWS SAM (Serverless Application Model) `AutoPublishAlias` + `DeploymentPreference`; CodeDeploy deployment configurations for Lambda |

## Answers

1. 1,500 × 0.3 = **450**. At 900 ms: **1,350** — triple, possibly throttling at the account quota.
2. PC pre-initializes environments from a fixed artifact; `$LATEST` can change at any moment, so the pool could be stale.
3. **Throttles** (429). RC caps total concurrency at 200 and PC uses all of it; there is no on-demand headroom left to spill into.
4. PC is split by weight (≈ 2 envs for the 10% version) and routing is probabilistic; a short burst to the new version exceeds its 2 environments.
5. PC Init may run minutes or hours before the first request, so the "5 minutes ahead" expiry has already passed.
6. **10** — numbers are never reused.

---

**Back to** [00-overview.md](00-overview.md) · [03-concept-map.md](03-concept-map.md)
