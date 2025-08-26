## Technical Specification Document: SF_To_SFTP_Iflow

### 1. Overview

This document provides a technical specification for the SAP Cloud Integration (CPI) Iflow named `SF_To_SFTP_Iflow`. The purpose of this Iflow is to receive employee data via an HTTPS call from a SuccessFactors (SF) system, log the incoming payload for auditing, transform the incoming JSON data into an XML format, and then securely transmit the resulting XML file to an SFTP server.

### 2. General Iflow Properties

| Property                 | Value                                          | Description                                                                                             |
| :----------------------- | :--------------------------------------------- | :------------------------------------------------------------------------------------------------------ |
| **Log Level**            | All events                                     | All message processing events will be logged.                                                           |
| **Trace Level**          | None                                           | Message content tracing is disabled.                                                                    |
| **Retry on Exception**   | No                                             | Iflow will not retry message processing on an exception.                                                |
| **Return Exception**     | false                                          | Exceptions will not be returned to the sender.                                                          |
| **HTTP Session Handling**| None                                           | No specific HTTP session handling is configured.                                                        |
| **Component Version**    | 1.1                                            | Version of the Iflow component.                                                                         |

### 3. Sender Channel Configuration

| Property           | Value                        | Description                                                                 |
| :----------------- | :--------------------------- | :-------------------------------------------------------------------------- |
| **System**         | SF                           | Represents the sender system (likely SuccessFactors).                       |
| **Adapter Type**   | HTTPS                        | The Iflow exposes an HTTPS endpoint to receive messages.                    |
| **Direction**      | Sender                       | This channel is configured to receive messages.                             |
| **URL Path**       | `/sf_to_sftp`                | The endpoint path where the Iflow will listen for incoming messages.        |
| **Authentication** | Role-Based                   | Authentication is based on predefined user roles.                           |
| **User Role**      | `ESBMessaging.send`          | Users must have this role to send messages to the endpoint.                 |
| **Maximum Body Size**| 40 MB                      | Maximum allowed size for the message body.                                  |
| **XSRF Protection**| Disabled                     | Cross-Site Request Forgery protection is disabled.                          |
| **Client Certificates**| N/A                        | No specific client certificates are configured for mutual authentication.   |

### 4. Integration Process Steps

The integration process involves the following steps:

1.  **Start Event - `Start` (Message Start Event)**
    *   This step initiates the Iflow upon receiving an inbound message via the configured HTTPS sender adapter.

2.  **Groovy Script - `Log Payload`**
    *   **Resource:** `Log_Payload.gsh`
    *   **Description:** This step executes a Groovy script to log the entire incoming message payload as a plain text attachment in the message monitor. This is primarily used for auditing and debugging purposes.

3.  **Content Modifier - `JSON to XML Converter`**
    *   **Type:** JSON to XML Converter
    *   **Add XML Root Element:** True
    *   **Additional Root Element Name:** `root`
    *   **Use Namespaces:** False
    *   **JSON Namespace Separator:** `:`
    *   **Description:** This step converts the JSON formatted message body into an XML format. A new root element named `root` will be added to the XML output. No namespaces are used in the conversion.

4.  **End Event - `End` (Message End Event)**
    *   This step marks the end of the integration process, passing the processed message to the SFTP receiver adapter for outbound transmission.

### 5. Receiver Channel Configuration

| Property              | Value                                     | Description                                                                     |
| :-------------------- | :---------------------------------------- | :------------------------------------------------------------------------------ |
| **System**            | SFTP                                      | Represents the receiver SFTP system.                                            |
| **Adapter Type**      | SFTP                                      | The Iflow uses an SFTP adapter to send messages.                                |
| **Direction**         | Receiver                                  | This channel is configured to send messages.                                    |
| **Host**              | `solexsftpdev.voracloud.sapcloud.io`      | The hostname of the target SFTP server.                                         |
| **Directory**         | `/data/BTPIS/DBRSumitomo/output`          | The remote directory on the SFTP server where files will be written.            |
| **File Name**         | `SF_Data.xml`                             | The name of the file to be created on the SFTP server.                          |
| **Authentication**    | User Name/Password                        | Authentication is performed using a username and password.                      |
| **Credential Name**   | `sftpuser`                                | The alias for the User Credentials artifact stored in CPI's security material.  |
| **File Exists**       | Override                                  | If a file with the same name already exists, it will be overwritten.            |
| **Connect Timeout**   | 10000 ms                                  | Timeout for establishing a connection to the SFTP server.                       |
| **Maximum Reconnect Attempts**| 3                             | Number of attempts to reconnect if the initial connection fails.                |
| **Reconnect Delay**   | 1000 ms                                   | Delay in milliseconds between reconnection attempts.                            |
| **Use Temp File**     | False                                     | Files are written directly to the target filename, not via a temporary file.    |
| **Proxy Type**        | None                                      | No proxy server is used for the SFTP connection.                                |
| **SFTP Security**     | Enabled                                   | Secure File Transfer Protocol is enabled.                                       |
| **Flatten Directory** | Disabled                                  | Directory structure is maintained (not flattened).                              |
| **Auto Create Directory**| Enabled                                | The SFTP adapter will automatically create the target directory if it doesn't exist. |

### 6. Resources

The following resources are attached to the Iflow:

#### 6.1 Scripts

*   **`Log_Payload.gsh` (Groovy Script)**
    *   **Purpose:** This script is used in the `Log Payload` step of the integration process. It takes the current message body, converts it to a String, and then adds it as an attachment named "Event Payload" to the message log, with content type "text/plain". This helps in inspecting the payload at that stage of processing.

*   **`UDF.groovy` (Groovy Script)**
    *   **Purpose:** This script contains generic utility functions for interacting with message headers and properties within the CPI message context.
    *   **Functions:**
        *   `getheader(String header_name, MappingContext context)`: Retrieves the value of a specified header.
        *   `getProperty(String property_name, MappingContext context)`: Retrieves the value of a specified exchange property.
        *   `setHeader(String header_name, String header_value, MappingContext context)`: Sets a specified header with a given value.
        *   `setProperty(String property_name, String property_value, MappingContext context)`: Sets a specified exchange property with a given value.
    *   **Note:** This script is defined as a `UsedFuncLib` within the `Map_SFSF_JSON_to_SPA_JSON.mmap` resource. However, as per the BPMN diagram, the message mapping itself is not invoked in this specific Iflow's execution flow. Therefore, these UDFs are not actively used in the current Iflow configuration.

*   **`script1.js` (JavaScript Script)**
    *   **Purpose:** This JavaScript script is designed to parse an incoming JSON message body, enrich its `context` section by adding hardcoded arrays for `equipments`, `trainings`, and `accessRequests` with predefined values, and then convert the modified JSON back into a string to set as the message body.
    *   **Note:** This script is attached as a resource but is **not referenced or used in any step** within the defined integration process (BPMN diagram) of this Iflow.

#### 6.2 Message Mappings

*   **`Map_SFSF_JSON_to_SPA_JSON.mmap` (Message Mapping)**
    *   **Source Structure:** `swagger_employee_SFSF.json` (specifically `/employee` definition from the `swagger_employee_SFSF.json` schema).
    *   **Target Structure:** `swagger_approval_workflow.json` (specifically `/employee` definition from the `swagger_approval_workflow.json` schema).
    *   **Mapping Logic Highlights:**
        *   `definitionId`: Mapped to a constant value `eu10.btp-demo.advnewhireonboardingexperiencewenterpriseautomation.newEmployeeEquipmentAndTrainingApprovalProcess`.
        *   `context`: Many fields under `context` (e.g., `employeeName`, `jobTitle`, `divisionText`, `employeeId`, `employeeFirstName`, `employeeLastName`, `hireDate`, `employeeGender`, `employeeDateOfBirth`, `managerHrName`, `companyName`, `departmentText`, `businessUnitText`, `country`, `location`) are directly mapped from the source `context` fields.
        *   `managerEmail`: Mapped using the `getProperty` UDF from `UDF.groovy`, retrieving the value of an exchange property named `managerEmailID`.
        *   `employeeEmail`: Mapped from the source `context/email` field.
        *   **Hardcoded Arrays:** The target arrays `trainings`, `equipments`, and `accessRequests` are populated with constant, hardcoded values for their respective sub-elements.
            *   `trainings`: Includes a single entry with `TrainingID` "TR9829", `TrainingName` "User Experience Basics", and `IsRequired` "true".
            *   `equipments`: Includes a single entry with various equipment details and `IsRequired` "true".
            *   `accessRequests`: Includes a single entry with `AccessName` "SAP Analytics Cloud Sales Tenant" and `IsRequired` "true".
    *   **Note:** This message mapping is attached as a resource but is **not referenced or used in any step** within the defined integration process (BPMN diagram) of this Iflow. A dedicated "Message Mapping" step would be required to invoke it.

#### 6.3 JSON Schemas

*   **`swagger_employee_SFSF.json`**
    *   **Purpose:** Defines the JSON structure of the incoming employee data from the SF (SuccessFactors) system. This schema is used as the source structure in the `Map_SFSF_JSON_to_SPA_JSON.mmap`.

*   **`swagger_approval_workflow.json`**
    *   **Purpose:** Defines the target JSON structure for an approval workflow, likely intended for an SAP Process Automation (SPA) or similar workflow service. This schema is used as the target structure in the `Map_SFSF_JSON_to_SPA_JSON.mmap`.