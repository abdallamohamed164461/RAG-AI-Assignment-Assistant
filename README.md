# RAG AI Assignment Assistant

A RAG-powered Telegram bot that helps students understand assignments and get instant AI feedback on submissions, while logging detailed results for instructors.

## Table of contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Tech stack](#tech-stack)
- [Setup and usage](#setup-and-usage)
  - [Prerequisites](#prerequisites)
  - [Import and configure the workflow](#import-and-configure-the-workflow)
  - [Prepare the knowledge base](#prepare-the-knowledge-base)
  - [Prepare the instructor log](#prepare-the-instructor-log)
  - [Upload assignment files](#upload-assignment-files)
  - [Student interaction](#student-interaction)
- [Project structure](#project-structure)
- [License](#license)

## Overview

This project uses retrieval-augmented generation (RAG) to answer assignment questions from instructor-provided materials rather than relying on the model's general knowledge.

The ingestion pipeline watches a Google Drive folder for new files, downloads them, loads their contents, creates Google Gemini embeddings, and inserts the resulting documents into a Pinecone vector index. The exported workflow uses n8n's Default Data Loader; configure a text splitter in the document-loading path if your n8n setup does not split the source material into suitable chunks automatically.

The student-facing pipeline receives Telegram messages and sends them to an AI Agent connected to Google Gemini, session memory, a Pinecone retrieval tool, and a Google Sheets logging tool. The prompt instructs the agent to explain assignment requirements without revealing the model solution. For submissions, it compares the work against retrieved assignment material, records a score and technical notes in the instructor sheet, and returns a short motivational response without exposing the score.

## Architecture

![Workflow Architecture](./assets/workflow-diagram.png)

The diagram is a placeholder. Add a screenshot of the n8n canvas at `assets/workflow-diagram.png`.

## Features

- Instant assignment Q&A grounded in the knowledge base, without revealing the model solution.
- Automatic grading of student submissions against the retrieved model solution.
- Motivational, student-facing feedback that does not disclose the score.
- Detailed instructor logging in Google Sheets, including the submission, rating, and technical feedback.
- HTML-formatted Telegram replies.

## Tech stack

- **Workflow automation:** n8n
- **Chat model and embeddings:** Google Gemini
- **Vector database:** Pinecone
- **Assignment source files:** Google Drive
- **Submission and feedback log:** Google Sheets
- **Student interface:** Telegram Bot API

## Setup and usage

### Prerequisites

- An n8n instance with the Google Drive, Google Sheets, Telegram, Gemini, Pinecone, and LangChain nodes available.
- A Telegram bot token.
- A Google Gemini API key.
- A Pinecone API key and a Pinecone project.
- Google OAuth credentials with access to the Drive folder and Sheet used by the workflow.
- An assignment-support Telegram bot/chat and a Google Sheet for instructor logs.

### Import and configure the workflow

1. Import [`workflow.sanitized.json`](./workflow.sanitized.json) into n8n. For a self-hosted instance with the n8n CLI available, you can also run:

   ```bash
   n8n import:workflow --input=workflow.sanitized.json
   ```

2. The public workflow export contains no credential references. Create the required credentials in n8n and select them in the matching nodes:
   - **Google Drive OAuth2:** `Google Drive Trigger` and `Download file`.
   - **Google Gemini API:** `Google Gemini Chat Model` and both `Embeddings Google Gemini` nodes.
   - **Pinecone API:** both `Pinecone Vector Store` nodes.
   - **Telegram API:** `Telegram Trigger` and `Send a text message`.
   - **Google Sheets OAuth2:** `feedbacks to doctor`.
3. Select your own Drive folder, Pinecone index, spreadsheet, and sheet in the relevant nodes. The sanitized JSON remains valid JSON; JSON does not support comments, so the resource placeholder names and this section explain their purpose:

   | Placeholder | Use |
   | --- | --- |
   | `YOUR_GOOGLE_DRIVE_FOLDER_ID` | Google Drive folder watched for new assignment files. |
   | `YOUR_GOOGLE_SHEETS_SPREADSHEET_ID` | Spreadsheet receiving submission logs. |
   | `YOUR_GOOGLE_SHEETS_SHEET_ID` | Tab/sheet identifier within that spreadsheet. |

   Credential references, webhook IDs, and account-specific workflow metadata are omitted from the public export. n8n will use your credentials and create local workflow/webhook metadata when you configure and save the imported workflow.
4. In the Telegram Trigger node, configure and activate the webhook for your bot. Confirm that the trigger and send-message nodes use the same Telegram credential.
5. Test the workflow with a non-production assignment and test student account before sharing the bot.

The configured Pinecone index name is **`rag-ai-agent`**. Create an index with embedding dimensions compatible with the Google Gemini embedding model selected in n8n, then select it in both Pinecone nodes.

### Prepare the knowledge base

1. Create a Google Drive folder named **`RAG`** (or choose another folder and select it in `Google Drive Trigger`).
2. Grant the Google OAuth account used by n8n access to the folder.
3. Add assignment materials to the folder. The trigger polls for newly created files every minute.

### Prepare the instructor log

Create a Google Sheet and provide these columns in the first row:

| Name | Code | Assigment Name | solution | Rate | Feedback |
| --- | --- | --- | --- | --- | --- |

The spelling `Assigment Name` is retained to match the workflow prompt and export. The exported Sheets node currently maps its rating field to **`Rate ` with a trailing space**. To use the requested `Rate` column exactly, update that field mapping in the `feedbacks to doctor` node after import and refresh the node's column schema.

### Upload assignment files

- Use text-searchable PDF, DOCX, or plain-text files supported by your n8n document loader.
- Keep each assignment in a clearly named file; avoid combining unrelated assignments in one document.
- Clearly label the assignment title, question, requirements, and model solution. Keep the model solution in instructor-controlled Drive storage, not in the student-facing Telegram chat.
- Use consistent headings, for example:

  ```text
  Assignment: Assignment 1

  Question
  [Assignment prompt and requirements]

  Model solution
  [Instructor reference solution]
  ```

- After uploading a file, check the n8n execution history to confirm that it was loaded and inserted into Pinecone. Remove outdated or duplicate vectors when replacing source material, if needed.

### Student interaction

Students can ask a question about an assignment or submit their work through the Telegram bot. If the student has not shared their name and code in the current conversation, the agent should request them before processing a submission.

Example:

```text
Student: What is Assignment 1 asking me to do?
Bot: Assignment 1 asks you to ... [a requirements-only explanation grounded in the course material]

Student: My name is Alex and my code is A102. Here is my solution: ...
Bot: ✅ Assignment: Assignment 1
     📝 تم استلام حلك بنجاح!
     Great effort—review the assignment requirements once more before your next submission.
```

The bot is instructed not to provide the model solution in response to student questions. Submission ratings and detailed technical feedback are intended for the instructor log, not the student-facing reply.

## Project structure

```text
.
├── .gitignore
├── README.md
├── workflow.sanitized.json
└── assets/
    └── workflow-diagram.png   # Add the n8n canvas screenshot
```

The original unsanitized workflow export and local `Screenshot*` images are excluded from version control. The existing local screenshot contains instance/workflow details; add a reviewed, appropriately cropped diagram at `assets/workflow-diagram.png` instead. Do not publish the original export or that local screenshot.

## License

To be determined. Add a license and its full text before redistributing this project.
