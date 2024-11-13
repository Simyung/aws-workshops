# AZ Failure Experiment Setup

## Scaling Instances <a href="#scaling-instances" id="scaling-instances"></a>

가용 영역(AZ) 장애의 전체 영향을 확인하기 위해 먼저 AZ당 두 개의 인스턴스로 확장하고 포드 수를 9개까지 늘려보겠습니다:

```bash
~$ export ASG_NAME=$(aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='eks-workshop']].AutoScalingGroupName" --output text)
~$ aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name $ASG_NAME \
    --desired-capacity 6 \
    --min-size 6 \
    --max-size 6
~$ sleep 60
~$ kubectl scale deployment ui --replicas=9 -n ui
~$ timeout 10s ~/$SCRIPT_DIR/get-pods-by-az.sh | head -n 30
 
------us-west-2a------
  ip-10-42-100-4.us-west-2.compute.internal:
       ui-6dfb84cf67-xbbj4   0/1   ContainerCreating   0     1s
  ip-10-42-106-250.us-west-2.compute.internal:
       ui-6dfb84cf67-4fjhh   1/1   Running   0     5m20s
       ui-6dfb84cf67-gkrtn   1/1   Running   0     5m19s
 
------us-west-2b------
  ip-10-42-139-198.us-west-2.compute.internal:
       ui-6dfb84cf67-7rfkf   0/1   ContainerCreating   0     4s
  ip-10-42-141-133.us-west-2.compute.internal:
       ui-6dfb84cf67-7qnkz   1/1   Running   0     5m23s
       ui-6dfb84cf67-n58b9   1/1   Running   0     5m23s
 
------us-west-2c------
  ip-10-42-175-140.us-west-2.compute.internal:
       ui-6dfb84cf67-8xfk8   0/1   ContainerCreating   0     8s
       ui-6dfb84cf67-s55nb   0/1   ContainerCreating   0     8s
  ip-10-42-179-59.us-west-2.compute.internal:
       ui-6dfb84cf67-lvdc2   1/1   Running   0     5m26s
```



## Setting up a Synthetic Canary <a href="#setting-up-a-synthetic-canary" id="setting-up-a-synthetic-canary"></a>

실험을 시작하기 전에 하트비트 모니터링을 위한 Synthetic Canary를 설정하세요:

1. 먼저 카나리 아티팩트를 위한 S3 버킷을 생성합니다:

```bash
~$ export BUCKET_NAME="eks-workshop-canary-artifacts-$(date +%s)"
~$ aws s3 mb s3://$BUCKET_NAME --region $AWS_REGION
 
make_bucket: eks-workshop-canary-artifacts-1724131402
```

2. 블루프린트를 생성합니다:

{% code title="~/environment/eks-workshop/modules/observability/resiliency/scripts/create-blueprint.sh" %}
```bash
#!/bin/bash

# Get Ingress URL
INGRESS_URL=$(kubectl get ingress -n ui -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')

# Create the required directory structure
mkdir -p nodejs/node_modules

# Create the Node.js canary script with heartbeat blueprint
cat << EOF > nodejs/node_modules/canary.js
const { URL } = require('url');
const synthetics = require('Synthetics');
const log = require('SyntheticsLogger');
const syntheticsConfiguration = synthetics.getConfiguration();
const syntheticsLogHelper = require('SyntheticsLogHelper');

const loadBlueprint = async function () {
    const urls = ['http://${INGRESS_URL}'];

    // Set screenshot option
    const takeScreenshot = true;

    // Configure synthetics settings
    syntheticsConfiguration.disableStepScreenshots();
    syntheticsConfiguration.setConfig({
       continueOnStepFailure: true,
       includeRequestHeaders: true,
       includeResponseHeaders: true,
       restrictedHeaders: [],
       restrictedUrlParameters: []
    });

    let page = await synthetics.getPage();

    for (const url of urls) {
        await loadUrl(page, url, takeScreenshot);
    }
};

// Reset the page in-between
const resetPage = async function(page) {
    try {
        await page.goto('about:blank', {waitUntil: ['load', 'networkidle0'], timeout: 30000});
    } catch (e) {
        synthetics.addExecutionError('Unable to open a blank page. ', e);
    }
};

const loadUrl = async function (page, url, takeScreenshot) {
    let stepName = null;
    let domcontentloaded = false;

    try {
        stepName = new URL(url).hostname;
    } catch (e) {
        const errorString = \`Error parsing url: \${url}. \${e}\`;
        log.error(errorString);
        throw e;
    }

    await synthetics.executeStep(stepName, async function () {
        const sanitizedUrl = syntheticsLogHelper.getSanitizedUrl(url);

        const response = await page.goto(url, { waitUntil: ['domcontentloaded'], timeout: 30000});
        if (response) {
            domcontentloaded = true;
            const status = response.status();
            const statusText = response.statusText();

            logResponseString = \`Response from url: \${sanitizedUrl}  Status: \${status}  Status Text: \${statusText}\`;

            if (response.status() < 200 || response.status() > 299) {
                throw new Error(\`Failed to load url: \${sanitizedUrl} \${response.status()} \${response.statusText()}\`);
            }
        } else {
            const logNoResponseString = \`No response returned for url: \${sanitizedUrl}\`;
            log.error(logNoResponseString);
            throw new Error(logNoResponseString);
        }
    });

    // Wait for 15 seconds to let page load fully before taking screenshot.
    if (domcontentloaded && takeScreenshot) {
        await new Promise(r => setTimeout(r, 15000));
        await synthetics.takeScreenshot(stepName, 'loaded');
    }
    
    // Reset page
    await resetPage(page);
};

exports.handler = async () => {
    return await loadBlueprint();
};
EOF

# Zip the Node.js script
python3 - << EOL
import zipfile
with zipfile.ZipFile('canary.zip', 'w') as zipf:
    zipf.write('nodejs/node_modules/canary.js', arcname='nodejs/node_modules/canary.js')
EOL

# Ensure BUCKET_NAME is set
if [ -z "$BUCKET_NAME" ]; then
    echo "Error: BUCKET_NAME environment variable is not set."
    exit 1
fi

# Upload the zipped canary script to S3
aws s3 cp canary.zip "s3://${BUCKET_NAME}/canary-scripts/canary.zip"

echo "Canary script has been zipped and uploaded to s3://${BUCKET_NAME}/canary-scripts/canary.zip"
echo "The script is configured to check the URL: http://${INGRESS_URL}"

```
{% endcode %}

이 카나리 블루프린트를 버킷에 넣습니다:

```bash
~$ ~/$SCRIPT_DIR/create-blueprint.sh
 
upload: ./canary.zip to s3://eks-workshop-canary-artifacts-1724131402/canary-scripts/canary.zip
Canary script has been zipped and uploaded to s3://eks-workshop-canary-artifacts-1724131402/canary-scripts/canary.zip
The script is configured to check the URL: http://k8s-ui-ui-5ddc3ba496-721427594.us-west-2.elb.amazonaws.com
```

3. Cloudwatch 경보와 함께 synthetic canary를 생성합니다:

```bash
~$ aws synthetics create-canary \
--name eks-workshop-canary \
--artifact-s3-location "s3://$BUCKET_NAME/canary-artifacts/" \
--execution-role-arn $CANARY_ROLE_ARN \
--runtime-version syn-nodejs-puppeteer-9.0 \
--schedule "Expression=rate(1 minute)" \
--code "Handler=canary.handler,S3Bucket=$BUCKET_NAME,S3Key=canary-scripts/canary.zip" \
--region $AWS_REGION
~$ sleep 40
~$ aws synthetics describe-canaries --name eks-workshop-canary --region $AWS_REGION
~$ aws synthetics start-canary --name eks-workshop-canary --region $AWS_REGION
~$ aws cloudwatch put-metric-alarm \
--alarm-name "eks-workshop-canary-alarm" \
--metric-name SuccessPercent \
--namespace CloudWatchSynthetics \
--statistic Average \
--period 60 \
--threshold 95 \
--comparison-operator LessThanThreshold \
--dimensions Name=CanaryName,Value=eks-workshop-canary \
--evaluation-periods 1 \
--alarm-description "Alarm when Canary success rate drops below 95%" \
--unit Percent \
--region $AWS_REGION
```

이렇게 하면 매분마다 애플리케이션의 상태를 확인하는 카나리와 성공 비율이 95% 미만으로 떨어질 경우 트리거되는 CloudWatch 경보가 설정됩니다.

이러한 단계를 완료하면 애플리케이션이 이제 AZ의 두 인스턴스로 확장되고 앞으로 있을 AZ 장애 시뮬레이션 실험에 필요한 모니터링이 설정됩니다.

