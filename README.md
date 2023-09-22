# DynamoDB Lambda Triggers

## Overview

This project focuses on setting up a DynamoDB table that stores information about babies awaiting adoption. Specifically, when the `adopted` field in the table is updated to `true`, a Lambda function is triggered, sending an email to the adoption center to notify them that the baby named "[baby abc]" has been adopted.

## Instructions

### Stage 1 - Creating the DynamoDB table

Follow these steps to create the DynamoDB table:

1. Access the DynamoDB dashboard by visiting [https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#](https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#).

2. Click on "Create table."

3. Set the **Table Name** as `baby-adoption`.

4. Specify the **Partition Key** as `babyID` (string).

5. Leave the **Sort Key** blank.

6. Customize the **Table Settings**.

7. Set the **Table Class** to "DynamoDB Standard."

8. Configure the **Read/write capacity settings** as "On-demand."

9. Under the **Secondary Indexes** section, create a **Global Index** with the following properties:
   - **Partition Key**: `adopted` (string)
   - **Sort Key**: Leave it blank
   - **Index Name**: "adopted-index"
   - **Attribute Projections**: "All"

10. Keep the **Encryption at rest** as "Owned by Amazon DynamoDB."

11. Click on "Create table."

### Stage 2 - Populating the table

Now, let's populate the `baby-adoption` table with babies:

1. Access your newly created `baby-adoption` table.

2. Click on "Actions" and select "Create item."

![Untitled](/images/Untitled.jpg)

3. Set the `babyID` **Value** to "baby0001".

4. Add a new attribute of type string:
   - **Attribute name**: "babyName"
   - **Value**: Provide the name of your chosen baby.

5. Add another new attribute of type string:
   - **Attribute name**: "adopted"
   - **Value**: Set it to "false."

6. Click on "Create item."

7. Repeat the above process as needed, ensuring to update the `babyID` with an incrementing number each time.

### Stage 2a - Querying our data

To demonstrate the effectiveness of the global secondary index, follow these steps:

1. Go to the "Explore table items" section.

![Untitled](/images/Untitled 1.jpg)

2. In the "Scan or query items" tab, select the **Query** option.

3. Change the **Index** to "adopted-index" and search for the value "true" in the partition key.

![Untitled](/images/Untitled 2.jpg)

4. This search for "true" will yield no results (assuming all baby records have "adopted: false").

![Untitled](/images/Untitled 3.jpg)

5. If you search for "false," it will retrieve all babies that have not been adopted.

![Untitled](/images/Untitled 4.jpg)

Querying utilizes indexes, making it more efficient and cost-effective. Without the secondary index, scanning the table would be necessary to identify adopted and non-adopted babies. This scanning process involves iterating over every record in DynamoDB, which can be expensive and slow, particularly for tables with a large number of records (thousands or millions).

## Stage 3 - SNS Setup

To set up SNS for your project, follow these steps:

1. Navigate to the SNS console by clicking on the following link: [https://us-east-1.console.aws.amazon.com/sns/v3/home?region=us-east-1#/topics](https://us-east-1.console.aws.amazon.com/sns/v3/home?region=us-east-1#/topics).

2. Click on "Create topic" in the console.

3. Set the **Type** to "Standard".

4. Specify the **Name** as "Adoption-Alerts".

5. In the **Access policy** section, keep the **Method** as "Basic".

6. Modify the **Define who can publish messages to the topic** option to "Only the specified AWS accounts" and enter your account ID (located in the top right corner of the screen).

7. Change the **Define who can subscribe to this topic** option to "Only the specified AWS accounts" and enter your account ID again.

Note: In a real-world scenario, it is recommended to restrict publishing access to only the necessary resources. However, for this temporary setup example, limiting access to the AWS account is sufficient and secure.

8. Leave all other options as default.

9. Click on "Create topic".

10. On the next page, click on "Create subscription".

11. Set the **Protocol** to "Email".

12. Enter your personal email address in the **Endpoint** field.

13. Click on "Create subscription".

14. Shortly after, you will receive a confirmation email with a link that needs to be clicked. This confirmation acknowledges your consent to receive emails from the topic and prevents spam through SNS.

Side note: While writing this, the confirmation email went to the Spam folder in Gmail, so remember to check there as well.

15. Your subscription should now be in the Confirmed state, as shown below:

![Untitled](/images/Untitled 5.jpg)

## Stage 4 - Lambda Creation

To create the Lambda function for your project, follow these steps:

1. Go to the Lambda console by clicking on the following link: [https://us-east-1.console.aws.amazon.com/lambda/home?region=us-east-1#/begin](https://us-east-1.console.aws.amazon.com/lambda/home?region=us-east-1#/begin).

2. Click on "Create function".

3. Leave the "Author from scratch" option selected.

4. Set the **Function name** to `baby-adoption-function`.

5. Select the **Runtime** as "Python 3.9".

6. Keep the **Architecture** as "x86_64".

7. Expand the **Change default execution role** section and set the **Execution role** to "Create a new role from AWS policy templates".

8. Specify the **Role name** as `baby-adoption-function-role`.

9. Add the `Amazon SNS publish policy` template to the **Policy templates** section.

10. Click on "Create function".

11. In the **Code** tab, insert the provided Python code:

```python
import boto3

def lambda_handler(event, context):
    sns = boto3.client('sns')
    accountid = context.invoked_function_arn.split(":")[4]
    region = context.invoked_function_arn.split(":")[3]
    for record in event['Records']:
        message = record['dynamodb']['NewImage']
        babyName = message['babyName']['S']

        response = sns.publish(
            TopicArn=f'arn:aws:sns:{region}:{accountid}:Adoption-Alerts',
            Message=f"{babyName} has been adopted!",
            Subject='Baby adopted!'
        )
```

This code iterates through the records passed to the Lambda function by DynamoDB and publishes a message to SNS.

Don't forget to click on "Deploy" to save the function.

![Untitled](/images/Untitled 6.jpg)

## Stage 4a - Adding DynamoDB Permissions to the Lambda Role

To incorporate DynamoDB permissions into the Lambda role, follow these steps:

1. Go to the Lambda console by accessing the following link: [https://us-east-1.console.aws.amazon.com/lambda/home?region=us-east-1#/functions](https://us-east-1.console.aws.amazon.com/lambda/home?region=us-east-1#/functions).

2. Click on the newly created function, then navigate to the **Configuration** tab and select **Permissions**.

3. Click on the associated role.

![Untitled](/images/Untitled 7.jpg)

4. Proceed to click on "Add Permissions" and then select "Attach Policies".

5. Search for the policy named "AmazonDynamoDBFullAccess" and choose it.

6. Click on "Attach Policies".

7. Your role policies should resemble the following example (note that the blurred policy ID will differ in your case, but it has been obscured for simplicity):

![Untitled](/images/Untitled 8.jpg)

This step is necessary to enable the Lambda function to read from the DynamoDB Stream. In a real-world scenario, it is advisable to enforce stricter access controls. However, for this particular case, granting the Lambda full DynamoDB permissions for **all** tables is acceptable.

## Stage 5 - Enabling the DynamoDB Stream

To enable the DynamoDB Stream, follow these instructions:

1. Return to the DynamoDB console using the link provided: [https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#table?name=baby-adoption](https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#table?name=baby-adoption).

2. Click on the ************************************Export and streams************************************ tab, and under **DynamoDB stream details**, select **Enable**.

3. On the subsequent page, choose "New image" as the **View type**. We are only interested in the new data and don't require the previous record data.

4. Click **Enable stream**.

5. Click on **Create trigger** under **DynamoDB stream details**.

6. On the subsequent page, select your newly created "baby-adoption-function" under **Lambda function** and leave the other options unchanged.

7. Click **Create trigger**.

![Untitled](/images/Untitled 9.jpg)

## Stage 6 - Testing the Solution

To test the implemented functionality, perform the following steps:

1. Return to the DynamoDB console using the provided link: [https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#tables](https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#tables).

2. Click on **Explore table items**.

3. You should be able to see the previously created baby items. If they are not visible, click on **Scan** ? **Run**.

![Untitled](/images/Untitled 10.jpg)

4. Modify one of the "adopted" fields to "true".

![Untitled](/images/Untitled 11.jpg)

5. After a few seconds, you should receive an email notification confirming the successful adoption of the baby!

![Untitled](/images/Untitled 12.jpg)

## Stage 7 - Cleanup

Perform the following steps to clean up the project:

1. DynamoDB Cleanup:
   - Access the DynamoDB console at [https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#tables](https://us-east-1.console.aws.amazon.com/dynamodbv2/home?region=us-east-1#tables).
   - Select the table you created and click on the "Delete" button.
   - Ensure that "Delete all CloudWatch alarms for this table" is selected and leave "Create a backup of this table before deleting it" unselected.
   - Type "confirm" into the confirmation field and click "Delete table".
   - Refer to the image below for guidance:
   
   ![Untitled](/images/Untitled 13.jpg)

2. Lambda Cleanup:
   - Go to the Lambda console at [https://us-east-1.console.aws.amazon.com/lambda/home?region=us-east-1#/functions](https://us-east-1.console.aws.amazon.com/lambda/home?region=us-east-1#/functions).
   - Select the function you created and click on "Actions" ? "Delete".
   - Type "delete" into the confirmation field and click "Delete".
   - Refer to the image below for guidance:
   
   ![Untitled](/images/Untitled 14.jpg)

3. SNS Cleanup:
   - Access the SNS console at [https://us-east-1.console.aws.amazon.com/sns/v3/home?region=us-east-1#/topics](https://us-east-1.console.aws.amazon.com/sns/v3/home?region=us-east-1#/topics).
   - Select your topic and click on "Delete".
   - Type "delete me" into the confirmation field and click "Delete".
   - Refer to the image below for guidance:
   
   ![Untitled](/images/Untitled 15.jpg)

4. Subscriptions Cleanup:
   - Navigate to the Subscriptions page and select your subscription.
   - Click on "Delete" and confirm the action by clicking "Delete" again.
   - Refer to the image below for guidance:
   
   ![Untitled](/images/Untitled 16.jpg)

5. IAM Cleanup:
   - Visit the IAM console at [https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/roles](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/roles).
   - Under **Roles**, search for "baby-adoption".
   - Select the role and click on "Delete".
   - Type "baby-adoption-function-role" into the confirmation field and click "Delete".
   - Refer to the image below for guidance:
   
   ![Untitled](/images/Untitled 17.jpg)

6. CloudWatch Logs Cleanup:
   - Go to the CloudWatch Logs console at [https://us-east-1.console.aws.amazon.com/cloudwatch/home?region=us-east-1#logsV2:log-groups](https://us-east-1.console.aws.amazon.com/cloudwatch/home?region=us-east-1#logsV2:log-groups).
   - Search for the "/aws/lambda/baby-adoption-function" log group.
   - Select the log group and click on "Actions" ? "Delete".
   - Confirm the deletion by clicking "Delete" in the popup.
   - Refer to the image below for guidance:
   
   ![Untitled](/images/Untitled 18.jpg)
Once you have completed these steps, the cleanup process will be finished.