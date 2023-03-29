## Send WhatsApp messages via Amazon Pinpoint

### Solution & Architecture

An integration between Amazon Pinpoint and WhatsApp can be achieved for both outbound and inbound messages. The next section dives deeper into the architecture for both outbound and inbound messages. The solution uses Amazon Pinpoint custom channel, AWS Lambda, Amazon API Gateway, AWS Cloudformation and AWS Secrets Manager.

#### Outbound messages
For outbound messages Amazon Pinpoint integrates with WhatsApp via its [custom channel](https://docs.aws.amazon.com/pinpoint/latest/developerguide/channels-custom.html) allowing users to send WhatsApp messages using Pinpoint campaigns and journeys. Specifically, Pinpoint invokes an AWS Lambda function and performs an API call to WhatsApp. The API call contains the WhatsApp authorization key, the customer's mobile number and the WhatsApp message template name.

![architecture_outbound](https://github.com/aws-samples/amazon-pinpoint-whatsapp/blob/main/Assets/Architecture-Outbound-Messages.PNG)
1. Amazon Pinpoint campaign or journey using endpoint type **CUSTOM** invokes an AWS Lambda function. The payload along with the endpoint data should contain the WhatsApp message template name as part of the **Custom Data** field.
2. The AWS Lambda obtains the WhatsApp access token from the AWS Secrets Manager and performs a POST API call to the WhatsApp API.
3. The WhatsApp message gets delivered to the customer.

#### Inbound messages
For inbound messages WhatsApp requires a Callback URL. This solution utilizes Amazon API Gateway to create the Callback URL and AWS Lambda to authorize and process inbound messages.

![architecture_inbound](https://github.com/aws-samples/amazon-pinpoint-whatsapp/blob/main/Assets/Architecture-Inbound-Message.PNG)
1. Customer sends a message to your WhatsApp number.
2. WhatsApp makes a GET API call to the Amazon API Gateway endpoint for verification purposes. All subsequent calls containing the customers' messages are POST.
3. If the API call method is GET, the AWS Lambda checks if the verify token matches the one stored as an AWS Lambda Environment Variable. If it's TRUE, it returns a code called **HubChallenge** that WhatsApp is expecting in order to verify the connection. For POST API calls, the AWS Lambda loops through the customer messages and retrieves the customer number, timestamp, message_id and message_body. For each message processed, the AWS Lambda function performs an API call to WhatsApp to mark the message as read.

### Considerations
1) [AWS account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/).
2) Amazon Pinpoint project – [How to create an Amazon Pinpoint project](https://catalog.workshops.aws/amazon-pinpoint-customer-experience/en-US/prerequisites/create-a-project).
3) An Amazon Pinpoint CUSTOM endpoint with address a mobile number which is associated to a WhatsApp account. See example CUSTOM endpoint in a CSV [here]().
4) A Meta (Facebook) developer account, for more details please go to the [Meta for Developers console](https://developers.facebook.com/).


### Prerequisites
1) [AWS account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/).
2) An Amazon Pinpoint project – [How to create an Amazon Pinpoint project](https://catalog.workshops.aws/amazon-pinpoint-customer-experience/en-US/prerequisites/create-a-project).
3) An Amazon Pinpoint **CUSTOM** endpoint with address a mobile number which is associated to a WhatsApp account. See example CUSTOM endpoint in a CSV [here](https://github.com/aws-samples/amazon-pinpoint-whatsapp/blob/main/Assets/segment-import%20-CUSTOM.csv).
4) [Set up developer assets and platform access](https://developers.facebook.com/docs/whatsapp/cloud-api/get-started#set-up-developer-assets).

### Implementation
1. Navigate and login into the [Meta for Developers](https://developers.facebook.com/) console, click **My Apps** and select **Create App** (or use an existing app of type Business).
2. Select **Business** as an app type, which supports WhatsApp and click **Next**.
3. Provide a **display name, contact email**, choose whether or not to attach Business Account (optional) and select **Create App**.
4. Navigate to the **Dashboard** and select **Set Up** in the **WhatsApp** service in the **Add product to your app** section
5. **Create** or select an existing Meta Business Account and select **Continue**.
6. Navigate to **WhatsApp/Getting Started** and take a note of the **Phone number ID**, which will be needed in AWS CloudFormation template later on. ![PhoneNumberId](https://github.com/aws-samples/amazon-pinpoint-whatsapp/blob/main/Assets/WhatsAppPhoneNumberId.png)
7. On the WhatsApp/Getting Started page, add your customer phone number you are going to use for testing in the Select a recipient phone number dropdown. Follow the instructions to add and verify your phone number. Note: You must have WhatsApp registered with the number and the WhatsApp client installed on your mobile device. Verification message could appear in the Archived list in your WhatsApp client and not in the main list of messages.

### Create a new user to access WhatsApp via API

1. Open Meta’s [Business Manager](https://business.facebook.com/settings/) and select business you created or associated your app with earlier.
2. Below **Users**, select **System Users** and choose **Add** to create a new system user.
3. Give a name to the system user and set their role as **Admin** and click **Create System User**.
4. Use the **Add Assets** button to associate the new user with your WhatsApp app. From the **Select asset type** list, select **Apps**, then in the **Select assets**, select your WhatsApp app’s name. Enable the **Test app Partial access** for the user, select Save Changes and Done. 
5. Click on the **Generate new token button**, select the WhatsApp app created earlier and choose **Permanent** as **Token expiration**.
6. Select **whatsapp_business_messaging** and **whatsapp_business_management** from the list of **Available Permissions** and click **Generate token** at the bottom.
7. Copy and save your access token. This will be needed in AWS CloudFormation template later on.  Make sure you copied the token before clicking on **OK**.

For more details on creating the access token, you can navigate to **WhatsApp/Configuration** and click on [Learn how to create a permanent token](https://developers.facebook.com/docs/whatsapp/business-management-api/using-the-api#1--acquire-an-access-token-using-a-system-user-or-facebook-login).

### Solution deployment

1. Download the [AWS CloudFormation template](https://github.com/aws-samples/amazon-pinpoint-whatsapp/blob/main/README.md) and navigate to the AWS CloudFormation console under the AWS region you want to deploy the solution.
2. Select **Create stack and With new resources**. Choose **Template is ready** as **Prerequisite – Prepare template** and **Upload a template file** as **Specify template**. Upload the template downloaded in **step 1**.
3. Fill the **AWS CloudFormation parameters** as shown below:
    1. **ApiGatewayName:** This is the name of the Amazon API Gateway resource.
    2. **PhoneNumberId:** This is the WhatsApp phone number Id you obtained from the Meta for Developers console under **WhatsApp/Getting Started**.
    3. **PinpointProjectId:** Paste your Amazon Pinpoint's project Id. This allows Amazon Pinpoint to invoke the AWS Lambda, which sends WhatsApp messages as part of a campaign or journey.
    4. **VerifyToken:** The verify token is an alphanumeric token that you provide to WhatsApp when setting up the Webhook Callback URL for inbound messages and notifications. You can decide the value of this token e.g. 123abc.
    5. **WhatsAppAccessToken:** The access token should start with **Bearer EEAEAE...** and you should have obtained it from the section of this blog **Create a new user to access WhatsApp via API**.
4. Once the AWS CloudFormation stack is deployed, copy the Amazon API GateWay endpoint from the AWS CloudFormation outputs tab. Navigate to the **Meta for Developers** App dashboard, choose **Webhooks**, select **Whatsapp Business Account** and subscribe to **messages**. ![SubscribeToMessages](https://github.com/aws-samples/amazon-pinpoint-whatsapp/blob/main/Assets/SubscribeToMessages.png)
5. Paste the Amazon API Gateway endpoint as a **Callback URL**. For the **Verify token**, provide the same value as the AWS CloudFormation template parameter **VerfiyToken** and select **Verify and save**. ![VerifyAndSave](https://github.com/aws-samples/amazon-pinpoint-whatsapp/blob/main/Assets/VerifyAndSave.png)

### Testing
- **Sending messages**: To test sending a message to WhatsApp using Amazon Pinpoint:
    - Navigate to the Amazon Pinpoint Campaigns.
    - Create a new Campaign with **WhatsAppCampaign** as the **Campaign name**, select **Standard campaign** as the **Campaign type**, choose **Custom** as **Channel** and select **Next**. 
    - Select a segment that includes the **CUSTOM** endpoint that you will send the message to.
    - Choose the AWS Lambda Function containing the name **WhatsAppSendMessageLambda**. Under **Custom data** type **hello_world**, choose **Custom** as **Endpoint Options** and select **Next**. Note that the **hello_world** is the WhatsApp default message template.
    - In **Step 4** leave everything with the default values, scroll to the bottom of the page and select **Next**.
    - Choose **Launch campaign**.
- **Receiving messages**: Text or reply to the WhatsApp number. The inbound messages are being printed in the Amazon CloudWatch logs of the AWS Lambda function containing the name **WhatsAppWebHookLambda**. ![ReceivedMessages](https://github.com/aws-samples/amazon-pinpoint-whatsapp/blob/main/Assets/ReceivedMessage.png)

### Next steps

There are several ways to extend this solution's functionality, see some of them below:
- Instead of specifying the WhatsApp message template name, provide directly the text you want to send using the Pinpoint's custom channel **Custom data** field. To do this, update the AWS Lambda function code responsible for sending messages with the one below: 
``` python
import os
import json
import boto3
from urllib import request, parse
from botocore.exceptions import ClientError
phone_number_id = os.environ['PHONE_NUMBER_ID']
secret_name = os.environ['SECRET_NAME']
def handler(event, context):
    print("Received event: {}".format(event))
    session = boto3.session.Session()
    client = session.client(service_name='secretsmanager')
    try:
        get_secret_value_response = client.get_secret_value(SecretId=secret_name)
    except ClientError as e:
        raise e
    else:
        secret = get_secret_value_response['SecretString']
        url = 'https://graph.facebook.com/v15.0/'+ phone_number_id + '/messages'
        message = event['Data'] # Obtaining the message from the Custom Data field
        for key in event['Endpoints'].keys(): 
            to_number = str(event['Endpoints'][key]['Address'])
            send_message(secret, to_number, url, message_template)
def send_message(secret, to_number, url, message_template):
    headers = {
        'content-type': 'application/json',
        'Authorization': secret
    }
    # Building the request body and insted of type = template, it's replaced with type = text
    data = parse.urlencode({
        'messaging_product': 'whatsapp',
        'to': to_number,
        'type': 'text',
        'text': {
            'body': message
        }
    }).encode()
    req =  request.Request(url, data=data, headers=headers)
    resp = request.urlopen(req)
```
- Use WhatsApp's message template components to populated dynamically variables. This requires an update on the respective WhatsApp message template and API request body to WhatsApp's API. The message template should look like this: ![PersonalizedMessageTemplate](https://github.com/aws-samples/amazon-pinpoint-whatsapp/blob/main/Assets/PersonalizedMessageTemplate.png)
And the API request body should look like this. Note that the value for each variable should be obtained from the Pinpoint endpoint or user attributes.
```json
{
  "from": from_number,
  "to": to_number,
  "channel": "whatsapp",
  "content": {   
    "contentType": "template",
    "template": {
        "templateId" : "first_pinpoint_message",
        "templateLanguage" : "en",
        "components" : {
            "body" : [
                    {
                        "type": "text",
                        "text": "Pavlos"
                    }
            ]           
        }
   }
  }
}
```

