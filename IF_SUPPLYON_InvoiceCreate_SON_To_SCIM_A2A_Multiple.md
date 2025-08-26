This document provides a comprehensive overview of an SAP CPI Iflow designed to process inbound invoice data from an IDOC format, including attachments, transform it into a structured JSON payload, and send it to an external SAP Ariba Central Invoice Management (CIM) API.

---

## **Iflow: Process Inbound IDOC with Attachments to SAP Ariba CIM**

This Iflow automates the processing of invoice IDOCs, extracts associated attachments (PDFs), transforms the IDOC data into a JSON format required by SAP Ariba Central Invoice Management (CIM), dynamically integrates attachment metadata into the JSON, constructs a multipart/form-data payload, and sends it to the CIM API. It also includes robust error handling to capture and log exceptions.

### **1. General Information**

*   **Integration Flow Name:** Collaboration_1Default Collaboration
*   **Description:** Processes invoice IDOCs with attachments, transforms to JSON, and sends to SAP Ariba CIM.
*   **Logging:** All events are logged.
*   **Error Handling:** Custom exception subprocess for detailed error capture.

### **2. Participants**

*   **Sender (Participant_741):** `EndpointSenderSender1` (implicitly triggered by a Timer Start Event).
*   **Receiver (Participant_2):** `EndpointRecevierReceiver` (an external HTTP endpoint, specifically the SAP Ariba CIM API).
*   **Integration Process (Participant_Process_1):** `Process_1Integration Process`.

### **3. Flow Steps and Configuration**

The integration process `Process_1Integration Process` executes the following sequence of steps:

1.  **Start Timer (StartEvent_4665)**
    *   **Name:** Start Timer 1
    *   **Type:** Timer Start Event
    *   **Schedule:** Configured with a cron expression to trigger the Iflow periodically every 5 minutes (`minute=0/5`, `hour=*`, `day_of_month=?`, `month=*`, `dayOfWeek=*`, `year=*`).
    *   **Purpose:** Initiates the integration flow at scheduled intervals.

2.  **Content Modifier (CallActivity_758)**
    *   **Name:** Content Modifier 3
    *   **Body Type:** `constant`
    *   **Content:** Sets the initial message body with a `StandardBusinessDocument` XML structure. This XML contains a Base64-encoded IDOC (`Attachment Id="ID_1" MimeType="application/xml"`) and two Base64-encoded PDF attachments (`Attachment Id="ID_2"` and `Attachment Id="ID_3"` both `MimeType="application/pdf"`).
    *   **Purpose:** Provides the initial inbound invoice data, including both the core IDOC (as an XML attachment) and supplementary PDF documents (as PDF attachments), all embedded in Base64 within a single XML message.

3.  **Groovy Script (CallActivity_761)**
    *   **Name:** ReadingTheCountOfAttachments
    *   **Script:** `script6.groovy`
    *   **Purpose:** Extracts the Base64-encoded PDF attachments from the incoming `StandardBusinessDocument` XML and stores their content and metadata as message properties for later use.
        *   **Logic:**
            *   Parses the XML message body.
            *   Finds `Attachment` elements with `@Id` starting with `ID_2` or `ID_3` and `MimeType='application/pdf'`.
            *   For each found attachment, it decodes the Base64 content and sets it as a message property, using the attachment `Id` (e.g., `ID_2`, `ID_3`) as the property key.
            *   It also collects all such attachment `Id`s into a list and sets two additional message properties: `attachmentIds` (a comma-separated string of IDs) and `attachmentCount` (the total number of found attachments).

4.  **Filter (CallActivity_736)**
    *   **Name:** Filter 1
    *   **XPath Expression:** `//*[local-name()='Attachment' and @MimeType='application/xml']`
    *   **Purpose:** Filters the `StandardBusinessDocument` XML to retain only the `Attachment` element that contains the Base64-encoded IDOC XML (`ID_1`). This isolates the core invoice data from the PDF attachments which have already been stored as properties.

5.  **Base64 Decoder (CallActivity_739)**
    *   **Name:** Base64 Decoder 1
    *   **Decoder Type:** `Base64 Decode`
    *   **Purpose:** Decodes the Base64-encoded content of the remaining XML attachment (which is the message body after the filter step). The message body now contains the plain XML IDOC.

6.  **Content Modifier (CallActivity_754)**
    *   **Name:** Content Modifier 2
    *   **Headers:** Sets a dynamic header `SAP_ApplicationID` by extracting the value from the XPath expression `INVOIC02/IDOC/E1EDK02/BELNR` from the current message body (the decoded IDOC XML).
    *   **Body Type:** `constant`
    *   **Content:** Overwrites the current message body with a *predefined constant XML IDOC payload*.
    *   **Purpose:** While the previous step decoded an IDOC from the initial payload, this step explicitly sets a fixed IDOC structure as the main message body. This ensures consistent input for the subsequent Message Mapping, potentially for demonstration or testing purposes, or if the initial IDOC is merely a container for different attachment types while the core data is standardized.

7.  **Message Mapping (CallActivity_712)**
    *   **Name:** Message Mapping 1
    *   **Mapping:** `MM_SON_CIM` (located at `src/main/resources/mapping/MM_SON_CIM.mmap`)
    *   **Source:** `INVOIC.INVOIC02.wsdl` (IDOC structure)
    *   **Target:** `CreateInvoiceStructuredPublicV1- CPICOMP.json` (JSON structure for SAP Ariba CIM)
    *   **Purpose:** Transforms the inbound IDOC XML data into a JSON payload compliant with the SAP Ariba Central Invoice Management API.
    *   **Key Field Mappings:**
        *   `origin`: Constant `04`.
        *   `supplierInvoiceId`: Mapped from `/INVOIC02/IDOC/E1EDK01/BELNR`.
        *   `purpose`: Conditionally mapped from `/INVOIC02/IDOC/E1EDK01/BSART` (INVO -> I, CRME -> C, SUBD -> 3, SUBC -> 4).
        *   `currency`: Mapped from `/INVOIC02/IDOC/E1EDK01/CURCY`.
        *   `grossAmount`: Mapped from `/INVOIC02/IDOC/E1EDS01/SUMME` (where `SUMID` is `011`).
        *   `documentDate`: Mapped from `/INVOIC02/IDOC/E1EDK03/DATUM` (where `IDDAT` is `012`), transformed from `yyyyMMdd` to `yyyy-MM-dd`.
        *   `invoiceReceiptDate`: Set to the current date (`yyyy-MM-dd`).
        *   `documentHeaderText`: Mapped from `/INVOIC02/IDOC/E1EDKT1/TDOBJECT`.
        *   `lineItems`: Mapped from `/INVOIC02/IDOC/E1EDP01` segment.
            *   `invoiceDocumentItem`: Mapped from `/E1EDP01/POSEX`.
            *   `currency`: Mapped from `/E1EDP01/CURCY`.
            *   `amount`: Mapped from `/E1EDP01/NETWR`.
            *   `externalUnitOfMeasure`: Mapped from `/E1EDP01/MENEE`.
            *   `unitPrice`: Mapped from `/E1EDP01/VPREI`.
            *   `quantity`: Mapped from `/E1EDP01/MENGE`.
            *   `description`: Mapped from `/E1EDP01/E1EDP19/KTEXT`.
            *   `taxRate`: Mapped from `/E1EDP01/E1EDP04/MSATZ`.
            *   `taxCountry`: Constant `DE`.
            *   `poReferences/documentNumber`: Conditionally mapped from `/E1EDP01/E1EDP02/BELNR` (where `QUALF` is `001`).
            *   `poReferences/itemNumber`: Conditionally mapped from `/E1EDP01/E1EDP02/ZEILE` (where `QUALF` is `001`).
        *   `paymentInfos/paymentReference`: Mapped from `/INVOIC02/IDOC/E1EDK01/ZTERM`.
        *   `taxes/percentage`: Mapped from `/INVOIC02/IDOC/E1EDK04/MSATZ`.
        *   `taxes/currency`: Mapped from `/INVOIC02/IDOC/E1EDK01/CURCY`.
        *   `taxes/amount`: Mapped from `/INVOIC02/IDOC/E1EDK04/MWSBT`.
        *   `externalIds/id`: Concatenated value from `S_` and `/INVOIC02/IDOC/E1EDKT1/E1EDKT2/TDLINE` (where `TDID` is `007`).
        *   `externalIds/type`: Constant `otherId`.
        *   `receiverSystemId`: Constant `0M4IK2D`.
        *   `receiverCompanyCode`: Constant `1010`.
        *   `receiverSupplier`: Conditionally mapped from `/INVOIC02/IDOC/E1EDKA1/LIFNR` (where `PARVW` is `LF`).
        *   `invoiceParties`: Mapped from multiple `/INVOIC02/IDOC/E1EDKA1` segments.
            *   `role`: Conditionally mapped to `soldTo` (if `PARVW`='AG') or `from` (if `PARVW`='LF').
            *   `name`: Mapped from `/INVOIC02/IDOC/E1EDKA1/NAME1`.
            *   `providerAddresses/postalAddresses/streetLines`: Mapped from `/INVOIC02/IDOC/E1EDKA1/STRAS`.
            *   `providerAddresses/postalAddresses/town`: Mapped from `/INVOIC02/IDOC/E1EDKA1/ORT01`.
            *   `providerAddresses/postalAddresses/region`: Mapped from `/INVOIC02/IDOC/E1EDKA1/REGIO`.
            *   `providerAddresses/postalAddresses/postalCode`: Mapped from `/INVOIC02/IDOC/E1EDKA1/PSTLZ`.
            *   `providerAddresses/postalAddresses/country`: Mapped from `/INVOIC02/IDOC/E1EDKA1/LAND1`.
            *   `providerAddresses/postalAddresses/scriptCodeKey`: Constant `Latn`.
            *   `providerOtherInfos/domain`: Constant `vatID`.
            *   `providerOtherInfos/value`: Mapped from `/INVOIC02/IDOC/E1EDK01/EIGENUINR`.

8.  **Groovy Script (CallActivity_4671)**
    *   **Name:** AddAttachmentNode
    *   **Script:** `script8.groovy`
    *   **Purpose:** Dynamically modifies the `attachments` array within the JSON payload (generated by the Message Mapping) based on the number of PDF attachments identified earlier.
        *   **Logic:**
            *   Parses the current JSON message body.
            *   Retrieves the `attachmentIds` property (a comma-separated string of IDs like `ID_2,ID_3`).
            *   For each ID, it adds an attachment object to the JSON's `attachments` array.
            *   The first attachment is set with `fileName: "invoice1.pdf"` and `documentType: "Invoice"`.
            *   Subsequent attachments are set with `fileName: "Legal.pdf"` and `documentType: "Legal"`. (Note: This is a fixed logic; if more than two PDFs are present, they will all be named "Legal.pdf").
            *   The modified JSON is set back as the message body.

9.  **Content Modifier (CallActivity_728)**
    *   **Name:** Invoice_payload
    *   **Properties:** Sets a message property `jsonbody` with the entire current message body (the dynamically updated JSON payload).
    *   **Purpose:** Preserves the final JSON payload in a property before the message body is used to construct the multipart/form-data.

10. **Groovy Script (CallActivity_4668)**
    *   **Name:** MultiformDataIWithProperties
    *   **Script:** `script7.groovy`
    *   **Purpose:** Constructs the final HTTP request body as `multipart/form-data`, combining the JSON payload and the previously stored PDF attachments.
        *   **Logic:**
            *   Sets the `Content-Type` header to `multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW`.
            *   Retrieves the JSON payload from the `jsonbody` property.
            *   Retrieves the Base64-encoded PDF attachments from properties `ID_2` and `ID_3`.
            *   Constructs a `ByteArrayOutputStream` to build the multipart payload:
                *   Appends the JSON part (`name="invoice"`) using the `jsonbody` content.
                *   For `ID_2`, it decodes the Base64 content and appends it as a file part (`name="invoice"; filename="invoice1.pdf"; Content-Type: application/pdf`).
                *   For `ID_3`, it decodes the Base64 content and appends it as another file part (`name="invoice"; filename="Legal.pdf"; Content-Type: application/pdf`).
            *   Sets the assembled byte array as the new message body.

11. **Request Reply (ServiceTask_4673)**
    *   **Name:** Request Reply 1
    *   **Adapter Type:** HTTP Receiver
    *   **Method:** `POST`
    *   **Address:** `https://cim-cf-eu10-subdomain-2025-2-27-191418.eu10.cim.cloud.sap/api/cim-create-invoice-structured/v1/`
    *   **Authentication:** `OAuth2 Client Credentials` using credential `ARIBACIM`.
    *   **Other Properties:** `enableMPLAttachments=true`, `httpRequestTimeout=60000`.
    *   **Purpose:** Sends the constructed `multipart/form-data` payload containing the JSON invoice data and PDF attachments to the SAP Ariba Central Invoice Management API.

12. **End (EndEvent_2)**
    *   **Name:** End
    *   **Purpose:** Terminates the main integration process upon successful execution.

### **4. Exception Subprocess**

The `SubProcess_745` (Exception Subprocess 1) is triggered if any error occurs during the main integration flow.

1.  **Error Start Event (StartEvent_746)**
    *   **Name:** Error Start 1
    *   **Purpose:** Catches any exception thrown in the main integration flow.

2.  **Content Modifier (CallActivity_749)**
    *   **Name:** Build alret boday
    *   **Body Type:** `expression`
    *   **Content:** Formats an alert message string using various message headers and properties to provide detailed error context.
        *   `Iflow: ${camelId}`
        *   `PURCHASE_ORDER:${header.SAP_ApplicationID}`
        *   `SAP_MessageProcessingLogID: ${property.SAP_MessageProcessingLogID}`
        *   `datetime now: ${date:now:yyyy-MM-dd'T'HH:mm:ss.SSS} (default UTC)`
        *   `datetime now: ${date-with-timezone:now:Asia/Calcutta:yyyy-MM-dd'T'HH:mm:ss.SSS} (with timezone Asia/Calcutta)`
        *   `http.statusCode: ${property.http.statusCode}`
        *   `exception.message: ${exception.message}`
        *   `http.responseBody: ${property.http.responseBody}`
        *   `message.stacktrace: ${exception.stacktrace}`
    *   **Purpose:** Gathers comprehensive error information into a human-readable format for alerting or logging.

3.  **Groovy Script (CallActivity_752)**
    *   **Name:** Groovy Script 3
    *   **Script:** `script5.groovy` (identical to `script4.groovy` in resources)
    *   **Purpose:** Processes the caught `CamelExceptionCaught` to extract specific error details (like HTTP status code and response body) and makes them available as message properties.
        *   **Logic:**
            *   Checks if `CamelExceptionCaught` property exists.
            *   If it's an `AhcOperationFailedException` (HTTP error), it sets the exception's response body and status code to message body and properties (`http.responseBody`, `http.statusCode`).
            *   If it's an `OsciException` (OData V2 error), it sets the current message body and `CamelHttpResponseCode` to properties.
            *   If it's a `SoapFault` (SOAP error), it extracts and serializes the fault detail to XML, setting it to message body and properties.
    *   **Usage Context:** This script ensures that the alert body (prepared in the previous step) has standardized `http.statusCode` and `http.responseBody` values, regardless of the specific exception type.

4.  **End (EndEvent_747)**
    *   **Name:** End 1
    *   **Purpose:** Terminates the exception subprocess after processing the error.

### **5. Resource Documentation**

#### **5.1 Groovy Scripts**

*   **`script1.groovy` (ReadBinaryDataAndSetAttachment)**
    *   **Purpose:** Reads Base64-encoded content from message properties, decodes it, and adds it as a binary attachment to the message.
    *   **Logic:** Iterates through message properties. If a property value is a non-empty string, it's assumed to be Base64-encoded, decoded, and added as an `application/pdf` attachment with a filename based on the property key. The message body is then cleared.
    *   **Usage in Iflow:** This script is **not actively used** in the described integration flow.

*   **`script2.groovy` (Groovy Script 1)**
    *   **Purpose:** Reads the first attachment of a message and sets its content as the message body.
    *   **Logic:** Checks if attachments exist. If so, it takes the first attachment's `InputStream`, reads its content into a `ByteArrayOutputStream`, and sets this byte array as the message body.
    *   **Usage in Iflow:** This script is **not actively used** in the described integration flow.

*   **`script3.groovy` (Groovy Script 2)**
    *   **Purpose:** Constructs a multipart/form-data payload combining a JSON payload (from a property) and a single PDF file (from the message body).
    *   **Logic:** Sets the `Content-Type` header for multipart. Combines a JSON string (from `jsonbody` property) as the first part and the current message body (expected to be PDF binary) as the second part, defining their respective `Content-Disposition` and `Content-Type` headers within the multipart structure.
    *   **Usage in Iflow:** This script is **not actively used** in the described integration flow.

*   **`script4.groovy` (Error handling logic)**
    *   **Purpose:** A utility script to extract and standardize error details from a `CamelExceptionCaught` property, specifically handling HTTP, OData V2, and SOAP fault exceptions.
    *   **Logic:** Inspects the class of `CamelExceptionCaught`. For `AhcOperationFailedException`, it extracts `responseBody` and `statusCode`. For `OsciException`, it uses the current message body and `CamelHttpResponseCode`. For `SoapFault`, it serializes the fault detail to XML. These extracted details are set as the message body and `http.responseBody`/`http.statusCode` properties.
    *   **Usage in Iflow:** This script is effectively used through `script5.groovy`, which is identical.

*   **`script5.groovy` (Groovy Script 3)**
    *   **Purpose:** Identical to `script4.groovy`. Its role is to extract and standardize error details from `CamelExceptionCaught` for use in the exception handling flow.
    *   **Logic:** Same as `script4.groovy`.
    *   **Usage in Iflow:** Used in the `CallActivity_752` step within the exception subprocess.

*   **`script6.groovy` (ReadingTheCountOfAttachments)**
    *   **Purpose:** Extracts Base64-encoded PDF attachments from an XML message and stores them as message properties, along with a count and list of their IDs.
    *   **Logic:** Parses the XML message body. Uses `XmlSlurper` to find `<Attachment>` elements that are `application/pdf` and whose `Id` attribute starts with `ID_2` or `ID_3`. For each found attachment, it Base64 decodes its content and sets it as a message property (key: `@Id` attribute, value: decoded binary string). It also creates `attachmentIds` (a comma-separated string of the extracted IDs) and `attachmentCount` properties.
    *   **Usage in Iflow:** Used in the `CallActivity_761` step.

*   **`script7.groovy` (MultiformDataIWithProperties)**
    *   **Purpose:** Constructs a `multipart/form-data` payload containing a JSON part and multiple PDF attachments, dynamically taking attachments from message properties.
    *   **Logic:** Sets the `Content-Type` header with a specific boundary. Retrieves the JSON payload from the `jsonbody` property. It then defines a fixed list of keys (`ID_2`, `ID_3`) and corresponding filenames (`invoice1.pdf`, `Legal.pdf`). For each key, it retrieves the Base64-encoded content from the message properties, decodes it, and appends it as a `form-data` part to a `ByteArrayOutputStream`. The JSON part is added first, followed by the PDF parts. The final byte array from the `OutputStream` is set as the message body.
    *   **Usage in Iflow:** Used in the `CallActivity_4668` step.

*   **`script8.groovy` (AddAttachmentNode)**
    *   **Purpose:** Modifies the JSON payload to dynamically populate its `attachments` array based on the `attachmentIds` message property.
    *   **Logic:** Parses the JSON message body. Retrieves the `attachmentIds` property (e.g., "ID_2,ID_3"). It then constructs a new `attachments` array for the JSON object:
        *   If `ID_2` is present, it adds an entry `[fileName: "invoice1.pdf", documentType: "Invoice"]`.
        *   If `ID_3` is present, it adds an entry `[fileName: "Legal.pdf", documentType: "Legal"]`.
        *   (Note: This logic is fixed for two specific attachments. If `attachmentIds` contains more or different IDs, they won't be explicitly handled by their IDs or dynamic filenames.)
    *   **Usage in Iflow:** Used in the `CallActivity_4671` step.

#### **5.2 Message Mappings**

*   **`MM_SON_CIM.mmap`**
    *   **Purpose:** Transforms an incoming SAP IDOC (INVOIC02) XML structure into a JSON payload required by the SAP Ariba Central Invoice Management API.
    *   **Source Structure:** WSDL definition for `INVOIC.INVOIC02` representing the SAP IDOC.
    *   **Target Structure:** JSON schema defined by `CreateInvoiceStructuredPublicV1- CPICOMP.json` for the SAP Ariba CIM API.
    *   **Key Transformations:**
        *   **Header Level:**
            *   `origin`: Constant `04`.
            *   `supplierInvoiceId`: From `IDOC/E1EDK01/BELNR`.
            *   `purpose`: Conditional mapping based on `IDOC/E1EDK01/BSART` (e.g., "INVO" maps to "I").
            *   `currency`: From `IDOC/E1EDK01/CURCY`.
            *   `grossAmount`: From `IDOC/E1EDS01/SUMME` (where `SUMID` is "011").
            *   `documentDate`: From `IDOC/E1EDK03/DATUM` (where `IDDAT` is "012"), formatted to `yyyy-MM-dd`.
            *   `invoiceReceiptDate`: Current date.
            *   `documentHeaderText`: From `IDOC/E1EDKT1/TDOBJECT`.
            *   `externalIds/id`: Concatenation of "S_" and `IDOC/E1EDKT1/E1EDKT2/TDLINE` (where `TDID` is "007").
            *   `externalIds/type`: Constant "otherId".
            *   `receiverSystemId`: Constant "0M4IK2D".
            *   `receiverCompanyCode`: Constant "1010".
            *   `receiverSupplier`: From `IDOC/E1EDKA1/LIFNR` (where `PARVW` is "LF").
            *   `taxes`: Mapped from `IDOC/E1EDK04` segments.
                *   `percentage`: From `IDOC/E1EDK04/MSATZ`.
                *   `currency`: From `IDOC/E1EDK01/CURCY`.
                *   `amount`: From `IDOC/E1EDK04/MWSBT`.
            *   `paymentInfos/paymentReference`: From `IDOC/E1EDK01/ZTERM`.
            *   `invoiceParties`: Mapped from multiple `IDOC/E1EDKA1` segments.
                *   `role`: "soldTo" (if `PARVW`="AG") or "from" (if `PARVW`="LF").
                *   `name`: From `IDOC/E1EDKA1/NAME1`.
                *   `providerAddresses/postalAddresses/*`: Mapped from corresponding fields in `IDOC/E1EDKA1` (e.g., `PSTLZ`, `LAND1`, `ORT01`, `STRAS`).
                *   `providerAddresses/postalAddresses/scriptCodeKey`: Constant "Latn".
                *   `providerOtherInfos/domain`: Constant "vatID".
                *   `providerOtherInfos/value`: From `IDOC/E1EDK01/EIGENUINR`.
        *   **Line Item Level (`lineItems`):** Mapped from `IDOC/E1EDP01` segments.
            *   `invoiceDocumentItem`: From `IDOC/E1EDP01/POSEX`.
            *   `currency`: From `IDOC/E1EDP01/CURCY`.
            *   `amount`: From `IDOC/E1EDP01/NETWR`.
            *   `externalUnitOfMeasure`: From `IDOC/E1EDP01/MENEE`.
            *   `unitPrice`: From `IDOC/E1EDP01/VPREI`.
            *   `quantity`: From `IDOC/E1EDP01/MENGE`.
            *   `description`: From `IDOC/E1EDP01/E1EDP19/KTEXT`.
            *   `taxRate`: From `IDOC/E1EDP01/E1EDP04/MSATZ`.
            *   `taxCountry`: Constant "DE".
            *   `poReferences/documentNumber`: From `IDOC/E1EDP01/E1EDP02/BELNR` (where `QUALF` is "001").
            *   `poReferences/itemNumber`: From `IDOC/E1EDP01/E1EDP02/ZEILE` (where `QUALF` is "001").

---