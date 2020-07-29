## Core Idea
There are situations where you need to send a ZohoSign Document via a Deluge custom function upon criteria met, instead of using the manual "Send for ZohoSign" button from a record. It is important to note that when a ZohoSign Document is sent from CRM using the default button, 3 records in 3 separate modules are automatically created in the backend - **ZohoSign Documents, ZohoSign Document Events, ZohoSign Recipients**. The same does not happen however, when it is sent via Deluge. For document trackability in CRM, we think that it's sufficient that function only accounts for the record creation in the **ZohoSign Documents** module.

## Configuration
Before you can use this script with Zoho CRM, you must specify the following in the script:
1. ZohoSign - ZohoCRM integration must be done.
2. Under Customization -> Module and Fields -> Organize Modules, make sure that all 3 modules (**ZohoSign Documents, ZohoSign Document Events, ZohoSign Recipients**) are enabled.
3. A ZohoSign template should be created first.
4. Create 3 custom fields in **ZohoSign Documents** module as variables for e-mail notification and field update using workflow rules.
    * Check Date (date field)
    * Frequency (number field)
    * Request ID (string field)

## Tutorial
### Send ZohoSign Document & Create record in CRM
This script works in the following order:
1. Merge the necessary fields from CRM to prefill the ZohoSign template.
2. Create a ZohoSign Document to be sent to the relevant recipient(s).
3. Create a **ZohoSign Documents** module with relevant fields filled up.

```javascript
//Get the name, address, email or any other necessary fields to merge in the Zoho Sign Document
record = zoho.crm.getRecordById("***ZOHO CRM RECORD","***RECORD ID***");
name = record.get("***NAME FIELD***");
street = record.get("Shipping_Street");
state = record.get("Shipping_State");
city = record.get("Shipping_City");
code = record.get("Shipping_Code");
nameaddress = name + "\n" + street + ", " + state + "," + "\n" + city + ", " + code;
email = record.get("***EMAIL FIELD***");

//Making a parameter map to create a Zoho Sign document with prefilled fields using a Zoho Sign Template - Note: You first need to create a template in Zoho Sign
actionMap = Map();
fieldTextData = Map();
fieldTextData.put("Name and Address",nameaddress);
actionMap.put("field_data",{"field_text_data":fieldTextData});
eachActionMap1 = Map();
eachActionMap1.put("recipient_name",name);
eachActionMap1.put("recipient_email",email);
eachActionMap1.put("action_type","SIGN");
eachActionMap1.put("action_id","111430000000143036");
eachActionMap1.put("role","Reviewer");
eachActionMap1.put("verify_recipient","false");
fieldList = List();
fieldList.add(eachActionMap1);
actionMap.put("actions",fieldList);
submitMap = Map();
submitMap.put("templates",actionMap);
parameters = Map();
parameters.put("is_quicksend","true");
parameters.put("data",submitMap);

//Send the Zoho Sign Template with the pre-filled fields to the desired recipient/ recipients
response = zoho.sign.createUsingTemplate("***ZOHO SIGN TEMPLATE ID***",parameters);

//Get the Template Name - used as the name for the record in the "ZohoSign Documents" module in Zoho CRM later
template = zoho.sign.getTemplateById("***ZOHO SIGN TEMPLATE ID***"");
templatename = template.get("templates").get("template_name");

//Get the Zoho Sign Document ID
requests = response.get("requests");
docs = requests.get("document_ids");
docinfo = docs.get(0);
docid = docinfo.get("document_id");

//Get the Zoho Sign Request ID
reqid = requests.get("request_id");
docinfo = docs.get(0);
docid = docinfo.get("document_id");

//Create Map for CRM Record - ZohoSign Documents
recordmap = Map();
recordmap.put("Name",templatename);
recordmap.put("zohosign__Account",accountid);
recordmap.put("zohosign__Deal",dealid);
recordmap.put("zohosign__Owner",deal.get("Owner").get("id"));
recordmap.put("zohosign__Date_Sent",today);
recordmap.put("zohosign__ZohoSign_Document_ID",docid);
recordmap.put("zohosign__Module_Record_ID",dealid.toString());
recordmap.put("zohosign__Document_Deadline",today.addDay(3)); //--> This is arbitrary 
recordmap.put("zohosign__Document_Status","Out for Signature");

//Check Date and Frequency are additional fields created in the "ZohoSign Documents" CRM Module to control the email notification triggering and field update via workflow rule
recordmap.put("Check_Date",today.addDay(1));
recordmap.put("Frequency",0);
recordmap.put("Request_ID",reqid);

//Create a Record in the "ZohoSign Documents" module in Zoho CRM
create = zoho.crm.createRecord("zohosign__ZohoSign_Documents",recordmap);

```
### Workflow Document Field Update & Reminder
Because the ZohoSign Document is being triggered manually via Deluge instead of the default system button, the fields in the **ZohoSign Document** CRM module do not get updated automatically. Hence, a workflow rule with custom function is needed to get the document status and other relevant information from ZohoSign, then update in the record in Zoho CRM. Concurrently, an email notification reminder to the record owner (which many people find useful) can also be acheived here.

### Workflow Configuration
1. **Module**: **ZohoSign Documents**
2. **When**
    * Execute on `Check Date`
    * Execute at `05:00 AM` (*time is arbitrary, though it's recommended to set the time before work usually starts for updated data*)
    * Recur `Once`    
3. **Condition**
    * `Document Status` doesn't contain `SIGNED` AND
    * `Document Deadline` due in days >= `0` AND
    * `Frequency` < `3` (this acts as a controller to stop the workflow after a specified number is reached - you may change it to any other number you desire)
4. **Instant Actions**
    * Email Notification to Record Owner
    * Custom Function (which will be elaborated in the following section)
    
 ### Custom Function
This script works in the following order:
1. Get status of the ZohoSign Document status that was sent
2. * IF the document status is **SIGNED**
      * The following fields in the **ZohoSign Document** CRM record gets updated:
        * Status: SIGNED
        * Date Completed: "Sign Date" (by getting info on the action time)
   * IF the document status is not **SIGNED**
      * The following fields in the **ZohoSign Document** CRM record gets updated:
        * Check Date: +1 (this keeps the workflow running if the Document doesn't get signed)
        * Frequency: +1 (this keeps count of the number of times the function is run - used as a controller in workflow)

```javascript
//Get the Document Status
record = zoho.crm.getRecordById("zohosign__ZohoSign_Documents",recordid);
requestid = record.get("Request_ID");
document = zoho.sign.getDocumentById(requestid);
status = document.get("requests").get("actions").get(0).get("action_status");

//Validation - If document is signed, update status and sign date. Otherwise, add the Check Date and Frequency by 1 day
if(status = "SIGNED")
{
	signdate = document.get("requests").get("action_time");
	signdate = signdate.toDate("yyyy-MM-dd");
	update1 = zoho.crm.updateRecord("zohosign__ZohoSign_Documents",recordid,{"zohosign__Document_Status":status});
	update2 = zoho.crm.updateRecord("zohosign__ZohoSign_Documents",recordid,{"zohosign__Date_Completed":signdate});
}
else
{
	frequency = record.get("Frequency") + 1;
	update3 = zoho.crm.updateRecord("zohosign__ZohoSign_Documents",recordid,{"Check_Date":today.addDay(1)});
	update4 = zoho.crm.updateRecord("zohosign__ZohoSign_Documents",recordid,{"Frequency":frequency});
}

```
