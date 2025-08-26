This document provides a detailed overview of an SAP CPI (Cloud Platform Integration) iFlow designed for processing invoice data.

---

## iFlow: CIM Invoice Creation with Attachments

### 1. Overview

This SAP CPI iFlow is responsible for processing invoice data, including attachments, converting it into a structured JSON format, and sending it to an SAP Ariba Central Invoice Management (CIM) API endpoint using a `multipart/form-data` request. The iFlow is triggered on a scheduled basis. It also incorporates a robust exception handling mechanism to capture and enrich error details in case of processing failures.

**Key Features**:
*   **Scheduled Trigger**: The iFlow runs automatically at predefined intervals.
*   **Data Transformation**: Converts incoming XML IDoc data into a JSON structure required by the target system.
*   **Attachment Handling**: Extracts Base64 encoded PDF attachments from the incoming payload and sends them as binary parts in the outgoing `multipart/form-data` request.
*   **Dynamic JSON Modification**: Adjusts the JSON payload based on the number and type of attachments.
*   **Error Management**: Includes a dedicated exception subprocess to log detailed error information for various types of exceptions (HTTP, OData, SOAP).

### 2. Flow Details

The iFlow consists of the following steps:

1.  **Start Timer Event (`StartEvent_4665`)**
    *   **Description**: This is the entry point of the iFlow, configured to trigger it periodically.
    *   **Configuration**:
        *   **Schedule**: Simple Cron-based.
        *   **Cron Expression**: `0/5 * * ? * * *` (triggers every 5 minutes).
        *   **Time Zone**: UTC 0:00 (Greenwich Mean Time).

2.  **Content Modifier 3 (`CallActivity_758`)**
    *   **Description**: Initializes the message body with a predefined `StandardBusinessDocument` XML payload. This XML contains an embedded IDoc structure and multiple `Attachment` elements, some of which are Base64-encoded PDFs. This acts as a test or sample input.
    *   **Configuration**:
        *   **Body Type**: Constant.
        *   **Content**: An XML structure wrapping an `INVOIC02` IDoc and three `Attachment` nodes, with `Id` values `ID_1`, `ID_2`, and `ID_3`. `ID_1` contains XML, while `ID_2` and `ID_3` contain Base64-encoded PDF data.

3.  **ReadingTheCountOfAttachments (`CallActivity_761`)**
    *   **Description**: This Groovy script (`script6.groovy`) parses the incoming XML payload (from the previous step) to identify and extract specific PDF attachments.
    *   **Logic**: It finds `Attachment` elements with `Id` starting with `ID_2` or `ID_3` and `MimeType` as `application/pdf`. The Base64 content of these attachments is extracted and stored as message properties (e.g., `message.setProperty("ID_2", ...)`). It also sets `attachmentIds` and `attachmentCount` properties.

4.  **Filter 1 (`CallActivity_736`)**
    *   **Description**: Filters the message body to extract only the `application/xml` attachment from the `StandardBusinessDocument`.
    *   **Configuration**:
        *   **XPath Expression**: `//*[local-name()='Attachment' and @MimeType='application/xml']`
        *   **XPath Type**: String.

5.  **Base64 Decoder 1 (`CallActivity_739`)**
    *   **Description**: Decodes the Base64-encoded XML attachment (which is the current message body after the filter) into its original XML format.

6.  **Content Modifier 2 (`CallActivity_754`)**
    *   **Description**: Overwrites the current message body with a *different* static `INVOIC02` IDoc XML. This step makes the previous filtering and decoding of the XML attachment from the `StandardBusinessDocument` redundant if this static payload is always intended for mapping.
    *   **Configuration**:
        *   **Body Type**: Constant.
        *   **Content**: A static `INVOIC02` IDoc XML payload.
        *   **Header**: `SAP_ApplicationID` is created with an XPath value from `INVOIC02/IDOC/E1EDK02/BELNR` of *this* static payload.

7.  **Message Mapping 1 (`CallActivity_712`)**
    *   **Description**: Transforms the incoming `INVOIC02` IDoc XML (from Content Modifier 2) into the target JSON structure required by the SAP Ariba CIM API.
    *   **Configuration**:
        *   **Mapping Name**: `MM_SON_CIM`.
        *   **Mapping Type**: Message Mapping.

8.  **AddAttachmentNode (`CallActivity_4671`)**
    *   **Description**: This Groovy script (`script8.groovy`) dynamically modifies the JSON payload (from the mapping) to include an `attachments` array with filenames based on the attachments identified earlier.
    *   **Logic**: It reads the `attachmentIds` property (set by `script6.groovy`) and constructs the `attachments` array in the JSON, adding entries like `invoice1.pdf` and `Legal.pdf` with appropriate `documentType` values.

9.  **Invoice_payload (`CallActivity_728`)**
    *   **Description**: This Content Modifier stores the current JSON message body into a message property named `jsonbody`. The message body itself is not changed or explicitly cleared by this step's configuration, but the property `jsonbody` will now hold the mapped and modified JSON.

10. **MultiformDataIWithProperties (`CallActivity_4668`)**
    *   **Description**: This Groovy script (`script7.groovy`) constructs the final `multipart/form-data` request body by combining the JSON payload (from `jsonbody` property) and the previously extracted PDF attachments (from `ID_2` and `ID_3` properties).
    *   **Logic**: It builds a `multipart/form-data` string, with the JSON as one part (`name="invoice"`) and the decoded binary content of the PDFs (from `ID_2` and `ID_3` properties) as separate parts (`name="invoice"`, `filename="invoice1.pdf"`, `filename="Legal.pdf"`). The `Content-Type` header is also set.

11. **Request Reply 1 (`ServiceTask_4673`)**
    *   **Description**: Sends the constructed `multipart/form-data` request to the SAP Ariba CIM API endpoint.
    *   **Configuration**:
        *   **Adapter Type**: HTTP.
        *   **Method**: `POST`.
        *   **Address**: `https://cim-cf-eu10-subdomain-2025-2-27-191418.eu10.cim.cloud.sap/api/cim-create-invoice-structured/v1/`
        *   **Authentication**: OAuth2 Client Credentials.
        *   **Credential Name**: `ARIBACIM`.
        *   **Enable MPL Attachments**: `true`.
        *   **HTTP Should Send Body**: `false` (Note: For a `POST` request with a constructed body, this setting is unusual and might prevent the body from being sent. It is typically `true`.)
        *   **Throw Exception On Failure**: `true`.

12. **End Event (`EndEvent_2`)**
    *   **Description**: Marks the successful completion of the iFlow processing.

### 3. Exception Subprocess

The iFlow includes an exception subprocess (`SubProcess_745`) to handle errors gracefully:

1.  **Error Start 1 (`StartEvent_746`)**
    *   **Description**: Catches any exceptions that occur in the main integration process.

2.  **Build alert body (`CallActivity_749`)**
    *   **Description**: A Content Modifier that constructs a detailed alert message body using various iFlow properties and exception details.
    *   **Configuration**:
        *   **Body Type**: Expression.
        *   **Content**: Includes iFlow ID (`${camelId}`), purchase order ID (`${header.SAP_ApplicationID}`), message processing log ID (`${property.SAP_MessageProcessingLogID}`), current timestamp, HTTP status code (`${property.http.statusCode}`), exception message (`${exception.message}`), HTTP response body (`${property.http.responseBody}`), and stacktrace (`${exception.stacktrace}`).

3.  **Groovy Script 3 (`CallActivity_752`)**
    *   **Description**: This script (`script5.groovy`) analyzes the caught exception and extracts relevant error information (like HTTP response body/status code, OData, or SOAP fault details) to set them in the message body and properties. This standardizes the error message for subsequent logging or alerting.
    *   **Note**: This script is identical to `script4.groovy` which is not used in the main flow but is provided as a resource.

4.  **End 1 (`EndEvent_747`)**
    *   **Description**: Ends the exception handling process.

### 4. Scripts Documentation

#### 4.1 `script1.groovy` (ReadBinaryDataAndSetAttachment)

*   **Purpose**: Decodes Base64-encoded strings stored in message properties and adds them as binary attachments (specifically PDF files) to the message.
*   **Logic**:
    1.  Iterates through all existing message properties.
    2.  If a property's value is a non-empty string, it assumes the string is Base64 encoded.
    3.  Removes any whitespace (newlines, carriage returns, spaces) from the Base64 string.
    4.  Decodes the cleaned Base64 string into a byte array.
    5.  Creates a new attachment with the decoded bytes, specifying `application/pdf` as the MIME type.
    6.  Adds the attachment to the message, using the original property key as the base filename and appending `.pdf` (e.g., if property `ID_2` contains Base64, an attachment `ID_2.pdf` is created).
    7.  Logs the addition of each attachment to the message processing log.
    8.  The main message body is optionally cleared at the end (though its state is not directly used after this script in the current iFlow).
*   **Dependencies**: Expects message properties to contain Base64-encoded binary data (likely PDF).

#### 4.2 `script2.groovy` (Groovy Script 1)

*   **Purpose**: Extracts the content of the *first* attachment from the message and sets it as the main message body.
*   **Logic**:
    1.  Checks if the message contains any attachments.
    2.  If attachments are found, it retrieves the first one from the `attachments` map.
    3.  Reads the input stream of this attachment into a `ByteArrayOutputStream` to get its raw byte content.
    4.  Sets the collected byte array as the new message body.
    5.  If no attachments are found, it sets a warning XML message (`<warning>Attachment is missing</warning>`) as the body.
*   **Dependencies**: Requires the message to contain at least one attachment for normal, non-warning operation.

#### 4.3 `script3.groovy` (Groovy Script 2)

*   **Purpose**: Constructs a `multipart/form-data` payload for an HTTP POST request, combining a JSON payload (from a property) and a single binary file (from the message body, assumed to be a PDF).
*   **Logic**:
    1.  Retrieves the `jsonbody` message property, which is expected to hold the JSON content.
    2.  Defines a `multipart/form-data` boundary string.
    3.  Sets the message's `Content-Type` header to `multipart/form-data` with the specified boundary.
    4.  Retrieves the current message body as a byte array (expected to be the PDF binary content).
    5.  Constructs the `multipart/form-data` payload:
        *   First part: The JSON payload (`jsonbody` property) with `Content-Disposition: form-data; name="invoice"`.
        *   Second part: The PDF binary content (from the message body) with `Content-Disposition: form-data; name="invoice"; filename="invoice.pdf"` and `Content-Type: application/pdf`.
    6.  Combines these parts with the appropriate boundaries into a single byte array.
    7.  Sets this combined byte array as the new message body.
*   **Dependencies**: Relies on the `jsonbody` message property containing the JSON string and the message body containing the binary PDF data. This script's logic is superseded by `script7.groovy` in the actual flow for handling multiple PDFs.

#### 4.4 `script4.groovy` & `script5.groovy` (Groovy Script 3)

*   **Purpose**: Both scripts are identical. They extract detailed error information from the `CamelExceptionCaught` exchange property when an exception occurs in the iFlow, standardizing the error message in the body and properties.
*   **Logic**:
    1.  Checks if the `CamelExceptionCaught` property exists in the message properties.
    2.  If an exception is present, it identifies the type of exception:
        *   **`org.apache.camel.component.ahc.AhcOperationFailedException` (HTTP Error)**: Extracts the `responseBody` and `statusCode` from the exception object. These are set as the message body and as new properties (`http.responseBody`, `http.statusCode`).
        *   **`com.sap.gateway.core.ip.component.odata.exception.OsciException` (OData V2 Error)**: Uses the current message body and the `CamelHttpResponseCode` header as the `http.responseBody` and `http.statusCode` properties, respectively.
        *   **`org.apache.cxf.binding.soap.SoapFault` (SOAP Error)**: Extracts the SOAP fault detail, serializes it to XML, and sets this XML as the message body and `http.responseBody` property. It also attempts to get a `statusCode` from the exception.
*   **Dependencies**: Relies on the `CamelExceptionCaught` property being populated by the Apache Camel exchange in case of an error.

#### 4.5 `script6.groovy` (ReadingTheCountOfAttachments)

*   **Purpose**: Parses an XML message to find specific Base64-encoded PDF attachments, stores their content as message properties, and tracks their identifiers.
*   **Logic**:
    1.  Parses the incoming XML message body using `groovy.util.XmlSlurper`.
    2.  Searches for all `Attachment` elements within the XML that meet two criteria:
        *   Their `Id` attribute starts with either `ID_2` or `ID_3`.
        *   Their `MimeType` attribute is `application/pdf`.
    3.  For each matching `Attachment` element:
        *   Extracts its `Id` attribute value.
        *   Extracts the Base64-encoded content (the text node within the `Attachment` element).
        *   Sets a message property with the `Id` as the key and the Base64 content as the value.
        *   Adds the `Id` to an internal list (`idList`).
    4.  After processing all attachments, sets two additional message properties:
        *   `attachmentIds`: A comma-separated string of all extracted attachment `Id`s.
        *   `attachmentCount`: The total number of extracted attachments.
*   **Dependencies**: Requires the message body to be a valid XML document containing `<Attachment>` elements in the expected structure.

#### 4.6 `script7.groovy` (MultiformDataIWithProperties)

*   **Purpose**: Constructs a `multipart/form-data` HTTP request body by combining a JSON payload (from a property) and multiple binary PDF attachments (from specific properties set by `script6.groovy`). This script effectively replaces `script3.groovy`'s functionality for multiple attachments.
*   **Logic**:
    1.  Retrieves the `jsonbody` message property, which holds the main JSON content.
    2.  Defines a `multipart/form-data` boundary string.
    3.  Sets the message's `Content-Type` header to `multipart/form-data` with the specified boundary.
    4.  Initializes a `ByteArrayOutputStream` to build the complete multipart body.
    5.  **Adds the JSON part**: Appends the boundary, `Content-Disposition: form-data; name="invoice"`, and the `jsonPayload` (from the property) to the output stream.
    6.  **Adds the PDF parts**:
        *   Iterates through a hardcoded list of property keys (`ID_2`, `ID_3`) and corresponding filenames (`invoice1.pdf`, `Legal.pdf`).
        *   For each key, it retrieves the Base64 content from the message properties (expected to be set by `script6.groovy`).
        *   Removes whitespace from the Base64 string and decodes it to a byte array.
        *   Appends a new multipart section for each PDF, including `Content-Disposition`, `filename`, `Content-Type: application/pdf`, and the decoded PDF binary data.
    7.  Appends the final closing boundary (`--boundary--`).
    8.  Sets the entire content of the `ByteArrayOutputStream` as the new message body.
*   **Dependencies**: Relies on the `jsonbody` message property and specific properties (`ID_2`, `ID_3`) containing Base64-encoded PDF data.

#### 4.7 `script8.groovy` (AddAttachmentNode)

*   **Purpose**: Modifies a JSON payload to dynamically populate its `attachments` array based on the `attachmentIds` property, mapping specific filenames and document types.
*   **Logic**:
    1.  Parses the incoming JSON message body using `groovy.json.JsonSlurper`.
    2.  Retrieves the `attachmentIds` message property (a comma-separated string of IDs, populated by `script6.groovy`).
    3.  Splits the `attachmentIds` string into a list of individual IDs.
    4.  Creates a new `attachments` list.
    5.  Iterates through the `attachmentIds` list, populating the new `attachments` list:
        *   For the first attachment (`idx == 0`), it creates an entry `{fileName: "invoice1.pdf", documentType: "Invoice"}`.
        *   For subsequent attachments, it creates an entry `{fileName: "Legal.pdf", documentType: "Legal"}`.
    6.  Updates the `json.attachments` field of the parsed JSON object with this newly created list.
    7.  Converts the modified JSON object back into a JSON string using `groovy.json.JsonOutput.toJson()` and sets it as the new message body.
*   **Dependencies**: Requires the message body to be a valid JSON string and the `attachmentIds` message property to be present and correctly formatted.

### 5. Message Mapping Documentation

#### 5.1 `MM_SON_CIM.mmap`

*   **Description**: This message mapping transforms a source `INVOIC02` IDoc XML structure into a target JSON structure, conforming to the `CreateInvoiceStructuredPublicV1` OpenAPI specification for SAP Ariba Central Invoice Management.
*   **Source Structure (Input)**: Based on `INVOIC.INVOIC02.wsdl`
    *   Represents an SAP IDoc for Invoice/Billing documents.
    *   Contains standard IDoc segments like `EDI_DC40` (control record), `E1EDK01` (header general data), `E1EDKA1` (partner information), `E1EDK02` (reference data), `E1EDK03` (date segment), `E1EDK04` (taxes), `E1EDKT1` (text identification), `E1EDP01` (item general data), `E1EDP02` (item reference data), `E1EDP03` (item date segment), `E1EDP19` (item object identification), `E1EDP26` (item amount segment), `E1EDP04` (item taxes), and `E1EDS01` (summary segment).
*   **Target Structure (Output)**: Based on `CreateInvoiceStructuredPublicV1- CPICOMP.json`
    *   Represents a JSON object for creating supplier invoices with structured data in SAP Ariba CIM.
    *   Includes fields like `origin`, `supplierInvoiceId`, `purpose`, `currency`, `grossAmount`, `documentDate`, `invoiceReceiptDate`, `documentHeaderText`, `externalIds` (array), `paymentInfos` (array), `taxes` (array), `poReferences` (array), `lineItems` (array), `invoiceParties` (array), `attachments` (array), `receiverSystemId`, `receiverCompanyCode`, `receiverSupplier`.

#### 5.2 Key Mappings and Transformations

This section details significant field mappings and any applied transformations or conditional logic.

**Header Level Mappings**:

| Target Field              | Source Field / Value           | Transformation / Logic                                                                                                                                                                                                                                                                                                                                                                  |
| :------------------------ | :----------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `origin`                  | `04`                           | Constant value.                                                                                                                                                                                                                                                                                                                                                                         |
| `supplierInvoiceId`       | `INVOIC02/IDOC/E1EDK01/BELNR`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `purpose`                 | `INVOIC02/IDOC/E1EDK01/BSART`  | Conditional mapping: <br/>- If `BSART` is `INVO` (Invoice), maps to `I`.<br/>- Else if `BSART` is `CRME` (Credit Memo), maps to `C`.<br/>- Else if `BSART` is `SUBD` (Subsequent Debit), maps to `3`.<br/>- Else if `BSART` is `SUBC` (Subsequent Credit), maps to `4`.                                                                                                                      |
| `currency`                | `INVOIC02/IDOC/E1EDK01/CURCY`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `grossAmount`             | `INVOIC02/IDOC/E1EDS01/SUMME`  | Conditional mapping: Takes `SUMME` value only if `INVOIC02/IDOC/E1EDS01/SUMID` is `011`.                                                                                                                                                                                                                                                                                                            |
| `documentDate`            | `INVOIC02/IDOC/E1EDK03/DATUM`  | Conditional mapping: Takes `DATUM` value if `INVOIC02/IDOC/E1EDK03/IDDAT` is `012`. Date format transformed from `yyyyMMdd` to `yyyy-MM-dd`.                                                                                                                                                                                                                                                  |
| `invoiceReceiptDate`      | `currentDate`                  | Populated with the current date using the `currentDate` function, formatted as `yyyy-MM-dd`.                                                                                                                                                                                                                                                                                                    |
| `documentHeaderText`      | `INVOIC02/IDOC/E1EDKT1/TDOBJECT` | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `externalIds`             | `INVOIC02/IDOC/EDI_DC40/DOCNUM` | Direct mapping for the array item. This might conflict with `externalIds/id` and `externalIds/type` mappings if `EDI_DC40/DOCNUM` is the parent.                                                                                                                                                                                                                                                |
| `externalIds/id`          | `INVOIC02/IDOC/E1EDKT1/E1EDKT2/TDLINE` | Conditional mapping: Concatenates "S_" with `TDLINE` if `INVOIC02/IDOC/E1EDKT1/TDID` is `007`. (`removeContexts` and `concat` functions used).                                                                                                                                                                                                                                          |
| `externalIds/type`        | `otherId`                      | Constant value.                                                                                                                                                                                                                                                                                                                                                                                 |
| `receiverSupplier`        | `INVOIC02/IDOC/E1EDKA1/LIFNR`  | Conditional mapping: Takes `LIFNR` if `INVOIC02/IDOC/E1EDKA1/PARVW` is `LF` (Supplier). (`removeContexts` function used).                                                                                                                                                                                                                                                                           |
| `receiverCompanyCode`     | `1010`                         | Constant value.                                                                                                                                                                                                                                                                                                                                                                                 |
| `receiverSystemId`        | `0M4IK2D`                      | Constant value.                                                                                                                                                                                                                                                                                                                                                                                 |
| `taxes`                   | (Constant empty array)         | The `taxes` array is initially mapped to a constant empty array. However, its child elements `percentage`, `currency`, `amount` are mapped from source fields. CPI typically overrides the empty array if child elements are mapped.                                                                                                                                                          |
| `taxes/percentage`        | `INVOIC02/IDOC/E1EDK04/MSATZ`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `taxes/currency`          | `INVOIC02/IDOC/E1EDK01/CURCY`  | Direct mapping from header currency.                                                                                                                                                                                                                                                                                                                                                            |
| `taxes/amount`            | `INVOIC02/IDOC/E1EDK04/MWSBT`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `paymentInfos`            | (Constant empty array)         | The `paymentInfos` array is initially mapped to a constant empty array, with its child `paymentReference` mapped from source. Similar to `taxes`, this indicates potential for a single item to be created.                                                                                                                                                                                |
| `paymentInfos/paymentReference` | `INVOIC02/IDOC/E1EDK01/ZTERM`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `poReferences`            | (Constant empty array)         | The `poReferences` array is explicitly mapped to a constant empty array at the header level.                                                                                                                                                                                                                                                                                                      |
| `invoiceParties`          | `INVOIC02/IDOC/E1EDKA1`        | Parent element mapped from multiple `E1EDKA1` segments, with context removal.                                                                                                                                                                                                                                                                                                                   |
| `invoiceParties/role`     | `INVOIC02/IDOC/E1EDKA1/PARVW`  | Conditional mapping: <br/>- If `PARVW` is `AG`, maps to `soldTo`.<br/>- Else if `PARVW` is `LF`, maps to `from`.                                                                                                                                                                                                                                                                                        |
| `invoiceParties/name`     | `INVOIC02/IDOC/E1EDKA1/NAME1`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `invoiceParties/providerAddresses/postalAddresses/streetLines` | `INVOIC02/IDOC/E1EDKA1/STRAS`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `invoiceParties/providerAddresses/postalAddresses/town` | `INVOIC02/IDOC/E1EDKA1/ORT01`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `invoiceParties/providerAddresses/postalAddresses/region` | `INVOIC02/IDOC/E1EDKA1/REGIO`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `invoiceParties/providerAddresses/postalAddresses/postalCode` | `INVOIC02/IDOC/E1EDKA1/PSTLZ`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `invoiceParties/providerAddresses/postalAddresses/country` | `INVOIC02/IDOC/E1EDKA1/LAND1`  | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                 |
| `invoiceParties/providerAddresses/postalAddresses/scriptCodeKey` | `Latn`                         | Constant value.                                                                                                                                                                                                                                                                                                                                                                         |
| `invoiceParties/providerOtherInfos/domain` | `vatID`                        | Constant value.                                                                                                                                                                                                                                                                                                                                                                         |
| `invoiceParties/providerOtherInfos/value` | `INVOIC02/IDOC/E1EDK01/EIGENUINR` | Direct mapping using `CopyValue`.                                                                                                                                                                                                                                                                                                                                                               |
| `attachments`             | (Constant empty array)         | The `attachments` array is explicitly mapped to a constant empty array at the header level. This array is *later populated dynamically by `script8.groovy`*.                                                                                                                                                                                                                                         |

**Line Item Level Mappings (`lineItems` array maps from `E1EDP01` segments)**:

| Target Field                        | Source Field / Value                         | Transformation / Logic                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| :---------------------------------- | :------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `lineItems/invoiceDocumentItem`     | `INVOIC02/IDOC/E1EDP01/POSEX`                | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `lineItems/quantity`                | `INVOIC02/IDOC/E1EDP01/MENGE`                | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `lineItems/externalUnitOfMeasure`   | `INVOIC02/IDOC/E1EDP01/MENEE`                | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `lineItems/unitPrice`               | `INVOIC02/IDOC/E1EDP01/VPREI`                | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `lineItems/description`             | `INVOIC02/IDOC/E1EDP01/E1EDP19/KTEXT`        | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `lineItems/currency`                | `INVOIC02/IDOC/E1EDP01/CURCY`                | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `lineItems/amount`                  | `INVOIC02/IDOC/E1EDP01/NETWR`                | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `lineItems/taxRate`                 | `INVOIC02/IDOC/E1EDP01/E1EDP04/MSATZ`        | Direct mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `lineItems/taxCountry`              | `DE`                                         | Constant value.                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `lineItems/poReferences/documentNumber` | `INVOIC02/IDOC/E1EDP01/E1EDP02/BELNR`        | Conditional mapping: If `INVOIC02/IDOC/E1EDP01/E1EDP02/QUALF` is `001`, then maps `BELNR`. The mapping uses `SplitByValue` with `useOneAsMany` functions, which can indicate handling of multiple `E1EDP02` segments per `E1EDP01` or handling contexts carefully for mapping to multiple PO references for a line item.                                                                                                                                                        |
| `lineItems/poReferences/itemNumber` | `INVOIC02/IDOC/E1EDP01/E1EDP02/ZEILE`        | Conditional mapping: If `INVOIC02/IDOC/E1EDP01/E1EDP02/QUALF` is `001`, then maps `ZEILE`. Similar context handling as `documentNumber`.                                                                                                                                                                                                                                                                                                                             |

---

### 6. Potential Inconsistencies / Notes

*   **Redundant XML Processing**: Content Modifier 3 sets an SBDH XML containing an IDoc and PDFs. The flow then filters and decodes the XML attachment. However, Content Modifier 2 immediately *overwrites* this processed XML with a *different* static IDoc XML. This makes the initial XML filtering/decoding redundant. It's possible the intent was to use the initial SBDH as the *only* source of the IDoc payload, in which case Content Modifier 2 should be removed.
*   **HTTP Body Sending**: The `Request Reply 1` step has `httpShouldSendBody` set to `false`. For a `POST` request where a `multipart/form-data` body is explicitly constructed by `script7.groovy`, this setting is highly unusual and would typically prevent the constructed body from being sent, leading to an empty request payload. This is a likely misconfiguration that would prevent the iFlow from functioning correctly.
*   **Static vs. Dynamic Attachments**: `MM_SON_CIM.mmap` maps `attachments` to a constant empty array, but `script8.groovy` later dynamically populates this array in the JSON. This is a common pattern for dynamic content, but it's important to understand the sequence.
*   **Script Re-use**: `script4.groovy` and `script5.groovy` are identical. This indicates potential for simplification in resource management.