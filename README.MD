# Student Management System - AWS Serverless Application

A serverless **Student Management System** built with **HTML/CSS/JavaScript** on the frontend and **AWS Lambda (Python)** on the backend. The application is hosted on **AWS S3** as a static website and uses **DynamoDB** for data persistence, with **API Gateway** exposing RESTful endpoints.

---

## Features

- **Add Students** - Submit student records (ID, Name, Class, Age) through a web form
- **View Students** - Retrieve and display all student records in a dynamic table
- **Serverless Architecture** - No servers to manage; fully powered by AWS Lambda
- **Static Website Hosting** - Frontend served via AWS S3
- **RESTful API** - API Gateway connects the frontend to Lambda functions
- **Scalable Storage** - DynamoDB handles structured student data with auto-scaling

---

## Tech Stack

| Layer        | Technology                     |
|--------------|--------------------------------|
| **Frontend** | HTML, CSS, JavaScript          |
| **Backend**  | Python 3.x (AWS Lambda)        |
| **Database** | AWS DynamoDB                   |
| **API**      | AWS API Gateway (REST)         |
| **Hosting**  | AWS S3 (Static Website)        |

---

## Project Structure

```
AWS-Student-Management-System/
├── index.html                  # Homepage with navigation
├── add_student.html            # Form to add a new student
├── fetch_all_students.html     # Table view of all students
├── scripts.js                  # Frontend API calls (fetch/POST)
├── README.MD                   # Project documentation
│
└── Python/
    ├── insertStudentData.py    # Lambda function - insert student into DynamoDB
    ├── getStudents.py          # Lambda function - retrieve all students from DynamoDB
    ├── BucketPolicy.txt        # S3 Bucket Policy (public read access)
    └── s3-CORS Policy.txt      # S3 CORS configuration
```

---

## Architecture Diagram

```
┌────────────┐       HTTPS        ┌───────────────┐        Invoke        ┌────────────────┐
│            │  ───────────────►  │               │  ──────────────────► │                │
│  Browser   │                    │  API Gateway  │                      │  AWS Lambda    │
│  (S3 Site) │  ◄───────────────  │  (REST API)   │  ◄──────────────────  │  (Python 3.x)  │
│            │       JSON         │               │       Response       │                │
└────────────┘                    └───────────────┘                      └───────┬────────┘
                                                                                │
                                                                         Read / Write
                                                                                │
                                                                        ┌───────▼────────┐
                                                                        │                │
                                                                        │   DynamoDB     │
                                                                        │  (studentData) │
                                                                        │                │
                                                                        └────────────────┘
```

---

## Setup Instructions

### Step 1: Prerequisites

- **Python** >= 3.7
- **AWS Account** with access to S3, DynamoDB, Lambda, IAM, and API Gateway
- **AWS CLI** installed and configured:
  ```bash
  aws configure
  ```
- **Boto3** (for local testing):
  ```bash
  pip install boto3
  ```

---

### Step 2: Create DynamoDB Table

1. Open the **AWS Console** → **DynamoDB** → **Create table**.
2. Configure the table:
   - **Table name:** `studentData`
   - **Partition key:** `studentid` (String)
3. Leave all other settings as default and click **Create table**.

---

### Step 3: Create IAM Role for Lambda

1. Go to **IAM** → **Roles** → **Create Role**.
2. Select **AWS Service** → **Lambda** as the trusted entity.
3. Attach the managed policy: **AmazonDynamoDBFullAccess**.
4. Name the role: `LambdaDynamoDBRole`.
5. Click **Create role**.

---

### Step 4: Create Lambda Functions

You need **two** Lambda functions:

#### Function 1: `insertStudentData`

- **Purpose:** Inserts a student record into DynamoDB.
- **Runtime:** Python 3.x
- **Role:** `LambdaDynamoDBRole`
- **Code:**

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('studentData')

def lambda_handler(event, context):
    student_id = event['studentid']
    name = event['name']
    student_class = event['class']
    age = event['age']

    response = table.put_item(
        Item={
            'studentid': student_id,
            'name': name,
            'class': student_class,
            'age': age
        }
    )

    return {
        'statusCode': 200,
        'body': json.dumps('Student data saved successfully!')
    }
```

#### Function 2: `getStudentData`

- **Purpose:** Retrieves all student records from DynamoDB.
- **Runtime:** Python 3.x
- **Role:** `LambdaDynamoDBRole`
- **Code:**

```python
import json
import boto3

def lambda_handler(event, context):
    dynamodb = boto3.resource('dynamodb', region_name='eu-west-2')
    table = dynamodb.Table('studentData')

    response = table.scan()
    data = response['Items']

    while 'LastEvaluatedKey' in response:
        response = table.scan(ExclusiveStartKey=response['LastEvaluatedKey'])
        data.extend(response['Items'])

    return data
```

**Steps to deploy each function:**

1. Go to **AWS Lambda** → **Create function**.
2. Choose **Author from scratch**.
3. Set the **Runtime** to Python 3.x.
4. Under **Permissions**, select the existing role `LambdaDynamoDBRole`.
5. Paste the corresponding code and click **Deploy**.
6. Test with a sample event to verify.

---

### Step 5: Set Up API Gateway

1. Go to **API Gateway** → **Create API**.
2. Choose **REST API** → **Build**.
3. Set **API name** to `student`.
4. Create resources and methods:
   - **POST** method → Integrate with `insertStudentData` Lambda
   - **GET** method → Integrate with `getStudentData` Lambda
5. **Enable CORS** for both GET and POST methods.
6. **Deploy the API:**
   - Create a new stage named `prod`.
   - Deploy to the root path (`/`).
7. Copy the **Invoke URL** from the stage editor.
8. Update `scripts.js` with the Invoke URL:
   ```javascript
   var API_ENDPOINT = "https://your-api-id.execute-api.region.amazonaws.com/prod";
   ```

---

### Step 6: Create S3 Bucket (Static Website Hosting)

1. Go to **S3** → **Create bucket**.
2. Upload the following files:
   - `index.html`
   - `add_student.html`
   - `fetch_all_students.html`
   - `scripts.js`
3. Enable **Static Website Hosting:**
   - Go to the bucket **Properties** tab.
   - Scroll to **Static website hosting** → **Edit** → **Enable**.
   - Set **Index document** to `index.html`.
4. **Add Bucket Policy** (for public read access):
   ```json
   {
       "Version": "2012-10-17",
       "Id": "Policy1728925754371",
       "Statement": [
           {
               "Sid": "Stmt1728925752745",
               "Effect": "Allow",
               "Principal": "*",
               "Action": "s3:GetObject",
               "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
           }
       ]
   }
   ```
   > **Note:** Replace `YOUR-BUCKET-NAME` with your actual S3 bucket name.

5. **Add CORS Configuration** (under the **Permissions** tab):
   ```json
   [
       {
           "AllowedHeaders": ["*"],
           "AllowedMethods": ["GET", "POST", "PUT", "DELETE", "HEAD"],
           "AllowedOrigins": ["*"],
           "ExposeHeaders": ["ETag"],
           "MaxAgeSeconds": 3000
       }
   ]
   ```

---

### Step 7: Local Testing (Optional)

Before deploying to Lambda, you can test the Python scripts locally:

```bash
python Python/insertStudentData.py
python Python/getStudents.py
```

> **Note:** Ensure your AWS CLI credentials are configured and you have network access to DynamoDB.

---

### Step 8: Access the Application

1. Open the **S3 static website URL** in your browser (found in the S3 bucket properties).
2. Click **Add Student** to submit new student records.
3. Click **View All Students** to retrieve and display all records from DynamoDB.

---

## Screenshots

| Page | Description |
|------|-------------|
| `index.html` | Homepage with navigation buttons to Add or View students |
| `add_student.html` | Form with fields for Student ID, Name, Class, and Age |
| `fetch_all_students.html` | Table displaying all student records loaded from DynamoDB |

---

## Future Enhancements

- Add **user authentication** (AWS Cognito) for secure access
- Implement **input validation** on both frontend and backend
- Add **update and delete** operations for student records
- Implement **search and filter** functionality
- Add **pagination** for large datasets
- Set up **CI/CD pipeline** for automated deployments

---

## Author

**Khushal Ravindra Bhavsar**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/khushal-bhavsar-/)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Khushal41)

Full Stack MERN Developer | Python & Cloud Enthusiast

---

## License

This project is open-source and available under the [MIT License](LICENSE).
