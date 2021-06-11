## Eigen API Quickstart Guide

## Introduction

This is a quick introduction to using the Eigen API to interact with the Eigen Platform. Used in conjunction with the documentation served by the webserver itself (accessible at `https://hostname/api/v1/docs/`), when authenticated, it should provide enough information to begin using the Eigen platform as part of your own toolchain.

All URLs in this document are given relative to the root path, names as `{BASE_URL}` and so are assumed to include the `https://hostname/api/v1/` as a prepend - e.g. the url `{BASE_URL}/docs/` is actually `https://hostname/api/v1/docs/`.

Similarly, unless otherwise noted, all requests made have a content type of `application/json`.

## Authentication

Before being able to perform any actions with the Eigen server, you first need to authenticate. To accomplish this, execute the following request:

```
type: POST
url: {BASE_URL}/api-token-auth/
data = {
    "username": "<your-username>",
    "password": "<your-password>",
}
```

This request will return a response with a 200 status code if authentication is successful, and a 400 if invalid credentials have been passed.

200 Response:

```
{
    'id': <assigned-database-user-id>,
    'token': "<authentication-token>",
    'username': "<registered-username>",
    'email': "<registered-email>",
    'permissions': [<permissions>], # List of permissions assigned to the user
    'projects': [<projects>], # List of projects assigned to the user
    'isSuperuser': <isSuperuser>, # Boolean flage denoting if user is a superuser
    ...
}
```

From Eigen version 3.7 onwards, this end point is redirected to an authentication service. Authentication directly against this new service is the recommended approach, though the redirect will be maintained for backwards compatibility. The request is updated to:

```
type: POST
url: https://hostname/auth/v1/token/
data = {
    "username": "<your-username>",
    "password": "<your-password>",
}
```

A successful response (200) will yield:

```
{
    'pk': 1234567890,
    'refresh_token': "<refresh-token>",
    'token': "<authentication-token>"
}
```

The JWT access token and the refresh token both get set as HttpOnly Cookies as `authtoken` and `refresh_token` respectively on the client.

For further progress only the token key is required from the above response. This contains a string which is your authentication for future requests. You can either place this token as part of an `AUTH_HEADER` prepended by the word `Token` e.g. `Authorization: Token <token>` or as a cookie with name `authtoken`.

Neither the manual setting of the cookie nor the setting of the token header are needed if using the same client that was used to retrieve the token since the token would be set as a cookie on the client as mentioned above.

## API Preview

In this preview we are going to walk through:

1.  Making Queries
    - Listing All Document Types
    - Listing All Questions
2.  Uploading Documents
    - Bulk Uploading
    - Checking Document Upload Status
3.  Extraction Frontend Flow
    - Creating an Analysis
    - Starting Analysis Extraction
    - Exporting Results from Analysis
4.  Extraction Service Flow
    - Starting Extraction
5.  Task Inspection
6.  Answer Inspection

To make perform the requests below, please make sure the [Authentication](#authentication) step is completed successfully and the authentication token/cookie is set.

### Making Queries

#### Listing All Document Types

To query all document types in the database, execute the following request:

```
type: GET
url: {BASE_URL}/document_type/
```

Response:

```
[
    {
        "id": 1,
        "name": "Demo",
        "documents": [1, 2, 3],
        "language": "english",
        "questions": [1, 2, 3],
        "project": 1,
        "created_at": "1585579461.921797"
    },
    ...
]
```

#### Listing All Questions

To query all questions in the database, execute the following request:

```
type: GET
url: {BASE_URL}/question/
```

Response:

```
[
    {
        "id": 1,
        "question_type": 0,
        "document_type": 1,
        "text": "Date",
        "qnum": "1",
        "active_seedling": 1,
        "latest_seedling": 1,
        "version": 1,
        "task_status": "SUCCESS",
        "trained": true,
        "training_documents": [1, 2, 3],
        "logic": null,
        "logic_group": null,
        "force_single": false,
        "order": 0,
        "labelled_document_count": 3,
        "needs_cross_validation": False,
        "no_answer_count": 0
    },
    ...
]
```

### Uploading documents

#### Bulk Uploading

To upload many documents at once, execute the following request:

Please note that:

- This is an **asynchronous** endpoint
- We recommended to upload less than 1GB of data at a time and wait for the documents to complete uploading (see [Checking Document Upload Status](#checking-document-upload-status) for details) before sending the next batch.

```
type: POST
url: {BASE_URL}/document/document_uploader/
data = {
    "document_type_id": <document-type-id>, # Document type id to upload documents to
    "files": [<files>], # List of files to upload
}
content_type = "multipart/form-data"
```

Response:

```
[
    {
        "document": {
            "id": 1,
            "files": {},
            "filename": "Demo txt Document",
            "uploaded_filename": "Demo txt Document.txt",
            "status": 0,
            "has_readable_pdf": false,
            "updated_at": "1585758496.386053",
            "document_type": 1,
            "labelling": false
        },
        "file": "Demo txt Document.txt"
    },
    ...
]
```

The response will list documents with `status: 0`, indicating the upload process has begun but has not completed.

#### Checking Document Upload Status

You can check if the upload process has finished by polling each document using the following request:

```
type: GET
url: {BASE_URL}/document/<document-id>
```

Response:

```
{
    "id": 1,
    "document_type": 1,
    "files": {},
    "family": null,
    "tags": [],
    "metadata": {},
    "filename": "Demo txt Document",
    "status": 2,
    "updated_at": "1585583501.738109",
    "labelling": false
}
```

- The value of `status` has the following associations:

```
0 --> Uploading
1 --> OCR Started
2 --> Complete
3 --> Failure
```

- When `status` has a value of `2` the upload process has finished and that document can be used for training/extraction.

### Extraction Frontend Flow

This flow demonstrates how to create extractions that are visible in the frontend of the Eigen Platform.

#### Creating an Analysis

An analysis is a collection of documents and questions within a document type. An analysis can be created for easier viewing of the extraction results by executing the following request:

```
type: POST
url: {BASE_URL}/tag/
data = {
    "name": "<analysis_name>", # Analysis name to create (must be unique in document type)
    "document_type": <document-type-id>, # Document type id you want to create analysis in
    "documents": [<document-ids>], # List of document ids to include in the analysis
    "questions": [<question-ids>], # List of question ids to include in the analysis
}
```

Response:

```
{
    "id": 1,
    "name": "Demo Analysis",
    "description": null,
    "is_system_tag": false,
    "document_type": 1,
    "documents": [1, 2, 3],
    "num_documents": null,
    "num_questions": null,
    "questions": [1],
    "category": "analysis",
    "analysis": 8
}
```

**Notes**:

- The returned `id` value is used to identify the Analysis/Tag for starting extractions or exporting results.
- The `analysis` created by this request will be available in user interface of the application under the target document type, with the name given in the request.

#### Starting Analysis Extraction

The [previous step](#creating-an-analysis) will create an Analysis object (an empty table on the frontend). To begin extraction on the questions/documents in the analysis, execute the following request:

```
type: POST
url: /document_type/<document-type-id>/analyse/
data = {
    "analysis_id": <analysis-id>, # Existing analysis id to start extraction on
}
```

Response:

```
{
    "task_ids": [
        "a54d485c-f9b0-45ed-bc7f-ac6a2d182508",
        "12fa02ba-6d6f-4481-8da9-cb3845549a4b"
    ],
    "document_tasks_ids": {
        "4": [
            "a54d485c-f9b0-45ed-bc7f-ac6a2d182508"
        ],
        "5": [
            "12fa02ba-6d6f-4481-8da9-cb3845549a4b"
        ]
    }
}
```

**Notes**:

- See [Task Inspection](#task-inspection) subsection for information on how to check the status of the returned task ids.
- When all the tasks are finished you can view the results at: `{BASE_URL}/document_type/<document-type-id>/analysis/<analysis>/results`.

#### Exporting Results from Analysis

An analysis can be used to export completed extraction data in csv or xlsx format:

_Prior to exporting the results, make sure all extraction tasks have reached a `SUCCESS`._

```
type: GET
url: {BASE_URL}/document_type/<document-type-id>/export_results/?analysis=<analysis-id>&export_format=csv
```

Response:

```
Document Type,Demo
Filename,1 Date
"Demo txt Document","March 22, 2011"
"Demo docx Document","July 27, 2004"
```

**Notes**:

- If export_format query parameter is omitted, the endpoint will return an `.xls` file.

### Extraction Service Flow

Extraction can also be performed as a backend service, where no analysis table is displayed in the frontend.

#### Starting Extraction

To start an extraction as a backend service execute the following request:

```
type: POST
url: /document_type/<document-type-id>/extract
data = {
    "document_ids": [<document-ids>], # List of document ids that exist in the specified document type to be included in the extraction
    "question_ids": [<question-ids>], # List of question ids that exist in the specified document type to be included in the extraction
}
```

Response:

```
{
    "task_ids": [
        "a8a8d143-4e3f-40d7-badb-d2a61ec7f668",
        "85928b1e-6dd6-480a-8993-2a5ba6b2ceac"
    ],
    "document_tasks_ids": {
        "1": [
            "a8a8d143-4e3f-40d7-badb-d2a61ec7f668"
        ],
        "2": [
            "85928b1e-6dd6-480a-8993-2a5ba6b2ceac"
        ]
    }
}
```

**Notes**:

- See [Task Inspection](#task-inspection) subsection for information on how to check the status of the returned task ids.
- See [Answer Inspection](#answer-inspection) subsection for information on how to retrieve the answers of a completed extraction.

### Task Inspection

Assuming you have a list of extraction task ids, you can check the status of your extraction tasks by executing the following request:

```
type: GET
url: {BASE_URL}/tasks/?task_id_in=<task-ids> # <task-ids>: List of task ids separated by ","
```

e.g.

```
type: GET
localhost:8000/api/v1/tasks/?task_id_in=7dd8e895-37d5-48a0-beb8-b7f15d864471,14e270fd-91c8-4b75-b74a-c868723ef178
```

Response:

```
[
    {
        "id": 126,
        "task_id": "7dd8e895-37d5-48a0-beb8-b7f15d864471",
        "document_ids": [1],
        "extracting_document": 1,
        "task_type": null,
        "status": "SUCCESS",
        "result": [50],
        "date_done": "1585591289.977831",
        "analysis": 34
    },
    {
        "id": 127,
        "task_id": "14e270fd-91c8-4b75-b74a-c868723ef178",
        "document_ids": [2],
        "extracting_document": 2,
        "task_type": null,
        "status": "SUCCESS",
        "result": [52],
        "date_done": "1585591290.840547",
        "analysis": 34
    }
]
```

### Answer Inspection

You can inspect the answer from the tasks above by executing following request:

_Please wait until all extraction tasks have reached `SUCCESS` status prior to exporting results._

```
type: GET
url: {BASE_URL}/answer/?document_in=<document-ids>&question_in=<question-ids>&archived=<true || false>
# <document-ids> and <question-ids>: List of ids separated by ","
# archived=false specifies you want the most recent extractions for the given document/question ids
```

e.g.

```
type: GET
url: {BASE_URL}/answer/?document_in=1,2,3&question_in=1&archived=false
```

Response:

```
[
    {
        "id": 53,
        "data": [
            {
                "end": 102,
                "text": "March 22, 2011",
                "start": 93,
                "endOffset": 14,
                "startOffset": 6,
                "endPageNumber": 1,
                "endContainerId": "text-1-7",
                "startPageNumber": 1,
                "startContainerId": "text-1-7"
            }
        ],
        "question": 1,
        "document": 1,
        "archived": false,
        "flag": "no_flag",
        "logic_id": null,
        "is_from_training": true,
        "is_family_answer": false,
        "is_reviewed": false
    },
    {
        "id": 54,
        "data": [
            {
                "end": 77,
                "text": "July 27, 2004",
                "start": 69,
                "endOffset": 75,
                "startOffset": 67,
                "endPageNumber": 1,
                "endContainerId": "text-1-0",
                "startPageNumber": 1,
                "startContainerId": "text-1-0"
            }
        ],
        "question": 1,
        "document": 2,
        "archived": false,
        "flag": "no_flag",
        "logic_id": null,
        "is_from_training": true,
        "is_family_answer": false,
        "is_reviewed": false
    }
]
```

**Notes:**

- `data` will contain a list of the answer(s) for a question/document. Each answer will have a `text` attribute which is the extracted answer from the document and `start`/`end` attributes which are the index offsets of the answer in the document.

## Postman Collections

This quickstart guide is supplemented by two `.json` files: `eigen-api-demo.postman_collection.json` and `eigen-api-demo-env.postman_environment.json`. These files contain a Postman collection and environment respectively.

1. Load them into Postman
   - Open Postman
   - Select the "Import" button and select both `.json` files
   - This will create the `eigen-api-demo` collection and the `eigen-api-demo-env` environment in Postman
2. Activate `eigen-api-demo-env` and fill in necessary values
   - Select `eigen-api-demo-env` from the list of available environments to activate it
   - Edit `eigen-api-demo-env`:
     - Enter the api base url for the instance you want to interact with in the form `<host-address>/api/v1` (Note no trailing slash)
     - Enter in the username you wish to authenticate with
     - Enter in the password associated with the entered username
     - Leave `auth_token` blank
3. Send "Auth Token" POST request included in the collection. This will automatically store the token in the response to the `eigen-api-demo-env` environment. All requests included in the collection will automatically source the token from the environment. If the token expires, simply run the "Auth Token" POST request and the token value will update in the environment.

Please see [here](https://learning.postman.com/docs/postman/variables-and-environments/variables/) for more information on Postman environments and variables.

## Final comments

The full documentation of the endpoints discussed in this guide is available in the form of interactive swagger docs at `{BASE URL}/docs/`. This documentation also includes numerous other endpoints not dicussed in this guide, such as:

- Training
- Creating new Document Types
- Querying results