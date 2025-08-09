# Project Documentation: (RepoReady)AI-Powered Student Submission Notifier

This document provides an in-depth guide to setting up, configuring, and
understanding the **AI-Powered Student Submission Notifier** workflow. It's designed
for bootcamp administrators, fellow developers, or anyone interested in automating
student feedback processes using intelligent agents.

## Table of Contents

1. Introduction
2. Prerequisites
3. Workflow Import
4. Node-by-Node Configuration
   - 4.1 Webhook Node
   - 4.2 Code Node (Initial GitHub Payload Processing)
   - 4.3 Get row(s) in sheet Node (Google Sheets)
   - 4.4 Edit Fields Node
   - 4.5 AI Agent Node (Google Gemini)
   - 4.6 Merge Node
   - 4.7 Code1 Node (Normalize AI Output)
   - 4.8 Code2 Node (Sanitize Phone Number)
   - 4.9 Twilio / HTTP Request Node (SMS & WhatsApp)
     - 4.9.1 SMS Configuration (Twilio Node)
     - 4.9.2 WhatsApp Configuration (HTTP Request Node)
5. Key Learnings & Concepts
   - 5.1 Automation vs. AI Agents
   - 5.2 Data Flow and Merging
   - 5.3 Twilio API Specifics
6. Testing the Workflow
7. Troubleshooting Common Issues
8. Conclusion

## 1. Introduction

This documentation provides an exhaustive guide to the **AI-Powered Student
Submission Notifier** , an n8n workflow designed to automate and personalize the
project submission confirmation process for a bootcamp utilizing GitHub Classroom
and Google Sheets. The core functionality involves triggering a notification upon a
student's first push to their repository, leveraging a large language model (LLM) to craft a personalized message, and delivering it via SMS or WhatsApp.

This document will walk you through every step, from initial setup to detailed node
configurations and troubleshooting, ensuring you can deploy and adapt this solution
effectively.

## 2. Prerequisites

Before you begin, ensure you have the following accounts and access:

```
● n8n Instance:
- A running n8n instance (either self-hosted or via n8n Cloud). Ensure it's
accessible publicly if your GitHub webhooks need to reach it.
● GitHub Classroom:
- An active GitHub organization set up with GitHub Classroom.
- Administrator access to create assignments and manage webhooks.
● Google Account:
- Access to Google Sheets where your student data is stored.
- The sheet must contain at least GitHubUsername, StudentName, and Phone
columns.
● Twilio Account:
- An active Twilio account.
- An SMS-enabled Twilio phone number (for sending SMS).
- A Twilio WhatsApp Sandbox (for testing WhatsApp messages) or an approved
WhatsApp Business API profile.
```

## 3. Workflow Import

The easiest way to get started is by importing the provided n8n workflow JSON file.

1. **Download the Workflow JSON:**
   - Navigate to the main repository for this project.
   - Download the workflow.json file.
2. **Import into n8n:**
   - Open your n8n instance in your web browser.
   - In the n8n interface, click on **"Add New"** in the left sidebar.
   - Select **"Import from JSON"** from the options.
   - Upload the workflow.json file you downloaded.
   - The workflow will appear in your n8n editor.

## 4. Node-by-Node Configuration

Once the workflow is imported, you'll need to configure each node with your specific credentials and settings.

**4.1 Webhook Node**

This node serves as the entry point for your workflow, listening for events from GitHub
Classroom.

```
● Purpose: To receive push event payloads from GitHub.
● Configuration:
```

1. Click on the **Webhook** node.
2. Set the **HTTP Method** to POST.
3. Under **Webhook URLs** , you will see a unique URL. Copy this URL.
4. **GitHub Classroom Setup:**
   - Go to your GitHub Classroom settings for the specific assignment.
   - Navigate to the **Webhooks** section.
   - Add a new webhook.
   - **Payload URL:** Paste the URL copied from the n8n Webhook node.
   - **Content type:** Select application/json.
   - **Secret:** (Optional but recommended for security) Set a secret key here
     and configure the same secret in the n8n Webhook node under "Webhook
     Secret".
   - **Which events would you like to trigger this webhook?** Select Just the
     push event.
   - Ensure the webhook is **active**.

**4.2 Code Node (Initial GitHub Payload Processing)**

This small code snippet extracts essential information from the GitHub push event
payload.


● Purpose: To extract repoFullName, repoName, commitSha, and githubUser from
the incoming GitHub webhook payload. This simplifies downstream data access.
● Code:
```js
// Extract relevant data from the GitHub webhook payload
const rawBody = items[0].json._debug_rawBody; // Access the raw body which
contains the GitHub payload
// Check if rawBody and its properties exist
if (!rawBody || !rawBody.repository || !rawBody.sender || !rawBody.head_commit) {
// Log an error or return an empty item if the expected structure is not found
console.error("Invalid GitHub webhook payload structure received.");
return [] 
};
```
```js
const repoFullName = rawBody.repository.full_name;
const repoName = rawBody.repository.name;
const commitSha = rawBody.head_commit.id;
const githubUser = rawBody.sender.login;
```

```js
// Determine if it's the initial commit (first push to the repo)
// GitHub webhooks for initial push often have 'before' as all zeros,
// or if the repo was just created and this is the first real commit.
const isInitialCommit = rawBody.before ===
"0000000000000000000000000000000000000000";
```

```js
// Create a memory key for tracking purposes, useful for ensuring one-time
notifications
const memoryKey = `${repoFullName}::first-submission`;
```

```js
return [{
json: {
repoFullName: repoFullName,
repoName: repoName,
commitSha: commitSha,
githubUser: githubUser,
beforeHash: rawBody.before, // Include for debugging initial commit logic
isInitialCommit: isInitialCommit,
memoryKey: memoryKey,
_debug_rawBody: rawBody // Keep raw body for debugging
}
}];
```

**4.3 Get row(s) in sheet Node (Google Sheets)**

This node fetches student details from your Google Sheet based on their GitHub
username.

```
● Purpose: To retrieve StudentName and Phone number using the
GitHubUsername from the previous node.
● Configuration:
```

1. **Credentials:** Select or create your **Google Sheets API credential**. Ensure it
   has read access to your spreadsheet.

2. **Operation:** Set to Get.
3. **Resource:** Row.
4. **Spreadsheet ID:** Enter the ID of your Google Sheet. You can find this in the
   sheet's URL (e.g.,
   https://docs.google.com/spreadsheets/d/YOUR_SPREADSHEET_ID/edit).
5. **Sheet Name:** Enter the exact name of the sheet (e.g., Sheet1 or Students).
6. **Filter By:**
   - **Name:** GitHubUsername (This should match the column header in your
     Google Sheet).
   - **Value:** ={{$node["Code"].json["githubUser"]}} (This dynamic expression
     pulls the GitHub username from the output of the previous "Code" node).

**4.4 Edit Fields Node**

This node consolidates and structures the data for subsequent AI message
generation and communication.

```
● Purpose: To ensure that all relevant data (studentName, phoneNumber,
repoFullName, githubUser) is present and correctly mapped for the downstream
AI Agent and messaging nodes.
● Configuration:
```

1. **Mode:** Manual Mapping.
2. **Fields to Set:** Add the following fields:
   - **Name:** studentName
     - **Value:** ={{$node["Get row(s) in sheet"].json["StudentName"]}}
   - **Name:** phoneNumber
     - **Value:** ={{$node["Get row(s) in sheet"].json["Phone"]}}
   - **Name:** repoFullName
     - **Value:** ={{$node["Code"].json["repoFullName"]}}
   - **Name:** githubUser
     - **Value:** ={{$node["Code"].json["githubUser"]}}

**4.5 AI Agent Node (Google Gemini)**

This is where the magic of personalized messaging happens, using an LLM to craft
engaging responses.

```
● Purpose: To generate a friendly and personalized submission confirmation
message for the student.
● Configuration:
```

1. **Model:** Select your preferred Google Gemini Chat Model (e.g., Google Gemini
   Chat Model). Ensure you have the necessary Google credentials configured in
   n8n.

2. **Chat Model Parameters:**
   - **Prompt:** This is where you instruct the AI on what message to generate.
     Use expressions to inject dynamic data.
     Generate a short, friendly, and enthusiastic message to a student
     confirming their project submission.
     The student's name is: {{$json.studentName}}
     The repository name is: {{$json.repoFullName}}
     Make sure to congratulate them and acknowledge their effort.
3. **Memory:** Use a Simple Memory for basic conversational context if needed,
   though for a one-off notification, it might be less critical.
4. The output of this node will contain the generated message, usually in a field
   like output or generatedMessage.

**4.6 Merge Node**

Crucial for combining data streams when your workflow branches.

```
● Purpose: To merge the student details (from Edit Fields) with the AI-generated
message (from AI Agent) into a single item that contains all necessary information
for sending the notification.
● Configuration:
```

1. **Mode:** Combine.
2. **Combine By:** Position.
   - This ensures that the first item from Input 1 (Edit Fields) is combined with
     the first item from Input 2 (AI Agent). This is critical because both
     branches produce a single item that needs to be unified.

**4.7 Code1 Node (Normalize AI Output)**

This node standardizes the field name for the generated message, making it
consistent for downstream use.


● Purpose: To ensure that the AI's generated message is consistently available
under the generatedMessage field, regardless of how the AI Agent node (or other
LLMs) might name its output field. It also passes through all other original fields
from the merged data.
● Code:
```js
// Get the first item from the incoming data
const p = items[0].json || {};
```

```js
// Prioritize 'generatedMessage', then 'text', 'output', 'content',
// or from 'choices' for various LLM node outputs
const message = p.generatedMessage ||
p.text ||
p.output ||
p.content ||
(p.choices && p.choices[0] && (p.choices[0].text ||
(p.choices[0].message && p.choices[0].message.content))) ||
'';
```

```js
// Return the item, spreading the original properties and adding/overwriting
generatedMessage
return [{
json: {
...p, // This line is crucial! It passes through all original fields (including
phoneNumber)
generatedMessage: message
}
}];
```

**4.8 Code2 Node (Sanitize Phone Number)**

Ensures the phone number is in the correct international format for messaging
services.

● Purpose: To take the phoneNumber from the previous node, convert it to a string,
trim any whitespace, and ensure it starts with a + for E.164 compliance.
● Code:
```js
// Get the raw phone number from the input item
const raw = (items[0].json.phoneNumber || '').toString().trim();
```
```js
// Initialize sanitized phone number with the raw value
let sanitized = raw;
```

```js
// If the raw number exists and doesn't start with '+', add it
if (raw && !raw.startsWith('+')) {
sanitized = '+' + raw;
}
```

```js
// Return the original item's JSON data, adding the sanitizedPhone field
return [{
json: {
```

```js
...items[0].json, // Pass all original fields through
sanitizedPhone: sanitized
}
}];
```

**4.9 Twilio / HTTP Request Node (SMS & WhatsApp)**

This is the final step where the personalized message is delivered. The configuration
differs based on whether you're sending SMS or WhatsApp.

**4.9.1 SMS Configuration (Twilio Node)**

If you want to send SMS messages, use the dedicated Twilio node in SMS mode.

● Node Type: Send an SMS/MMS/WhatsApp message (Twilio node)

● Credentials: Select your Twilio account credential.

● Resource: SMS

● Operation: Send

● From: Your Twilio SMS-enabled phone number (e.g., +14783304581).

● To: ={{$json["sanitizedPhone"]}} (Pulls the sanitized phone number from Code
node's output).

● Message: ={{$json["generatedMessage"]}} (Pulls the AI-generated message
from Code1 node's output).

● To Whatsapp: Ensure this toggle is OFF.


**4.9.2 WhatsApp Configuration (HTTP Request Node)**

If you want to send WhatsApp messages, you'll likely need to use a generic HTTP
Request node configured for Twilio's API, as the standard Twilio node might not
expose a direct WhatsApp resource.

● Node Type: HTTP Request

● Credentials: Select your Twilio account credential (under "Authentication" ->
"Predefined credential type").

● Method: POST

● URL:
https://api.twilio.com/2010-04-01/Accounts/{YOUR_ACCOUNT_SID}/Messages.jso
n
- Important: Replace {YOUR_ACCOUNT_SID} with your actual Twilio Account
SID, found on your Twilio Dashboard.
● Authentication:
- Type: Predefined credential type
- Credential: Select your configured Twilio API key credential.

● Send Body: Enable this toggle.

● Body Content Type: Form-urlencoded

● Body Parameters: Add the following parameters:
- Name: To
  - Value: whatsapp:{{$json["sanitizedPhone"]}} (The whatsapp: prefix is
crucial for WhatsApp messages).
- Name: From
  - Value: whatsapp:+14155238886 (This must be your Twilio WhatsApp
Sandbox number or your approved WhatsApp Business API sender
number).
- Name: Body
  - Value: ={{$json["generatedMessage"]}}

## 5. Key Learnings & Concepts

Building this project provided valuable insights into workflow automation and the
integration of AI agents.

**5.1 Automation vs. AI Agents**

```
● Automation: Refers to systems that follow a predefined, fixed set of steps to
accomplish tasks. They excel in repetitive and predictable scenarios. In this
project, the overall workflow (triggering on push, looking up data, sending
messages) is an automation.
● AI Agent: Is a more intelligent component within an automation that can reason,
plan, and act autonomously to achieve a goal. It offers flexibility and
adaptability in dynamic situations. In this workflow, the AI Agent node (using
Google Gemini) provides this dynamic reasoning by generating a custom
message based on contextual input, rather than just sending a static, predefined
text.
```

This project is a prime example of **Intelligent Automation** , where an AI agent
enhances a traditional automated workflow.

**5.2 Data Flow and Merging**

A critical aspect of this workflow was managing data flow, especially when branches
diverge and then need to converge again.

```
● The initial data from the Webhook and Google Sheets is processed.
● The workflow splits to generate an AI message in parallel.
● The Merge node is essential for bringing these disparate data streams back
together into a single, comprehensive item. Understanding its Combine mode with
```

```
Position combining strategy was key to ensuring all necessary fields
(phoneNumber and generatedMessage) were available for the final messaging
step.
```

**5.3 Twilio API Specifics**

Integrating with Twilio required careful attention to their API requirements:

```
● E.164 Format: All phone numbers (To and From) must be in the international E.
format (e.g., +2348161522624). The Code2 node handles this sanitization.
● Channel Specificity:
○ SMS: Requires only the E.164 formatted number in the To field.
○ WhatsApp: Requires the whatsapp: prefix for both To and From numbers
(e.g., whatsapp:+14155238886).
● API Call Structure: For Custom API Call mode, parameters like To, From, and
Body must be sent as Form-urlencoded body parameters, not query parameters.
● Sender Authentication: The From number must be a valid, enabled Twilio
number for the chosen channel (SMS or WhatsApp).
```

## 6. Testing the Workflow

To thoroughly test your workflow:

1. **Activate the Workflow:** Ensure your workflow is active in n8n.
2. **Trigger Manually:**
   - In the n8n editor, go to the Webhook node.
   - Click on **"Listen for Test Event"**.
   - Manually push a commit to one of your GitHub Classroom student
     repositories.
   - Observe the data flowing through each node in n8n's execution view.
3. **Check Outputs:** Verify the output of each node, especially the Merge node
   (should be a single item with all data), Code2 (should have a sanitizedPhone in
   E.164 format), and the final Twilio/HTTP Request node (should show status:
   "queued" for successful submission).
4. **Verify SMS/WhatsApp Delivery:** Check the recipient's phone for the SMS or
   WhatsApp message.

## 7. Troubleshooting Common Issues

Here are some common issues encountered during development and their solutions:

```
● [ERROR: No path back to node] (n8n) :
- Cause: A node is trying to reference data from an upstream node that is not
directly connected or is outside its accessible lineage.
```

```
- Solution: Ensure the connections are correct. If data is needed from multiple
branches, use a Merge node to consolidate them before the referencing node.
Always use the correct node name in expressions (e.g., $node["Get row(s) in
sheet"].json["StudentName"]).
● Invalid 'To' Phone Number (Twilio Error Code 21211) :
- Cause: The phone number format is incorrect, or the whatsapp: prefix is used
for an SMS message (or vice-versa).
- Solution:
  - Ensure the number is in E.164 format (e.g., +234...).
  - If sending SMS, the To field should be just ={{$json["sanitizedPhone"]}}.
  - If sending WhatsApp, the To field should be
whatsapp:{{$json["sanitizedPhone"]}}.
  - Verify the Twilio node's Resource setting matches your intention (e.g., SMS
for SMS, or Custom API Call with whatsapp: for WhatsApp).
● Twilio could not find a Channel with the specified From address (Twilio Error
Code 63007) :
- Cause: The From phone number is not properly configured for the channel
(e.g., trying to send WhatsApp from a number only enabled for SMS), or the
whatsapp: prefix is missing for WhatsApp.
- Solution:
  - Ensure your From number is a Twilio number specifically configured for
the channel you are using (SMS or WhatsApp).
  - For WhatsApp, use your Twilio WhatsApp Sandbox number (e.g.,
whatsapp:+14155238886) in the From field.
● A 'To' phone number is required (Twilio Error Code 21604) :
- Cause: When using HTTP Request node, the To (and other) parameters were
sent as Query Parameters instead of Body Parameters.
- Solution: In the HTTP Request node, disable Send Query Parameters, enable
Send Body, set Body Content Type to Form-urlencoded, and add To, From,
Body fields under Body Parameters.
```

## 8. Conclusion

This AI-Powered Student Submission Notifier is a practical solution that automates a
common bootcamp administrative task while significantly enhancing the student
experience. By combining webhook triggers, data lookups, intelligent message
generation, and multi-channel communication, it demonstrates the power and
flexibility of modern workflow automation platforms like n8n. Feel free to adapt and
extend this project for your own needs!
