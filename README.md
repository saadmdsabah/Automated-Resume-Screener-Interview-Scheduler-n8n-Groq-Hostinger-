# Automated Resume Screener & Interview Scheduler (n8n + Groq + Hostinger)

This is a project I built in n8n to automate the first-pass screening of resumes and schedule interviews for qualified candidates. It pulls resumes from a Google Drive folder, uses Groq's Llama models to analyze them, asks for my manual approval via email, and then either schedules an interview in Google Calendar or sends a rejection email.

The entire workflow is hosted on an n8n instance running on Hostinger.

## Workflow Diagram

<img width="1642" height="530" alt="Screenshot 2025-11-16 205148" src="https://github.com/user-attachments/assets/61269df7-4e98-42c8-b01b-28755a874dec" />

## 🚀 Core Features

* **Automated Resume Parsing:** Monitors a Google Drive folder for new resumes.
* **AI-Powered Analysis:** Uses Groq (Llama models) to extract key information (name, email, experience) and provide a hiring recommendation.
* **Human-in-the-Loop:** Pauses the workflow and sends an email to the recruiter (me) to approve or reject the candidate.
* **Automatic Calendar Scheduling:** If approved, it finds the next available 1-hour slot in the *next* working week (Mon-Fri, 8 am-6 pm) and books the interview.
* **Automated Rejections:** If rejected, it politely informs the candidate via email.

## 🛠️ Tech Stack

* **Orchestration:** [n8n.io](https://n8n.io/) (self-hosted on Hostinger)
* **AI Model:** [Groq](https://groq.com/) (using the free-tier Llama models)
* **Storage:** Google Drive (for resume files)
* **Communication:** Gmail (for approval and candidate emails)
* **Scheduling:** Google Calendar

## ⚙️ Setup & Prerequisites

Before running this, you'll need to set up a few things:

1.  **Google Credentials:** You need OAuth credentials for Google Drive, Gmail, and Google Calendar. Make sure they have the right scopes:
    * **Google Drive:** `drive.readonly` (to search and download files)
    * **Gmail:** `gmail.send` (to send emails) and `gmail.modify` (to read replies for the "Wait" node)
    * **Google Calendar:** `calendar.events` (to read free/busy slots and create new events)
2.  **Groq API Key:** You'll need a free account with Groq to get an API key for the `Groq Chat Model` nodes.
3.  **Google Drive Folder:** Create a specific folder in your Google Drive where you'll drop the resumes. You'll need this folder's ID for the `Search files and folders` node.

---

## 🔬 Workflow Breakdown

Here's a step-by-step look at how the nodes are configured.

### 1. Trigger: `When clicking 'Execute workflow'`

This is a manual trigger. I just click "Execute" when I've added a new batch of resumes to the drive.

### 2. `Search files and folders` (Google Drive)

This node points to my specific "Resumes" folder in Google Drive and just lists all the files inside it.

### 3. `Loop Over Items`

This takes the list of files from the previous node and processes them one by one.

### 4. `Download file` & `Extract from PDF`

Inside the loop, it first downloads the file's binary data from Google Drive and then uses the `Extract from PDF` node to convert it all into raw text.

### 5. `AI Agent` (The HR Analyst)

This is the first AI step. It takes the raw text from the PDF and uses Groq to analyze it.

* **System Message:** This sets the context for the AI.
    ```
    # Role
    You are a Human Resources employee working for a software development company. Your role is to analyse Curriculum Vitae's of applicants for a senior development position where the user must possess skills at a senior level in C# programming and have appropriate experience developing software using associated technologies with C# (e.g. MAUI or Blazor).

    # Instructions
    Follow these steps in analysing a Curriculum Vitae and outputting relevant data in your response
    1. Read through the provided information 
    2. Get the applicants full name and email address from the Curriculum Vitae
    3. Get the total number of years of experience the applicant has working with C# and associated technologies
    4. Get notable features about the applicants qualifications
    5. Get notable features about the applicants work experience
    6. Conclude as to whether you would recommend the applicant for the position of senior C# developer

    # Rules
    1. C# is the most important skill for the applicant to have. If the applicant does not have C# experience don't recommend the applicant for the position.
    ```
* **User Message:** This is the actual prompt, feeding in the text from the PDF.
    ```
    Please analyse the following Curriculum Vitae content and return relevant details about the applicant and your recommendation about the applicants suitability to fill the position as senior C# software developer:  {{ $json.text }}
    ```
* **Output Parser:** I'm using the `Structured Output Parser` to force the AI's response into a clean JSON format.
    ```json
    {
        "fullName": "Bob Jones",
        "email": "bobjones@gmail.com",
        "cSharpYearsExperience": 5,
        "notableQualifications": "The applicant has an honours degree in Computer Science",
        "notableWorkExperience": "The applicant has worked on several projects using Blazor and .NET MAUI. The applicant also has significant experience working with AI",
        "Recommended":true
    }
    ```

### 6. `Send a message` (The Approval Email)

This node sends an email *to me* (the recruiter) with the AI's summary. This node is set to **Wait** for a reply.

* **Email Body:**
    ```html
    Please approve or disapprove this applicant?

    Applicant Details:
    Name: {{ $json.output.fullName }}
    Email: {{ $json.output.email }}

    Years of Experience with C#: {{ $json.output.cSharpYearsExperience }}

    Notable Qualifications:
    {{ $json.output.notableQualifications }}

    Notable Work Experience
    {{ $json.output.notableWorkExperience }}

    My Recommendation
    {{ ($json.output.Recommended?"I recommend this applicant":"I do not recommend this applicant") }}
    ```

### 7. `IF` Node

This node checks the response from the email I sent. It simply checks if my reply email's body contains the word "approve".

### 8. The "False" Path: `Send Rejection Mail`

If I reply with "rejected" (or anything that isn't "approve"), this path runs. It's a simple Gmail node that sends a polite rejection email to the applicant's email (`{{ $('AI Agent').item.json.output.email }}`).

### 9. The "True" Path: `AI Agent1` (The Scheduler)

If I reply with "approve", this is where the magic happens. This second AI Agent is responsible for scheduling.

* **System Message:** This agent is purely for scheduling and uses tools.
    ```
    You are an assistant that schedules interviews in Google Calendar.

    Instructions

    Fetch Events
    Call the GetCalendarEvents tool to retrieve all existing events between Monday 00:00 and Friday 23:59 of the next calendar week after {{ $now }}.
    
    If today is within this week, you must skip it. Only consider next week.

    Find Availability
    Analyze the retrieved events. Identify the earliest available 1-hour free slot within working hours (08:00–18:00 local time).
    
    Do not select a time that overlaps with an existing event.
    Do not schedule outside 08:00–18:00.
    Only use dates from next week’s Monday–Friday window.

    Schedule Event
    Once a free slot is found, call the CreateCalendarEvent tool with:
    
    summary: "Interview with {{ $('AI Agent').item.json.output.fullName }}"
    start.dateTime: the beginning of the identified 1-hour slot (ISO 8601 format).
    end.dateTime: the start time plus exactly 60 minutes.
    attendees:
    {"email": "intervieweeemailaddress@gmail.com"} 
    {"email": "intervieweremailaddress@freecodecamp.com"} 

    Return Confirmation
    Return only the scheduled event details as valid JSON...
    
    Rules
    Today’s date/time is {{ $now }}.
    ...
    ```
* **Tools:** This agent has access to two tools:
    1.  **`getCalendarEvents`:** Fetches all my existing events for the *next* week.
    2.  **`createCalendarEvent`:** Creates the actual 1-hour event with the applicant and me as attendees.

### 10. `Workflow Completed`

After the loop finishes processing all the resumes, this final node sends me a simple "Workflow Completed" email so I know it's done.

---

## 🏃 How to Use

1.  Drop one or more PDF resumes into the designated Google Drive folder.
2.  Open the n8n workflow and click "Execute workflow".
3.  Check your email. You'll receive one email per applicant.
4.  Reply to each email with "approve" or "reject".
5.  ...That's it. Approved candidates will be added to your calendar, and rejected ones will get an email. You'll get a final "completed" email when all resumes are processed.
