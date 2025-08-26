This document outlines the configuration and functionality of the provided SAP CPI Iflow, detailing its purpose, flow steps, and embedded resources.

---

## SAP CPI Iflow Documentation

### Iflow Name/ID: Default Collaboration (Implicit from XML)

### 1. Overview

This SAP CPI Iflow is designed to receive an XML message containing invoice data and associated attachments (in Base64 encoding), transform this data into a structured JSON payload, and then send it as a `multipart/form-data` request to an external SAP Ariba Central Invoice Management (CIM) API. The process includes parsing XML, extracting and decoding attachments, mapping the invoice data, and constructing the final multipart request.

### 2. Trigger

The Iflow is triggered by a **Timer Event** (`StartEvent_4665 - Start Timer 1`).
*   **Schedule Type:** Cron
*   **Cron Expression:** `0/5 * * ? * * *` (Every 5th minute, every hour, every day, every month, every day of the week, every year, starting at 0 seconds).
*   **Time Zone:** UTC 0:00 (Greenwich Mean Time).

### 3. Participants

*   **Sender (Participant_741 - EndpointSenderSender1):** An unspecified sender initiates the flow via the scheduled timer.
*   **Integration Process (Participant_Process_1 - Integration ProcessProcess_1):** Contains the core logic for message processing.
*   **Receiver (Participant_2 - EndpointRecevierReceiver):** The external SAP Ariba CIM API.

### 4. Endpoint Configuration

#### 4.1. Sender

*   **Type:** Endpoint Sender
*   **Authentication:** Not enabled (Basic Authentication is `false`).

#### 4.2. Receiver (HTTP Adapter)

*   **System:** Receiver
*   **Adapter Type:** HTTP
*   **HTTP Method:** POST
*   **Address:** `https://cim-cf-eu10-subdomain-2025-2-27-191418.eu10.cim.cloud.sap/api/cim-create-invoice-structured/v1/`
*   **Authentication Method:** OAuth2 Client Credentials
*   **Credential Name:** ARIBACIM
*   **Proxy Type:** Default
*   **Request Timeout:** 60000 ms
*   **Enable MPL Attachments:** True (This allows message processing log to capture attachments).
*   **Throw Exception On Failure:** True
*   **Message Protocol:** None
*   **Allowed Response Headers:** `*` (All headers are allowed)
*   **Retry on Connection Failure:** False
*   **Retry Iteration:** 1
*   **Retry Interval:** 5 seconds

### 5. Main Processing Flow (Integration Process)

The Iflow processes the invoice data through the following steps:

1.  **Start Timer 1 (`StartEvent_4665`)**: Initiates the flow based on the configured cron schedule.
2.  **Content Modifier 3 (`CallActivity_758`)**:
    *   **Purpose:** Sets the initial message body with a `StandardBusinessDocument` XML structure containing Base64 encoded invoice (application/xml) and PDF attachments (application/pdf).
    *   **Body Content:** A static XML payload with Base64 encoded IDoc and PDF attachments.
    *   **Headers/Properties:** No direct headers/properties are set by this step, but the subsequent steps will work with the XML content.
3.  **ReadingTheCountOfAttachments (`CallActivity_761`) - Groovy Script (`script6.groovy`)**:
    *   **Purpose:** Parses the incoming `StandardBusinessDocument` XML, extracts the Base64 encoded content of specific PDF attachments (`ID_2`, `ID_3`), and stores them as message properties. It also stores the IDs and count of these attachments.
    *   **Details:** It looks for `Attachment` elements with `Id` starting with `ID_2` or `ID_3` and `MimeType='application/pdf'`. The Base64 content is stored in properties named after their `Id` (e.g., `ID_2`, `ID_3`). The IDs are also collected into an `attachmentIds` property (comma-separated string) and `attachmentCount` property.
4.  **Filter 1 (`CallActivity_736`)**:
    *   **Purpose:** Filters the `StandardBusinessDocument` XML to extract only the XML attachment.
    *   **Details:** Uses an XPath expression `//*[local-name()='Attachment' and @MimeType='application/xml']` to select the XML attachment node. The result (the `<Attachment>` element for the XML) becomes the new message body.
5.  **Base64 Decoder 1 (`CallActivity_739`)**:
    *   **Purpose:** Decodes the Base64 encoded XML attachment (which is now the message body) back into its raw XML format (an `INVOIC02` IDoc).
6.  **Content Modifier 2 (`CallActivity_754`)**:
    *   **Purpose:** Sets an example IDoc XML structure as the body (though it should already be the body from the previous step) and extracts a specific value into a header for logging.
    *   **Header (`SAP_ApplicationID`):** Extracts the value of `INVOIC02/IDOC/E1EDK02/BELNR` from the XML message body using XPath and sets it as a header.
7.  **Message Mapping 1 (`CallActivity_712`) - Message Mapping (`MM_SON_CIM.mmap`)**:
    *   **Purpose:** Transforms the incoming IDoc XML (`INVOIC02`) into the target JSON structure required by the SAP Ariba CIM API.
    *   **Source:** `INVOIC02` (IDoc).
    *   **Target:** `CreateInvoiceStructuredPublicV1- CPICOMP.json` (JSON schema for Ariba CIM).
    *   **Key Mappings:**
        *   `origin` (constant: "04")
        *   `supplierInvoiceId` (from `INVOIC02/IDOC/E1EDK01/BELNR`)
        *   `purpose` (conditional mapping based on `INVOIC02/IDOC/E1EDK01/BSART`: "I" for "INVO", "C" for "CRME", "3" for "SUBD", "4" for "SUBC")
        *   `currency` (from `INVOIC02/IDOC/E1EDK01/CURCY`)
        *   `grossAmount` (from `INVOIC02/IDOC/E1EDS01/SUMME` where `SUMID` is "011")
        *   `documentDate` (from `INVOIC02/IDOC/E1EDK03/DATUM` where `IDDAT` is "012", formatted from `yyyyMMdd` to `yyyy-MM-dd`)
        *   `invoiceReceiptDate` (current date `yyyy-MM-dd`)
        *   `documentHeaderText` (from `INVOIC02/IDOC/E1EDKT1/TDOBJECT`)
        *   `externalIds/id` (concatenates "S_" with `INVOIC02/IDOC/E1EDKT1/E1EDKT2/TDLINE` if `TDID` is "007", removes contexts)
        *   `externalIds/type` (constant: "otherId")
        *   `receiverSupplier` (from `INVOIC02/IDOC/E1EDKA1/LIFNR` if `PARVW` is "LF")
        *   `receiverCompanyCode` (constant: "1010")
        *   `receiverSystemId` (constant: "0M4IK2D")
        *   `taxes/percentage` (from `INVOIC02/IDOC/E1EDK04/MSATZ`)
        *   `taxes/currency` (from `INVOIC02/IDOC/E1EDK01/CURCY`)
        *   `taxes/amount` (from `INVOIC02/IDOC/E1EDK04/MWSBT`)
        *   `lineItems` (from `INVOIC02/IDOC/E1EDP01` segments, mapping quantity, UoM, unit price, net amount, currency, description, tax rate, tax country, PO references, item number).
        *   `invoiceParties` (based on `INVOIC02/IDOC/E1EDKA1` segments, mapping role, name, address details, and other info).
8.  **AddAttachmentNode (`CallActivity_4671`) - Groovy Script (`script8.groovy`)**:
    *   **Purpose:** Dynamically modifies the JSON payload generated by the Message Mapping to include attachment metadata based on the previously extracted attachment IDs.
    *   **Details:** It reads the `attachmentIds` property (e.g., "ID_2,ID_3") and constructs an `attachments` array within the JSON payload. For `ID_2`, it sets `fileName: "invoice1.pdf", documentType: "Invoice"`. For `ID_3`, it sets `fileName: "Legal.pdf", documentType: "Legal"`.
9.  **Invoice_payload (`CallActivity_728`)**:
    *   **Purpose:** Stores the current JSON message body (which now includes attachment metadata) into a message property named `jsonbody`.
    *   **Property (`jsonbody`):** Stores `in.body` (the JSON string).
10. **MultiformDataIWithProperties (`CallActivity_4668`) - Groovy Script (`script7.groovy`)**:
    *   **Purpose:** Constructs the final `multipart/form-data` request body required by the Ariba CIM API.
    *   **Details:** It combines the JSON payload (from `jsonbody` property) and the Base64 decoded PDF attachments (from `ID_2`, `ID_3` properties) into a single multipart message. It dynamically assigns filenames ("invoice1.pdf", "Legal.pdf") to the PDF attachments.
11. **Request Reply 1 (`ServiceTask_4673`)**:
    *   **Purpose:** Sends the constructed `multipart/form-data` message to the external SAP Ariba CIM endpoint.
12. **End (`EndEvent_2`)**: Marks the successful completion of the Iflow.

### 6. Error Handling (Exception Subprocess)

An **Exception Subprocess** (`SubProcess_745`) is configured to catch and handle any errors that occur during the main processing flow.

1.  **Error Start 1 (`StartEvent_746`)**: This event is triggered when an exception occurs in the main integration process.
2.  **Build alert body (`CallActivity_749`)**:
    *   **Purpose:** Formats a human-readable alert message containing details of the error.
    *   **Body Content:** Uses expression language to build a message including:
        *   Iflow ID (`${camelId}`)
        *   Purchase Order ID (`${header.SAP_ApplicationID}`)
        *   Message Processing Log ID (`${property.SAP_MessageProcessingLogID}`)
        *   Current date and time (UTC and Asia/Calcutta timezone)
        *   HTTP Status Code (`${property.http.statusCode}`)
        *   Exception message (`${exception.message}`)
        *   HTTP Response Body (`${property.http.responseBody}`)
        *   Message stacktrace (`${exception.stacktrace}`)
3.  **Groovy Script 3 (`CallActivity_752`) - Groovy Script (`script5.groovy`)**:
    *   **Purpose:** Extracts relevant error information from the `CamelExceptionCaught` property, specifically handling different types of exceptions (HTTP, OData V2, SOAP) to retrieve the response body and status code.
    *   **Details:** Checks the class of `CamelExceptionCaught` and extracts `responseBody` and `statusCode` into message properties accordingly.
4.  **End 1 (`EndEvent_747`)**: Marks the end of the exception handling process.

### 7. Scripts Documentation

The following Groovy scripts are used in this Iflow:

#### 7.1. `script6.groovy` (Used in `CallActivity_761: ReadingTheCountOfAttachments`)

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import groovy.util.XmlSlurper

def Message processData(Message message) {
    def xml = message.getBody(String)
    def root = new XmlSlurper().parseText(xml)
    def idList = []

    // Find all Attachment elements with Id starting with 'ID_2' or 'ID_3' and MimeType 'application/pdf'
    def attachments = root.'**'.findAll { 
        it.name() == 'Attachment' && 
        (it.@Id.toString().startsWith('ID_2') || it.@Id.toString().startsWith('ID_3')) &&
        it.@MimeType == 'application/pdf'
    }

    attachments.each { att ->
        def id = att.@Id.toString()
        def content = att.text()
        message.setProperty(id, content) // Store Base64 content in properties named ID_2, ID_3
        idList << id
    }

    // Set the array and count as properties
    message.setProperty("attachmentIds", idList.join(",")) // e.g., "ID_2,ID_3"
    message.setProperty("attachmentCount", idList.size()) // e.g., 2

    return message
}
```
**Purpose:** This script is crucial for parsing the initial `StandardBusinessDocument` XML payload. It identifies specific PDF attachments within the SBD (those with IDs starting `ID_2` or `ID_3` and `MimeType='application/pdf'`), extracts their Base64 encoded content, and stores each as a separate message property. It also compiles a comma-separated list of these attachment IDs and their total count into new message properties (`attachmentIds` and `attachmentCount`).

#### 7.2. `script8.groovy` (Used in `CallActivity_4671: AddAttachmentNode`)

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import groovy.json.JsonSlurper
import groovy.json.JsonOutput

def Message processData(Message message) {
    def jsonPayload = message.getBody(String)
    def attachmentIdsStr = message.getProperty("attachmentIds") ?: ""
    def attachmentIds = attachmentIdsStr.split(',').findAll { it } // Remove empty strings

    def json = new JsonSlurper().parseText(jsonPayload)

    // Build attachments array based on number of IDs
    def attachments = []
    attachmentIds.eachWithIndex { id, idx ->
        if (idx == 0) { // Assuming first attachment is invoice, second is legal. This is fragile.
            attachments << [fileName: "invoice1.pdf", documentType: "Invoice"]
        } else {
            attachments << [fileName: "Legal.pdf", documentType: "Legal"]
        }
    }
    json.attachments = attachments // Overwrites or adds 'attachments' array in the JSON payload

    message.setBody(JsonOutput.toJson(json))
    return message
}
```
**Purpose:** This script dynamically modifies the JSON payload (produced by the Message Mapping) to accurately reflect the attachments that were extracted from the initial SBD. It takes the `attachmentIds` property (set by `script6.groovy`), parses the existing JSON, and inserts an `attachments` array with appropriate `fileName` and `documentType` entries. The current logic assigns "invoice1.pdf" (type "Invoice") for the first attachment and "Legal.pdf" (type "Legal") for subsequent attachments. This depends on the order of `attachmentIds` and a fixed mapping logic within the script.

#### 7.3. `script7.groovy` (Used in `CallActivity_4668: MultiformDataIWithProperties`)

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import java.nio.charset.StandardCharsets
import java.util.Base64
import java.io.ByteArrayOutputStream

def Message processData(Message message) {
    def messageLog = messageLogFactory?.getMessageLog(message)
    def jsonPayload = message.getProperty("jsonbody") // JSON payload for the 'invoice' part
    def boundary = "----WebKitFormBoundary7MA4YWxkTrZu0gW" // Fixed multipart boundary

    try {
        message.setHeader("Content-Type", "multipart/form-data; boundary=" + boundary)

        def allProps = message.getProperties()

        // Only include ID_2 and ID_3 for PDF attachments
        def keys = ["ID_2", "ID_3"] // Hardcoded keys for attachments to include
        def filenames = ["invoice1.pdf", "Legal.pdf"] // Hardcoded filenames for attachments

        ByteArrayOutputStream outputStream = new ByteArrayOutputStream()

        // JSON part of the multipart request
        outputStream.write(("--${boundary}\r
").getBytes(StandardCharsets.UTF_8))
        outputStream.write("Content-Disposition: form-data; name="invoice"\r
\r
".getBytes(StandardCharsets.UTF_8))
        outputStream.write((jsonPayload + "\r
").getBytes(StandardCharsets.UTF_8))

        // PDF parts from ID_2 and ID_3 properties
        keys.eachWithIndex { key, idx ->
            def v = allProps[key] // Get Base64 content from properties set by script6.groovy
            if (v && v instanceof String && !v.trim().isEmpty()) {
                def cleanedString = v.replaceAll("[\
\r\s]", "")
                byte[] decodedBytes = Base64.getDecoder().decode(cleanedString) // Decode Base64 content
                String fileName = filenames[idx] // Get filename from hardcoded list

                outputStream.write(("--${boundary}\r
").getBytes(StandardCharsets.UTF_8))
                outputStream.write(("Content-Disposition: form-data; name="invoice"; filename="${fileName}"\r
").getBytes(StandardCharsets.UTF_8))
                outputStream.write("Content-Type: application/pdf\r
\r
".getBytes(StandardCharsets.UTF_8))
                outputStream.write(decodedBytes) // Write decoded binary PDF content
                outputStream.write("\r
".getBytes(StandardCharsets.UTF_8))
            }
        }

        // Final closing boundary
        outputStream.write(("--${boundary}--\r
").getBytes(StandardCharsets.UTF_8))

        message.setBody(outputStream.toByteArray()) // Set the constructed multipart body
        outputStream.close()

        messageLog?.addAttachmentAsString("GroovyScript", "Successfully created multipart form-data with JSON and 2 PDFs", "text/plain")
    } catch (Exception e) {
        messageLog?.addAttachmentAsString("GroovyScript", "Error creating multipart form-data: ${e.getMessage()}", "text/plain")
        throw e
    }

    return message
}
```
**Purpose:** This is the final step in preparing the request body for the Ariba CIM API. It takes the JSON payload (from the `jsonbody` property, which `script8.groovy` updated) and the Base64 decoded PDF content (from `ID_2`, `ID_3` properties, which `script6.groovy` extracted) and assembles them into a `multipart/form-data` payload. It sets the `Content-Type` header with a defined boundary and appends each part (JSON and PDFs) separated by this boundary. The filenames for the PDFs are hardcoded (`invoice1.pdf`, `Legal.pdf`) in the script, matching the logic in `script8.groovy`.

#### 7.4. `script5.groovy` (Used in `CallActivity_752: Groovy Script 3` within Exception Subprocess)

```groovy
import com.sap.gateway.ip.core.customdev.util.Message;
import org.w3c.dom.Node;
import groovy.xml.*

def Message processData(Message message) {
                
	def map = message.getProperties();
	
	def ex = map.get("CamelExceptionCaught"); // Get the exception object
	if (ex!=null) {
					
		// Handle HTTP errors (e.g., from HTTP receiver adapter)
		if (ex.getClass().getCanonicalName().equals("org.apache.camel.component.ahc.AhcOperationFailedException")) {
			message.setBody(ex.getResponseBody()); // Set response body as message body
			message.setProperty("http.responseBody", ex.getResponseBody()); // Store response body in a property
			message.setProperty("http.statusCode", ex.getStatusCode());	// Store status code in a property
		}
		
		// Handle OData V2 errors
		if (ex.getClass().getCanonicalName().equals("com.sap.gateway.core.ip.component.odata.exception.OsciException")) {                                      
			message.setBody(message.getBody()); // Use current message body (often contains error details)
            message.setProperty("http.responseBody", message.getBody()); // Store it in property
            message.setProperty("http.statusCode", message.getHeaders().get("CamelHttpResponseCode").toString()); // Get HTTP status from header
        }
		
		// Handle SOAP faults
		if (ex.getClass().getCanonicalName().equals("org.apache.cxf.binding.soap.SoapFault")) {
			def xml = XmlUtil.serialize(ex.getOrCreateDetail()); // Extract SOAP fault details as XML
			message.setBody(xml); // Set as message body
			message.setProperty("http.responseBody", xml); // Store in property
			message.setProperty("http.statusCode", ex.getStatusCode());	// Store status code
		}
	}
	
	return message;
}
```
**Purpose:** This script is part of the error handling mechanism. When an exception is caught, it inspects the type of exception (`CamelExceptionCaught` property) and attempts to extract relevant error information such as the HTTP response body and status code. This information is then set as the message body and stored in message properties (`http.responseBody`, `http.statusCode`) to be used in the alert message generated by `CallActivity_749`.

#### 7.5. `script1.groovy`, `script2.groovy`, `script3.groovy`, `script4.groovy`

These scripts are present in the provided `Resources` section but are **not actively connected** or used within the defined `BPMNDiagram`'s active message flow paths. They might be leftover from previous iterations, alternative processing branches, or utility scripts not directly invoked by the current flow.

### 8. Mapping Documentation

#### 8.1. `MM_SON_CIM.mmap` (Used in `CallActivity_712: Message Mapping 1`)

**Source Structure:** `INVOIC02` (SAP IDoc for Invoice/Billing document)
**Target Structure:** `CreateInvoiceStructuredPublicV1- CPICOMP.json` (SAP Ariba Central Invoice Management API JSON schema)

This message mapping is responsible for transforming the traditional SAP IDoc XML format into the modern JSON structure expected by the Ariba CIM API. It involves direct field-to-field mapping, constant value assignments, and conditional logic to determine certain target fields.

**Key Mapping Details:**

*   **Header Level Mappings:**
    *   `origin`: **Constant value "04"**
    *   `supplierInvoiceId`: Mapped from `INVOIC02/IDOC/E1EDK01/BELNR`
    *   `purpose`: **Conditional mapping** based on `INVOIC02/IDOC/E1EDK01/BSART` (Document Type):
        *   "I" (Invoice) if `BSART` is "INVO"
        *   "C" (Credit/Memo) if `BSART` is "CRME"
        *   "3" (Subsequent Debit) if `BSART` is "SUBD"
        *   "4" (Subsequent Credit) if `BSART` is "SUBC"
    *   `currency`: Mapped from `INVOIC02/IDOC/E1EDK01/CURCY`
    *   `grossAmount`: Mapped from `INVOIC02/IDOC/E1EDS01/SUMME` (Total Value of Sum Segment), conditioned by `INVOIC02/IDOC/E1EDS01/SUMID` being "011".
    *   `documentDate`: Mapped and formatted from `INVOIC02/IDOC/E1EDK03/DATUM` (IDOC: Date) if `INVOIC02/IDOC/E1EDK03/IDDAT` is "012". Format transformation: `yyyyMMdd` to `yyyy-MM-dd`.
    *   `invoiceReceiptDate`: **Current Date** (`currentDate` function with format `yyyy-MM-dd`).
    *   `documentHeaderText`: Mapped from `INVOIC02/IDOC/E1EDKT1/TDOBJECT`.
    *   `externalIds`:
        *   `id`: Concatenates "S_" with `INVOIC02/IDOC/E1EDKT1/E1EDKT2/TDLINE` if `INVOIC02/IDOC/E1EDKT1/TDID` is "007".
        *   `type`: **Constant value "otherId"**.
        *   The entire `externalIds` node is also mapped from `INVOIC02/IDOC/EDI_DC40/DOCNUM` at a higher level, suggesting multiple `externalIds` might be generated or there's a conflict in the mapping design (typically, one field would source one external ID).
    *   `receiverSupplier`: Mapped from `INVOIC02/IDOC/E1EDKA1/LIFNR` if `INVOIC02/IDOC/E1EDKA1/PARVW` is "LF".
    *   `receiverCompanyCode`: **Constant value "1010"**.
    *   `receiverSystemId`: **Constant value "0M4IK2D"**.
    *   `taxes`:
        *   `percentage`: Mapped from `INVOIC02/IDOC/E1EDK04/MSATZ`.
        *   `currency`: Mapped from `INVOIC02/IDOC/E1EDK01/CURCY`.
        *   `amount`: Mapped from `INVOIC02/IDOC/E1EDK04/MWSBT`.
    *   `paymentInfos`:
        *   `paymentReference`: Mapped from `INVOIC02/IDOC/E1EDK01/ZTERM`.
    *   `invoiceParties`: This is a complex mapping with multiple `E1EDKA1` segments.
        *   `role`: **Conditional mapping** based on `INVOIC02/IDOC/E1EDKA1/PARVW` (Partner Function): "soldTo" if "AG", "from" if "LF".
        *   `name`: Mapped from `INVOIC02/IDOC/E1EDKA1/NAME1`.
        *   `providerAddresses/postalAddresses`:
            *   `postalCode`: Mapped from `INVOIC02/IDOC/E1EDKA1/PSTLZ`.
            *   `country`: Mapped from `INVOIC02/IDOC/E1EDKA1/LAND1`.
            *   `town`: Mapped from `INVOIC02/IDOC/E1EDKA1/ORT01`.
            *   `streetLines`: Mapped from `INVOIC02/IDOC/E1EDKA1/STRAS`.
            *   `scriptCodeKey`: **Constant value "Latn"**.
        *   `providerOtherInfos`:
            *   `domain`: **Constant value "vatID"**.
            *   `value`: Mapped from `INVOIC02/IDOC/E1EDK01/EIGENUINR`.
*   **Line Item Level Mappings (`lineItems` from `INVOIC02/IDOC/E1EDP01`):**
    *   `invoiceDocumentItem`: Mapped from `INVOIC02/IDOC/E1EDP01/POSEX`.
    *   `quantity`: Mapped from `INVOIC02/IDOC/E1EDP01/MENGE`.
    *   `externalUnitOfMeasure`: Mapped from `INVOIC02/IDOC/E1EDP01/MENEE`.
    *   `unitPrice`: Mapped from `INVOIC02/IDOC/E1EDP01/VPREI`.
    *   `description`: Mapped from `INVOIC02/IDOC/E1EDP01/E1EDP19/KTEXT`.
    *   `currency`: Mapped from `INVOIC02/IDOC/E1EDP01/CURCY`.
    *   `amount`: Mapped from `INVOIC02/IDOC/E1EDP01/NETWR`.
    *   `taxRate`: Mapped from `INVOIC02/IDOC/E1EDP01/E1EDP04/MSATZ`.
    *   `taxCountry`: **Constant value "DE"**.
    *   `poReferences`:
        *   `documentNumber`: Mapped from `INVOIC02/IDOC/E1EDP01/E1EDP02/BELNR` if `QUALF` is "001".
        *   `itemNumber`: Mapped from `INVOIC02/IDOC/E1EDP01/E1EDP02/ZEILE` if `QUALF` is "001".

### 9. Target API Schema Summary (SAP Ariba CIM - `CreateInvoiceStructuredPublicV1.json`)

The target system expects a `multipart/form-data` request with a JSON part named "invoice" (containing the structured invoice data) and one or more binary parts for attachments.

**Endpoint:** `POST /api/cim-create-invoice-structured/v1/`

**Request Body (`multipart/form-data`):**

*   **Part 1: `invoice` (JSON object - Schema: `Invoices`)**
    *   **Required Fields:** `origin`, `supplierInvoiceId`, `purpose`, `currency`, `grossAmount`, `documentDate`, `invoiceReceiptDate`, `attachments`.
    *   **Key Fields:**
        *   `origin`: `string` (enum: "04")
        *   `supplierInvoiceId`: `string` (max 255)
        *   `purpose`: `string` (enum: "I", "C", "3", "4")
        *   `currency`: `string` (ISO 4217, max 3)
        *   `grossAmount`: `string` or `number` (decimal)
        *   `documentDate`: `string` (date format)
        *   `invoiceReceiptDate`: `string` (date format)
        *   `documentHeaderText`: `string` (max 25)
        *   `externalIds`: `array` of objects (`id`, `type:"otherId"`)
        *   `paymentInfos`: `array` of objects (`netDueDate`, `paymentReference`)
        *   `taxes`: `array` of objects (`percentage` or (`amount` and `currency`), `description`)
        *   `lineItems`: `array` of objects (invoice details per line, including `poReferences`, `deliveryNoteReferences`, `sesReferences`)
        *   `invoiceParties`: `array` of objects (`role` (from/soldTo), `name`, `providerAddresses`, `providerOtherInfos`)
        *   `attachments`: `array` of objects (`fileName`, `documentType` (Invoice/Legal/Others), `isDefault`) - *This array is dynamically populated by `script8.groovy`.*
        *   `deliveryNoteReferences`: `array` of objects (`deliveryNoteNumber`)
        *   `sesReferences`: `array` of objects (`documentNumber`, `itemNumber`)
*   **Part 2+:** Binary attachments (e.g., `invoice_pdf`)
    *   `type`: `string`
    *   `format`: `binary`
    *   `Content-Disposition`: `form-data; name="invoice"; filename="invoice1.pdf"` (example)
    *   `Content-Type`: `application/pdf` (example)

This API creates one or more supplier invoices in SAP Ariba Central Invoice Management using the provided structured data and associated attachments.

---