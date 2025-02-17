# AWS Batch Jobs as EventBridge Targets<a name="batch-cwe-target"></a>

Amazon EventBridge delivers a near real\-time stream of system events that describe changes in Amazon Web Services resources\. AWS Batch jobs are available as EventBridge targets\. Using simple rules, you can match events and submit AWS Batch jobs in response to them\. For more information, see [What is EventBridge?](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) in the *Amazon EventBridge User Guide*\.

You can also use EventBridge to schedule automated actions that are invoked at certain times using cron or rate expressions\. For more information, see [Creating an Amazon EventBridge rule that runs on a schedule](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html) in the *Amazon EventBridge User Guide*\.

Common use cases for AWS Batch jobs as an EventBridge target include the following use cases:
+ A scheduled job is created to occur at regular time intervals\. For example, a cron job occurs only during low\-usage hours when Amazon EC2 Spot Instances are less expensive\.
+ An AWS Batch job runs in response to an API operation that's logged in CloudTrail\. For example, a job is submitted whenever an object is uploaded to a specified Amazon S3 bucket\. Each time this happens, the EventBridge input transformer passes the bucket and key name of the object to AWS Batch parameters\.
**Note**  
In this scenario, all of related AWS resources must be in the same Region\. This includes resources such as the Amazon S3 bucket, EventBridge rule, and CloudTrail logs\.

Before you can submit AWS Batch jobs with EventBridge rules and targets, the EventBridge service needs several permissions to run AWS Batch jobs on your behalf\. When you create a rule in the EventBridge console that specifies an AWS Batch job as a target, you're provided with an opportunity to create this role\. For more information about the required service principal and IAM permissions for this role, see [EventBridge IAM role](CWE_IAM_role.md)\.

## Creating a Scheduled AWS Batch Job<a name="scheduled-batch-job"></a>

The procedure below shows how to create a scheduled AWS Batch job and the required EventBridge IAM role\.

**To create a scheduled AWS Batch job with EventBridge**

1. Open the Amazon EventBridge console at [https://console\.aws\.amazon\.com/events/](https://console.aws.amazon.com/events/)\.

1. Using the following values, create an EventBridge rule that schedules an AWS Batch job:
   + For **Rule type**, choose **Schedule**\.
   + For **Schedule pattern**, do one of the following:
     + To use a cron expression to define the schedule, choose **A fine\-grained schedule that runs at a specific time, such as 8:00 a\.m\. PST on the first Monday of every month** and enter the cron expression\.
     + To use a rate expression to define the schedule, choose **A schedule that runs at a regular rate, such as every 10 minutes** and enter the rate expression\.
   + For **Target types**, choose **AWS service**\.
   + For **Select a target**, choose **Batch job queue** and fill in the following fields appropriately:
     + **Job queue:** Enter the Amazon Resource Name \(ARN\) of the job queue to schedule your job in\.
     + **Job definition:** Enter the name and revision or full ARN of the job definition to use for your job\.
     + **Job name:** Enter a name for your job\.
     + **Array size:** \(Optional\) Enter an array size for your job to run more than one copy\. For more information, see [Array jobs](array_jobs.md)\.
     + **Job attempts:** \(Optional\) Enter the number of times to retry your job if it fails\. For more information, see [Automated job retries](job_retries.md)\.
   + For **Batch job queue** target types, EventBridge needs permission to send events to the target\. EventBridge can create the IAM role needed for your rule to run\. Do one of these things:
     + To create an IAM role automatically, choose **Create a new role for this specific resource**\.
     + To use an IAM role that you created before, choose **Use existing role**\.

     For more information, see [EventBridge IAM role](CWE_IAM_role.md)\.

   For details about creating rules, see [Creating an Amazon EventBridge rule that runs on a schedule](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html) in the *Amazon EventBridge User Guide*\.

## Passing Event Information to an AWS Batch Target using the EventBridge Input Transformer<a name="cwe-input-transformer"></a>

You can use the EventBridge input transformer to pass event information to AWS Batch in a job submission\. This can be especially valuable if you invoke jobs as a result of other AWS event information\. One example is an object upload to an Amazon S3 bucket\. You can also use a job definition with parameter substitution values in the container's command\. The EventBridge input transformer can provide the parameter values based on the event data\. For example, the following job definition expects to see parameter values called *S3bucket* and *S3key*\.

```
{
  "jobDefinitionName": "echo-parameters",
  "containerProperties": {
    "image": "busybox",
    "resourceRequirements": [
        {
            "type": "MEMORY",
            "value": "2000"
        },
        {
            "type": "VCPU",
            "value": "2"
        }
    ],
    "command": [
      "echo",
      "Ref::S3bucket",
      "Ref::S3key"
    ]
  }
}
```

Then, you simply create an AWS Batch event target that parses information from the event that starts it and transforms it into a `parameters` object\. When the job runs, the parameters from the trigger event are passed to the command of the job container\.

**Note**  
In this scenario, all of the AWS resources \(such as Amazon S3 buckets, EventBridge rules, and CloudTrail logs\) must be in the same Region\.

**To create an AWS Batch target that uses the input transformer**

1. Open the Amazon EventBridge console at [https://console\.aws\.amazon\.com/events/](https://console.aws.amazon.com/events/)\.

1. In the navigation pane, choose **Rules**\.

1. Choose **Create rule**\.

1. Enter a name and description for the rule\.

   A rule can't have the same name as another rule in the same Region and on the same event bus\.

1. For **Event bus**, choose the event bus that you want to associate with this rule\. If you want this rule to match events that come from your account, select **AWS default event bus**\. When an AWS service in your account emits an event, it always goes to your account's default event bus\.

1. For **Rule type**, choose **Rule with an event pattern**\.

1. Choose **Next**\.

1. For **Event source**, choose **AWS events**\.

1. For **Event pattern**, construct the rule as desired to match your application needs\.

1. Choose **Next**\.

1. For **Target types**, choose **AWS service**\.

1. For **Select a target**, choose **Batch job queue** and then specify the job queue, job definition, and job name to use for the jobs that are invoked by this rule\.

1. Choose an existing EventBridge IAM role to use for your job, or **Create a new role for this specific resource** to create a new one\. For more information, see [EventBridge IAM role](CWE_IAM_role.md)\.

1. In the **Additional settings** section, for **Configure target input**, choose **Input Transformer**\.

1. Choose **Configure input transformer**\.

1. In the **Target input transformer** section, for **Input path**, specify the values to parse from the triggering event\. For example, to parse the bucket and key name from an Amazon S3 event, use the following JSON\.

   ```
   {
     "S3BucketValue":"$.detail.bucket.name",
     "S3KeyValue":"$.detail.object.key"
   }
   ```

1. For **Template**, create the `Parameters` structure to pass to the AWS Batch job\. These parameters are substituted for the *Ref::S3bucket* and *Ref::S3key* placeholders in the command of the job container when the job runs\.

   ```
   {
    "Parameters" :
     {
      "S3bucket": <S3BucketValue>,
      "S3key": <S3KeyValue>
     }
   }
   ```

   You can also update the [ContainerOverrides](https://docs.aws.amazon.com/batch/latest/APIReference/API_ContainerOverrides.html) structure to pass to update commands, environment variables, and other settings\.

   ```
   {
    "Parameters" :
     {
      "S3bucket": <S3BucketValue>
     },
    "ContainerOverrides" :
     {
      "Command":
       [
        "echo",
        "Ref::S3bucket"
       ]
     }
   }
   ```
**Note**  
The names of the members of the [ContainerOverrides](https://docs.aws.amazon.com/batch/latest/APIReference/API_ContainerOverrides.html) structure must be capitalized\. For example, use `Command` and `ResourceRequirements`, not `command` or `resourceRequirements`\.

1. Choose **Confirm**\.

1. Choose **Next**\.

1. \(Optional\) Enter one or more tags for the rule\. For more information, see [Amazon EventBridge tags](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-tagging.html) in the *Amazon EventBridge User Guide*\.

1. Choose **Next**\.

1. Review the details of the rule and choose **Create rule**\.