# 3cx_Odoo
A guide on how to setup screen pops from 3cx to Odoo CRM

I'm a small MSP getting going with Odoo Online as my CRM. I plan to hire some help to answer calls and make outbound lead calls and wanted a way to make it easier on them to document interactions with screen pops. I've found several add-ins that work with on-prem instances, but nothing for Online. Several other threads say its not possible. Well - not being one to let things go, I figured it out and thought I'd share for the community.

Step 1: Enable API calls in Odoo Online. Log into your database > Click your avatar > go to My Profile, then open Account Security. Scroll to the bottom and create a new API key. Save this in a safe place, it's only shown once. While you're here, make note of your user ID. It's the last number in the URL. EX: https://www.yourodoo.domain/odoo/apps/action-651/2  <<< 2 is the ID. 

Step 2: Build the connector. I used Power Automate to act as an intermediary because that's what I'm comfortable in, but you can use anything that will accept and send API calls. 
Note - variables and their values are all case sensitive. This is particularly important for the conditions ("true") and when setting the response variable to "new". 

UPDATE: I added a JSON file you can import into N8N. In the Environmental Variables set, put in your DB name, API key, and User ID. If you want to go this route, download the file and import it to your N8N instance and skip to step 3. Remember to grab your 

1. Trigger: When a HTTP Request is received. Settings I use are: Accept a request from anyone, under Advanced parameters select Method and choose GET.
2. Initialize a variable, call it Response
3. Make the API call to Odoo: 
Replace values in < > with your info, remove the < > but KEEP THE QUOTES
#####################
<CODE>
URI: https://YOURDB.odoo.com/jsonrpc
Method: Post
Headers:
content-type:application/json
accept:application/json

Body:
{
  "jsonrpc": "2.0",
  "method": "call",
  "params": {
    "service": "object",
    "method": "execute_kw",
    "args": [
      "YOUR DATABASE NAME",
      YOUR USERID,
      "YOUR_API_KEY",
      "res.partner",
      "search_read",
      [
        [
          [
            "phone",
            "=",
            "@{concat('+',substring(trim(triggerOutputs()['queries']['TN']), 0, 1),' ',substring(trim(triggerOutputs()['queries']['TN']), 1, 3),'-',substring(trim(triggerOutputs()['queries']['TN']), 4, 3),'-',substring(trim(triggerOutputs()['queries']['TN']), 7, 4))}"
          ]
        ]
      ],
      {
        "fields": [
          "id"
        ],
        "limit": 1
      }
    ]
  },
  "id": YOUR USER ID
}
</CODE>
#####################
4. Set the variable Response with this as an expression: 
string(body('HTTP')?['result']?[0]?['id'])

5. Add in a Condition, check if the following input is "true"
empty(variables('Response'))]

   a. If True: Send a HTTP Request, label it "Create New Contact"

#####################
<CDOE>
URI: https://YOURDB.odoo.com/jsonrpc
Method: Post
Headers:
content-type:application/json
accept:application/json

Body:
{
  "jsonrpc": "2.0",
  "method": "call",
  "params": {
    "service": "object",
    "method": "execute_kw",
    "args": [
      "YOUR DATABASE NAME",
      YOUR USERID,
      "YOUR_API_KEY",
      "res.partner",
      "create",
      [
        {
          "phone": "@{concat(
  '+',
  substring(trim(triggerOutputs()['queries']['TN']), 0, 1),
  ' ',
  substring(trim(triggerOutputs()['queries']['TN']), 1, 3),
  '-',
  substring(trim(triggerOutputs()['queries']['TN']), 4, 3),
  '-',
  substring(trim(triggerOutputs()['queries']['TN']), 7, 4)
)
}",
          "name": "@{triggerOutputs()['queries']['Name']}"
        }
      ]
    ]
  },
  "id": 2
}  
 </CODE>
   #####################
   
   b. Set Response variable with ID from new contact: 
string(body('Create_New_Contact')?['result'])

   c. Add in another condition, check if "empty(variable('Response')) is "true" 
      a. If True, set response variable to "new"
7. Send a response:
#####################
Status Code: 200
Headers: content-type:text/html

Body:

<html>
  <head>
    <script type="text/javascript">
      window.location.href = "https://<YOURDATABASE>.odoo.com/odoo/contacts/@{variables('Response')}";
    </script>
  </head>
  <body>
    <p>Redirecting to contact...</p>
    <p>If you're not redirected, <a href="https://<YOURDATABASE>.odoo.com/odoo/contacts/@{variables('Response')}">click here</a>.</p>
  </body>
</html>

#####################
Publish the flow, then go to the trigger (labeled "manual" by default) to obtain the HTTP URI. We'll need this in the next step. 


Step 3: Configure 3CX and test
1. In 3CX, click Settings then Integration. Under CRM Integration, select Open Contact in Custom CRM. In the Open contact URL, paste the HTTP URI you copied from Power Automate and the following at the end:
&TN=%CallerNumber%&Name=%CallerDisplayName%

2. Set Notify when to your preference and give it a test.


When you receive a call, it will open the URI in a new browser window, kicking off the flow. It first checks if there are any contacts in your list with the caller's phone number. If it finds a record, it redirects you to their record in Odoo. If no records exist, power automate creates a new contact using the Caller ID display name as the "Name" and the ANI as the telephone number. If this is successful, it redirects you to the new contact. If that fails, it redirects you to the New Contact page so you can quickly and easily create a contact manually. In my tests with a contact list of around 70, this entire process takes around 3 seconds. 

I hope this helps someone out there. As always with any code or api calls, do your research and don't just copy/paste from some random person from the internet. Use at your own risk. If you have other ideas on how to make this better or further integrate with Odoo Online, I'd love to hear about it!

