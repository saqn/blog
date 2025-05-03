In this guide, you’ll learn how to seamlessly detach existing AWS resources from one CloudFormation stack and import them into another—**zero downtime**, **no data loss**, **configurations intact**. We’ll walk through both **AWS CLI** and **AWS Management Console** workflows using a **20 million-item DynamoDB** table as our concrete example, then generalize to other services (S3, RDS, EBS, IAM, etc.). You’ll also find links to the official AWS docs for each step, so you can dig deeper as needed ([AWS Documentation][1]).

---

## **Why Refactor Your Stacks?**

* **Break Monoliths**: Over time, a single mega-template becomes hard to manage. Splitting it improves deployment speed and team autonomy.
* **Zero-Downtime Moves**: Using **Resource Import**, you rehome resources without deleting them or incurring downtime ([AWS Documentation][1]).
* **Use Cases**:

    * Environment segmentation (VPC/IAM vs. application layers)
    * Cross-team ownership boundaries
    * Cross-account/region disaster-recovery stack rehoming

---

## **Prerequisites & Resource Support**
1. **Supported Resource Types**
    * Check the official list: “Resource type support” ([AWS Documentation][2])
2. **IAM Permissions**
    * `cloudformation:UpdateStack`
    * `cloudformation:CreateChangeSet` (with `IMPORT` type) ([AWS Documentation][3])
    * Service-specific rights (e.g., `dynamodb:DescribeTable`)
3. **Quiesce Workloads**
    * Pause writes/updates to avoid drift during import ([AWS Documentation][4])
---

## **CLI Workflow**

1. **Add `DeletionPolicy: Retain`** to the **source** template:

   ```yaml
   MyDynamoTable:
     Type: AWS::DynamoDB::Table
     DeletionPolicy: Retain
     Properties:
       TableName: prod-customer-table
       …other properties…
   ```

   This ensures the physical table isn’t deleted when removed from the stack ([AWS Documentation][5]).
2. **Deploy** the updated source stack:

   ```bash
   aws cloudformation deploy \
     --stack-name source-stack \
     --template-file source-template.yaml
   ```
3. **Remove** the `MyDynamoTable` block and **redeploy**. The table now exists unattached but intact.
4. **Copy** the original DynamoDB snippet into your **target** template—match **LogicalResourceId** and **TableName** exactly.
5. **Create** an **IMPORT** change set:

   ```bash
   aws cloudformation create-change-set \
     --stack-name target-stack \
     --change-set-name import-dynamodb \
     --change-set-type IMPORT \
     --resources-to-import file://import-resources.json \
     --template-body file://target-template.yaml
   ```

   Where `import-resources.json` is:

   ```json
   [
     {
       "ResourceType":"AWS::DynamoDB::Table",
       "LogicalResourceId":"MyDynamoTable",
       "ResourceIdentifier":{"TableName":"prod-customer-table"}
     }
   ]
   ```

   ([AWS Documentation][3])
6. **Execute** the change set:

   ```bash
   aws cloudformation execute-change-set \
     --stack-name target-stack \
     --change-set-name import-dynamodb
   ```

   The DynamoDB table is now managed by **target-stack** with no data movement ([AWS Documentation][3]).

---

## **Console UI Workflow**

### **Import into an Existing Stack**

1. Open the [CloudFormation console](https://console.aws.amazon.com/cloudformation) and select **target-stack** ([AWS Documentation][4]).
2. Choose **Stack actions ▶ Import resources into stack** ([AWS Documentation][4]).
3. **Upload** your updated `target-template.yaml` and click **Next**.
4. On **Identify resources**, select **TableName** and enter `prod-customer-table`. Click **Next**.
5. Review the change set, then click **Import resources**—execution starts immediately ([AWS Documentation][4]).
6. (Optional) Run **Detect drift** under **Stack actions** to confirm post-import consistency ([AWS Documentation][6]).

### **Create a New Stack with Existing Resources**

1. From **CloudFormation Stacks**, click **Create stack ▶ With existing resources (import resources)** ([AWS Documentation][7]).
2. Upload your template containing both new and existing resources.
3. Identify resource identifiers (e.g., BucketName, TableName), review, and click **Import resources** ([AWS Documentation][7]).

---

## **Generalizing to Other AWS Services**

| Service        | CFN Resource Type                  | Identifier Property                          |
| -------------- | ---------------------------------- | -------------------------------------------- |
| **S3 Bucket**  | `AWS::S3::Bucket`                  | `BucketName` ([AWS Documentation][2])        |
| **RDS**        | `AWS::RDS::DBInstance`/`DBCluster` | `DBInstanceIdentifier`/`DBClusterIdentifier` |
| **EBS Volume** | `AWS::EC2::Volume`                 | `VolumeId`                                   |
| **IAM Role**   | `AWS::IAM::Role`                   | `RoleName`                                   |
| **SQS Queue**  | `AWS::SQS::Queue`                  | `QueueName`                                  |
| **SNS Topic**  | `AWS::SNS::Topic`                  | `TopicName`                                  |
| **Lambda**     | `AWS::Lambda::Function`            | `FunctionName`                               |
| **OpenSearch** | `AWS::OpenSearch::Domain`          | `DomainName`                                 |

> *Full list*: [Supported resource types](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html) ([AWS Documentation][2]).

---

## **Best Practices**

* **Sandbox Dry-Runs**: Validate imports in non-prod accounts first.
* **Version Control**: Keep source/target templates in Git for auditing.
* **Single-Step Imports**: Don’t mix imports with other creates/deletes in one change set.
* **Drift Detection**: Post-import, run **Detect drift** to catch discrepancies ([AWS Documentation][6]).
* **Tight IAM**: Restrict who can modify `DeletionPolicy` or execute imports.

---

## **Conclusion**

By leveraging **`DeletionPolicy: Retain`** and CloudFormation’s **Resource Import** (via CLI or Console), you can **rehome** DynamoDB tables—as well as S3 buckets, RDS instances, EBS volumes, and more—across stacks with **zero downtime** and **no data loss**. This pattern empowers you to modularize your IaC, enforce team boundaries, and adapt your architecture as requirements evolve. For step-by-step reference, see:

* Resource Import Overview: [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import.html) ([AWS Documentation][1])
* Change Set Operations: [https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-change-set.html](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-change-set.html) ([AWS Documentation][3])
* DeletionPolicy Attribute: [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html) ([AWS Documentation][5])

[1]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import.html?utm_source=chatgpt.com "Import AWS resources into a CloudFormation stack"
[2]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html?utm_source=chatgpt.com "Resource type support - AWS CloudFormation"
[3]: https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-change-set.html?utm_source=chatgpt.com "create-change-set — AWS CLI 1.40.4 Command Reference"
[4]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-existing-stack.html?utm_source=chatgpt.com "Importing existing resources into a stack - AWS CloudFormation"
[5]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html?utm_source=chatgpt.com "DeletionPolicy attribute - AWS CloudFormation"
[6]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/detect-drift-stack.html?utm_source=chatgpt.com "Detect drift on an entire CloudFormation stack - AWS Documentation"
[7]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/import-resources-manually.html?utm_source=chatgpt.com "Import AWS resources into a CloudFormation stack manually"
