# Salesforce ↔ CompanyCam Integration Blueprint

This guide implements your requirement:

1. When a Salesforce **Opportunity** is created (or updated) and **Quote Number** has a value, create a **Project** in CompanyCam.
2. When users upload **photos/files** in CompanyCam, show them in Salesforce.

---

## 1) End-to-end architecture

- **Salesforce → CompanyCam (outbound):**
  - Use a **Record-Triggered Flow** on Opportunity.
  - Flow invokes an **Invocable Apex action**.
  - Apex calls CompanyCam API to create a project.
  - Store returned `companycam_project_id` back on Opportunity.

- **CompanyCam → Salesforce (inbound):**
  - Configure CompanyCam **Webhook** events for project/photo/file creation.
  - Webhook calls a Salesforce **REST endpoint** (`@RestResource`).
  - Endpoint validates webhook signature and upserts media records.
  - Media records are shown in an Opportunity related list.

---

## 2) Salesforce data model

Create these fields on **Opportunity**:

- `Quote_Number__c` (Text) *(you already have quote number; reuse existing field if present)*
- `CompanyCam_Project_Id__c` (Text, External ID)
- `CompanyCam_Project_URL__c` (URL)
- `CompanyCam_Last_Sync__c` (Date/Time)

Create custom object **CompanyCam_Media__c**:

- `Name` (Auto Number or Text)
- `Opportunity__c` (Lookup → Opportunity)
- `CompanyCam_Media_Id__c` (Text, External ID, Unique)
- `CompanyCam_Project_Id__c` (Text)
- `Media_Type__c` (Picklist: `photo`, `file`)
- `Media_URL__c` (URL)
- `Thumbnail_URL__c` (URL)
- `Captured_At__c` (Date/Time)
- `Raw_Payload__c` (Long Text Area)

---

## 3) Authentication and config

Use **Named Credential** for CompanyCam API:

- Named Credential: `CompanyCam_API`
- Base URL: CompanyCam API base URL
- Auth: Bearer token / OAuth (per CompanyCam account)

Create **Custom Metadata** `CompanyCam_Settings__mdt`:

- `Webhook_Secret__c`
- `Default_Project_Status__c`
- `Sync_Enabled__c`

---

## 4) Outbound create-project logic (Opportunity → CompanyCam)

### Trigger condition
In record-triggered flow (after save on Opportunity):

- `Quote_Number__c Is Null = False`
- `CompanyCam_Project_Id__c Is Null = True`

Then call invocable Apex `CompanyCamProjectService.createProjects`.

### Apex example (invocable)

```apex
public with sharing class CompanyCamProjectService {
    public class Request {
        @InvocableVariable(required=true) public Id opportunityId;
    }

    public class Response {
        @InvocableVariable public Id opportunityId;
        @InvocableVariable public String projectId;
        @InvocableVariable public String projectUrl;
        @InvocableVariable public String message;
    }

    @InvocableMethod(label='Create CompanyCam Project')
    public static List<Response> createProjects(List<Request> requests) {
        Map<Id, Opportunity> oppMap = new Map<Id, Opportunity>([
            SELECT Id, Name, Quote_Number__c, Account.Name, CompanyCam_Project_Id__c
            FROM Opportunity WHERE Id IN :new Map<Id, Request>(requests).keySet()
        ]);

        List<Opportunity> updates = new List<Opportunity>();
        List<Response> out = new List<Response>();

        for (Request req : requests) {
            Opportunity opp = oppMap.get(req.opportunityId);
            Response r = new Response();
            r.opportunityId = req.opportunityId;

            if (opp == null || String.isBlank(opp.Quote_Number__c) || !String.isBlank(opp.CompanyCam_Project_Id__c)) {
                r.message = 'Skipped: condition not met';
                out.add(r);
                continue;
            }

            HttpRequest hreq = new HttpRequest();
            hreq.setEndpoint('callout:CompanyCam_API/projects');
            hreq.setMethod('POST');
            hreq.setHeader('Content-Type', 'application/json');
            hreq.setBody(JSON.serialize(new Map<String, Object>{
                'name' => opp.Name,
                'external_id' => opp.Id,
                'address' => opp.Account != null ? opp.Account.Name : null,
                'note' => 'Quote Number: ' + opp.Quote_Number__c
            }));

            HttpResponse hres = new Http().send(hreq);
            if (hres.getStatusCode() >= 200 && hres.getStatusCode() < 300) {
                Map<String, Object> body = (Map<String, Object>) JSON.deserializeUntyped(hres.getBody());
                String projectId = (String) body.get('id');
                String projectUrl = (String) body.get('url');

                opp.CompanyCam_Project_Id__c = projectId;
                opp.CompanyCam_Project_URL__c = projectUrl;
                opp.CompanyCam_Last_Sync__c = System.now();
                updates.add(opp);

                r.projectId = projectId;
                r.projectUrl = projectUrl;
                r.message = 'Created';
            } else {
                r.message = 'Failed: ' + hres.getStatus() + ' | ' + hres.getBody();
            }

            out.add(r);
        }

        if (!updates.isEmpty()) update updates;
        return out;
    }
}
```

---

## 5) Inbound webhook logic (CompanyCam → Salesforce)

Create a public endpoint in Salesforce:

```apex
@RestResource(urlMapping='/companycam/webhook')
global with sharing class CompanyCamWebhookController {
    @HttpPost
    global static void handle() {
        RestRequest req = RestContext.request;
        String payload = req.requestBody.toString();

        // 1) Validate signature header using webhook secret (HMAC)
        // 2) Parse event type (photo/file created)
        // 3) Map CompanyCam project ID -> Opportunity
        // 4) Upsert CompanyCam_Media__c by CompanyCam_Media_Id__c

        RestContext.response.statusCode = 200;
        RestContext.response.responseBody = Blob.valueOf('{"ok":true}');
    }
}
```

### Webhook event mapping

- `project.photo.created` → `Media_Type__c = 'photo'`
- `project.file.created` → `Media_Type__c = 'file'`

Use `CompanyCam_Project_Id__c` to locate Opportunity:

```soql
SELECT Id FROM Opportunity WHERE CompanyCam_Project_Id__c = :projectId LIMIT 1
```

Then upsert to `CompanyCam_Media__c` (external ID = media ID).

---

## 6) Show files/photos in Salesforce

Options:

1. **Quickest:** Add `CompanyCam_Media__c` related list to Opportunity layout.
2. **Better UX:** Build an LWC gallery component showing thumbnails and links.
3. **Optional mirror into Salesforce Files:** Download binary from CompanyCam and create `ContentVersion`.

Recommended starting point: related list + thumbnail URL field.

---

## 7) Error handling and retry strategy

- Store failed outbound requests in a custom object `Integration_Log__c`.
- Use Queueable Apex for retries (exponential backoff).
- Make webhook processing **idempotent** using unique `CompanyCam_Media_Id__c`.
- Log webhook signature failures separately.

---

## 8) Security checklist

- Restrict webhook endpoint via signature validation.
- Keep API token in Named Credential only.
- Add field-level security for integration fields.
- Add connected app/IP restrictions if available.

---

## 9) UAT test cases

1. Create Opportunity without Quote Number → no project created.
2. Add Quote Number to Opportunity → project created once.
3. Re-edit Opportunity → duplicate project is not created.
4. Upload photo in CompanyCam project → record appears in Salesforce related list.
5. Upload file in CompanyCam project → record appears in Salesforce related list.
6. Replay same webhook payload → no duplicate media rows.
