# freightfox - S3 File Storage Service

A Spring Boot REST API to search, upload, and download files stored in AWS S3.
Each user's files are isolated in their own S3 folder (e.g., `sandy/`).

---
# Video reference for the explanation of the final output: (please watch it for the working using swaggerUI + AWS IAM , S3 bucket):
https://drive.google.com/file/d/1h6yoDs3N2dpVom5nFkPvzoOtZW2Pfmjg/view?usp=drive_link


# Photo of the final output: 
Creation of the s3 bucket with swagger UI
<img width="1914" height="913" alt="image" src="https://github.com/user-attachments/assets/00516501-b04f-48de-97c9-429ac674c07c" />
<img width="1916" height="909" alt="image" src="https://github.com/user-attachments/assets/26c11b19-b887-4bac-a38c-40576c70c891" />

Creation of IAM user before creating s3 bucket to give the user role for s3 bucket to access aws resources - given s3Fullaccess
<img width="1610" height="552" alt="image" src="https://github.com/user-attachments/assets/c8600596-21c5-45f1-a1ce-3ea6905b1d09" />

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Project Structure](#2-project-structure)
3. [AWS S3 Setup](#3-aws-s3-setup)
4. [Configure the Application](#4-configure-the-application)
5. [Build & Run](#5-build--run)
6. [API Endpoints](#6-api-endpoints)
7. [Testing with Swagger UI](#7-testing-with-swagger-ui)
8. [Testing with Postman](#8-testing-with-postman)
9. [Running JUnit Tests](#9-running-junit-tests)
10. [How It Works (Explained Simply)](#10-how-it-works-explained-simply)
11. [Common Errors & Fixes](#11-common-errors--fixes)

---

## 1. Prerequisites

Install the following before starting:

| Tool | Version | Download |
|------|---------|----------|
| Java JDK | 17+ | https://adoptium.net |
| Maven | 3.8+ | https://maven.apache.org/download.cgi |
| AWS Account | — | https://aws.amazon.com |
| Postman (optional) | Latest | https://www.postman.com/downloads |

**Verify installations:**
```bash
java -version       # Should print: openjdk 17...
mvn -version        # Should print: Apache Maven 3.x.x
```

---

## 2. Project Structure

```
s3-file-storage/
├── pom.xml                          ← Maven dependencies (like package.json for Java)
└── src/
    ├── main/
    │   ├── java/com/storage/
    │   │   ├── FileStorageApplication.java   ← App entry point (main method)
    │   │   ├── config/
    │   │   │   └── S3Config.java             ← Creates the S3 connection
    │   │   ├── controller/
    │   │   │   └── FileStorageController.java← REST API endpoints (URLs)
    │   │   ├── service/
    │   │   │   └── S3FileService.java        ← Business logic (talks to S3)
    │   │   ├── dto/
    │   │   │   ├── FileMetadataDto.java      ← Data shape for a single file
    │   │   │   └── SearchResponseDto.java    ← Data shape for search results
    │   │   └── exception/
    │   │       ├── FileNotFoundException.java
    │   │       └── GlobalExceptionHandler.java ← Handles errors nicely
    │   └── resources/
    │       └── application.properties        ← AWS config goes here
    └── test/
        └── java/com/storage/
            └── S3FileServiceTest.java        ← JUnit tests
```

**Key concept:** Think of it in 3 layers:
- **Controller** → receives HTTP request → calls Service
- **Service** → business logic → calls AWS S3
- **S3** → stores/retrieves the actual files

---

## 3. AWS S3 Setup

### Step 3.1 – Create an S3 Bucket

1. Go to [AWS Console → S3](https://s3.console.aws.amazon.com/s3)
2. Click **Create bucket**
3. Enter a unique name, e.g., `my-user-storage-bucket-2024`
4. Choose a region (e.g., `us-east-1`)
5. **Block all public access** → leave it ON (keep files private)
6. Click **Create bucket**

### Step 3.2 – Create an IAM User with S3 Access

> IAM = Identity & Access Management (controls who can do what on AWS)

1. Go to [AWS Console → IAM → Users](https://console.aws.amazon.com/iam/home#/users)
2. Click **Create user**
3. Enter username: `s3-file-storage-user`
4. Click **Next**
5. Select **Attach policies directly**
6. Search for and select: `AmazonS3FullAccess`
   - _(For production, create a custom policy with minimal permissions)_
7. Click **Next → Create user**

### Step 3.3 – Generate Access Keys

1. Click on the user you just created
2. Go to **Security credentials** tab
3. Scroll to **Access keys** → click **Create access key**
4. Select **Application running outside AWS**
5. Click **Next → Create access key**
6. **IMPORTANT:** Copy and save both:
   - `Access key ID` (looks like: `AKIAIOSFODNN7EXAMPLE`)
   - `Secret access key` (looks like: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`)
   - ⚠️ You cannot see the secret key again after closing this page!

### Step 3.4 – (Optional) Add Test Files to S3

To test search right away, add some files manually:

1. In your S3 bucket, click **Create folder**
2. Name it `sandy` (this is the user folder)
3. Open the `sandy` folder → click **Upload**
4. Upload any file (e.g., `logistics-report.pdf`, `invoice.txt`)

---

## 4. Configure the Application

Open this file: `src/main/resources/application.properties`

Replace the placeholder values:

```properties
# Your S3 bucket name (from Step 3.1)
aws.s3.bucket-name=my-user-storage-bucket-2024

# Region you chose when creating the bucket
aws.s3.region=us-east-1

# From Step 3.3
aws.access-key=AKIAIOSFODNN7EXAMPLE
aws.secret-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

> **Security tip:** Never commit your actual keys to Git.
> Add `application.properties` to `.gitignore`, or use environment variables instead:
> ```bash
> export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
> export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
> ```
> If you use env vars, leave `aws.access-key` and `aws.secret-key` blank in the properties file — the app will automatically pick them up.

---

## 5. Build & Run

### Step 5.1 – Download dependencies and build
```bash
cd s3-file-storage
mvn clean install -DskipTests
```
This downloads all libraries listed in `pom.xml` (first run may take 1-2 minutes).

### Step 5.2 – Start the application
```bash
mvn spring-boot:run
```

You should see output like:
```
Started FileStorageApplication in 2.345 seconds
Tomcat started on port(s): 8080
```

The app is now running at: **http://localhost:8080**

---

## 6. API Endpoints

### 🔍 Search Files
```
GET /api/files/{userName}/search?term={searchTerm}
```
Returns all files in the user's folder whose name contains the search term.

**Example:**
```
GET http://localhost:8080/api/files/sandy/search?term=logistics
```

**Response:**
```json
{
  "userName": "sandy",
  "searchTerm": "logistics",
  "totalResults": 2,
  "files": [
    {
      "fileName": "logistics-report.pdf",
      "s3Key": "sandy/logistics-report.pdf",
      "sizeBytes": 204800,
      "lastModified": "2024-01-15T10:30:00Z",
      "downloadUrl": "/api/files/sandy/download/logistics-report.pdf"
    }
  ]
}
```

---

### ⬇️ Download a File
```
GET /api/files/{userName}/download/{fileName}
```
Downloads the file directly as a binary stream.

**Example:**
```
GET http://localhost:8080/api/files/sandy/download/logistics-report.pdf
```
The file will be downloaded to your browser or Postman.

---

### ⬆️ Upload a File
```
POST /api/files/{userName}/upload
Content-Type: multipart/form-data
Body: file=<your file>
```

**Example:**
```
POST http://localhost:8080/api/files/sandy/upload
```

**Response:**
```json
{
  "fileName": "new-document.pdf",
  "s3Key": "sandy/new-document.pdf",
  "sizeBytes": 51200,
  "downloadUrl": "/api/files/sandy/download/new-document.pdf"
}
```

---

## 7. Testing with Swagger UI

Swagger gives you a visual interface to test all APIs — no Postman needed!

1. Start the app (`mvn spring-boot:run`)
2. Open browser: **http://localhost:8080/swagger-ui.html**
3. You'll see all 3 endpoints listed
4. Click any endpoint → **Try it out** → fill in parameters → **Execute**

---

## 8. Testing with Postman

### Import the API

**Search Files:**
- Method: `GET`
- URL: `http://localhost:8080/api/files/sandy/search?term=logistics`
- Click **Send**

**Download a File:**
- Method: `GET`
- URL: `http://localhost:8080/api/files/sandy/download/logistics-report.pdf`
- Click **Send**
- Click **Save Response** to save the file

**Upload a File:**
- Method: `POST`
- URL: `http://localhost:8080/api/files/sandy/upload`
- Go to **Body** tab → select **form-data**
- Add key: `file`, change type to **File**, choose a file from your computer
- Click **Send**

---

## 9. Running JUnit Tests

```bash
mvn test
```

Tests are located in `src/test/java/com/storage/S3FileServiceTest.java`.

The tests use **Mockito** to simulate S3 — no real AWS connection needed to run tests.

**What the tests cover:**
- Search returns only matching files
- Search returns empty when nothing matches
- Search is case-insensitive
- Search paginates correctly across multiple S3 pages
- Download throws a proper error for missing files
- Download URL format is correct

---

## 10. How It Works (Explained Simply)

```
You (Postman/Browser)
       │
       ▼
FileStorageController  ← Receives HTTP requests, validates input
       │
       ▼
S3FileService          ← Business logic: builds S3 keys, filters results
       │
       ▼
AWS S3 SDK             ← Makes actual API calls to Amazon S3
       │
       ▼
S3 Bucket              ← Your files stored as:
                            sandy/
                              logistics-report.pdf
                              invoice.txt
                            john/
                              contract.docx
```

**Key Design Decisions:**
- **User isolation:** Files are stored under `{userName}/` prefix — searching for `sandy` only looks inside `sandy/` folder
- **Pagination:** S3 returns max 1000 files per request. The service loops until all pages are fetched
- **Extensibility:** Adding new features (e.g., move file, delete file) = add a method to `S3FileService` + a new endpoint in `FileStorageController`

---

## 11. Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `The AWS Access Key Id does not exist` | Wrong access key | Double-check `aws.access-key` in application.properties |
| `Access Denied` | IAM user lacks S3 permission | Attach `AmazonS3FullAccess` policy to your IAM user |
| `The specified bucket does not exist` | Wrong bucket name | Check `aws.s3.bucket-name` matches your actual bucket |
| `Port 8080 already in use` | Another app is on port 8080 | Add `server.port=8081` to application.properties |
| `File 'x' not found for user 'sandy'` | File doesn't exist in S3 | Upload the file first, or check the exact filename |
| `Connection refused` | App not started | Run `mvn spring-boot:run` first |
