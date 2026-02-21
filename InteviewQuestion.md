# AWS Student Management System - Interview Questions & Answers

These 50 interview questions cover the AWS services, architecture, and concepts used in this Student Management System project. Includes conceptual, technical, and scenario-based questions.

---

## Section 1: AWS DynamoDB (Questions 1-10)

### Q1. What is Amazon DynamoDB and why did you choose it for this project?

**Answer:**
Amazon DynamoDB is a fully managed, serverless NoSQL database service that provides fast and predictable performance with seamless scalability. I chose it for this project because:
- It integrates natively with AWS Lambda without additional drivers.
- It offers single-digit millisecond response times ideal for student record lookups.
- It requires no server provisioning or maintenance.
- The on-demand capacity mode is cost-effective for a student management system with variable traffic.

---

### Q2. What is a Partition Key in DynamoDB? Why did you use `studentid` as the partition key?

**Answer:**
A Partition Key is a primary key attribute that DynamoDB uses to distribute data across partitions for scalability. I used `studentid` because:
- Each student has a unique ID, ensuring no duplicate records.
- It allows direct lookups by student ID with O(1) performance.
- DynamoDB uses the partition key's hash value to determine the physical partition where the item is stored.

---

### Q3. What is the difference between `Scan` and `Query` operations in DynamoDB?

**Answer:**
| Feature | Scan | Query |
|---------|------|-------|
| **Scope** | Reads every item in the entire table | Reads items matching a specific partition key |
| **Performance** | Slower and more expensive (full table read) | Faster and cheaper (targeted read) |
| **Use Case** | Retrieving all records | Fetching records by a known key |
| **Filter** | Applied after reading all items | Applied during the read operation |

In this project, I used `Scan` in `getStudents.py` because the requirement is to retrieve **all** student records. If I needed to fetch a single student by ID, I would use `Query` or `get_item`.

---

### Q4. What happens when a DynamoDB Scan returns more data than the 1 MB limit?

**Answer:**
DynamoDB paginates the results automatically. The response includes a `LastEvaluatedKey` field, which acts as a cursor. You must make subsequent `Scan` calls with `ExclusiveStartKey` set to this value to retrieve the next page. In my `getStudents.py`, I handle this:

```python
while 'LastEvaluatedKey' in response:
    response = table.scan(ExclusiveStartKey=response['LastEvaluatedKey'])
    data.extend(response['Items'])
```

This ensures all records are fetched regardless of the dataset size.

---

### Q5. What is the difference between DynamoDB On-Demand and Provisioned capacity modes?

**Answer:**
- **On-Demand Mode:** You pay per request. DynamoDB auto-scales to handle any traffic. Ideal for unpredictable workloads or new applications.
- **Provisioned Mode:** You specify the number of reads/writes per second (RCU/WCU). Suitable for predictable workloads with consistent traffic.

For a student management system with variable usage (e.g., higher during enrollment periods), **On-Demand** is more cost-effective.

---

### Q6. How does DynamoDB ensure high availability?

**Answer:**
DynamoDB automatically replicates data across **three Availability Zones (AZs)** within an AWS Region. This provides:
- Built-in fault tolerance if one AZ fails.
- Synchronous replication for strong consistency.
- 99.999% availability SLA for Global Tables.
- Automatic failover without manual intervention.

---

### Q7. What is a Global Secondary Index (GSI)? When would you add one to the `studentData` table?

**Answer:**
A GSI is an index with a partition key and optional sort key that differ from the table's primary key. It allows querying on non-key attributes.

I would add a GSI if the application needed to:
- Search students by **name** (GSI with `name` as partition key).
- Filter students by **class** (GSI with `class` as partition key and `name` as sort key).

Example:
```
GSI: ClassIndex
  Partition Key: class
  Sort Key: name
```

---

### Q8. **Scenario:** Your DynamoDB table has 100,000 student records. The `getStudents` Lambda function is timing out. How would you fix this?

**Answer:**
Several approaches:
1. **Increase Lambda timeout** from the default 3 seconds to up to 15 minutes.
2. **Implement pagination** on the API - return results in pages of 50-100 records instead of all at once.
3. **Use `ProjectionExpression`** to fetch only required attributes instead of all columns.
4. **Add a GSI** and use `Query` instead of `Scan` if filtering by specific criteria.
5. **Enable DynamoDB Accelerator (DAX)** for caching frequently accessed data.
6. **Increase Lambda memory** - this also increases CPU allocation, making the function faster.

---

### Q9. **Scenario:** Two users submit the same `studentid` simultaneously. What happens?

**Answer:**
DynamoDB's `put_item` operation will overwrite the existing item with the last write winning (last-writer-wins). To prevent this:
1. **Use Conditional Expressions:**
   ```python
   table.put_item(
       Item={...},
       ConditionExpression='attribute_not_exists(studentid)'
   )
   ```
   This throws a `ConditionalCheckFailedException` if the ID already exists.
2. **Generate unique IDs** server-side using UUID instead of relying on user input.

---

### Q10. What is DynamoDB TTL (Time to Live)? Give a use case for this project.

**Answer:**
TTL automatically deletes items after a specified expiration timestamp (epoch time). DynamoDB removes expired items in the background at no additional cost.

**Use case:** If you want to auto-delete student records after graduation (e.g., 4 years after enrollment), you could add a `ttl` attribute with the expiration epoch time. DynamoDB would automatically clean up old records.

---

## Section 2: AWS Lambda (Questions 11-20)

### Q11. What is AWS Lambda and how does it work in this project?

**Answer:**
AWS Lambda is a serverless compute service that runs code in response to events without provisioning or managing servers. In this project:
- **`insertStudentData` function** - Triggered by a POST request via API Gateway; writes student data to DynamoDB.
- **`getStudentData` function** - Triggered by a GET request via API Gateway; scans DynamoDB and returns all records.

Lambda automatically scales, running separate instances for concurrent requests.

---

### Q12. What is a Lambda handler function? Explain the `event` and `context` parameters.

**Answer:**
The handler is the entry point that AWS Lambda calls when the function is invoked.

```python
def lambda_handler(event, context):
```

- **`event`** - A dictionary containing the input data. For API Gateway, it includes the HTTP request body, headers, query parameters, etc.
- **`context`** - A runtime object providing metadata such as:
  - `context.function_name` - Name of the Lambda function
  - `context.memory_limit_in_mb` - Allocated memory
  - `context.get_remaining_time_in_millis()` - Time left before timeout

---

### Q13. What are Lambda cold starts? How do they affect this project?

**Answer:**
A **cold start** occurs when Lambda creates a new execution environment for a function that hasn't been invoked recently. This involves:
1. Downloading the deployment package
2. Setting up the runtime (Python interpreter)
3. Running initialization code (e.g., `import boto3`)

**Impact:** The first API call after inactivity may take 1-3 seconds longer. Subsequent calls reuse the warm container.

**Mitigation strategies:**
- Use **Provisioned Concurrency** to keep instances warm.
- Keep the deployment package small.
- Move initialization code (like `boto3.resource()`) outside the handler so it runs only once per container.

---

### Q14. What is the maximum execution time for a Lambda function?

**Answer:**
The maximum timeout for a Lambda function is **15 minutes (900 seconds)**. The default timeout is **3 seconds**. For this project, both Lambda functions complete well within a few seconds since DynamoDB operations are fast.

---

### Q15. How does Lambda scaling work? What happens if 1,000 students submit data simultaneously?

**Answer:**
Lambda scales automatically by running separate instances concurrently:
- **Burst concurrency limit:** 500-3,000 instances immediately (varies by region).
- **Scaling rate:** After the burst, it scales by 500 instances per minute.
- **Account concurrency limit:** Default 1,000 concurrent executions (can be increased).

If 1,000 students submit data at once, Lambda creates up to 1,000 concurrent instances, each processing one request. If the concurrency limit is reached, additional requests are throttled and return a 429 error.

---

### Q16. **Scenario:** Your `insertStudentData` Lambda function works in testing but fails in production with an "Access Denied" error. How do you troubleshoot?

**Answer:**
Step-by-step troubleshooting:
1. **Check CloudWatch Logs** - Lambda logs all invocations. Look for the exact error message.
2. **Verify IAM Role** - Ensure the `LambdaDynamoDBRole` is correctly attached and has `AmazonDynamoDBFullAccess`.
3. **Check Resource Policy** - Ensure the Lambda's execution role has permission to access the specific DynamoDB table.
4. **Verify Table Name** - Ensure the table name in code (`studentData`) matches the actual table name (case-sensitive).
5. **Check Region** - Ensure the Lambda function and DynamoDB table are in the same region, or the correct region is specified in the code.

---

### Q17. What is the difference between synchronous and asynchronous Lambda invocation?

**Answer:**
| Type | Behavior | Example |
|------|----------|---------|
| **Synchronous** | Caller waits for the response. Lambda returns the result directly. | API Gateway → Lambda |
| **Asynchronous** | Caller does not wait. Lambda queues the event and processes it. Events may be retried (2 retries). | S3 Events, SNS |

In this project, API Gateway invokes Lambda **synchronously** - the browser waits for the response.

---

### Q18. How do environment variables work in Lambda? How would you use them in this project?

**Answer:**
Lambda environment variables are key-value pairs available at runtime via `os.environ`. Instead of hardcoding values, I would use them for:

```python
import os
TABLE_NAME = os.environ['TABLE_NAME']
table = dynamodb.Table(TABLE_NAME)
```

Then set `TABLE_NAME=studentData` in the Lambda configuration. Benefits:
- Change table names without modifying code.
- Use different tables for dev/staging/prod environments.
- Store configuration securely (can be encrypted with KMS).

---

### Q19. What are Lambda Layers? When would you use one?

**Answer:**
Lambda Layers are ZIP archives containing libraries, custom runtimes, or dependencies that can be shared across multiple functions. Benefits:
- Reduces deployment package size.
- Shares common code across functions.
- Enables independent versioning of dependencies.

**Use case for this project:** If both Lambda functions needed a shared utility library or a specific version of `boto3`, I would package it as a Layer rather than including it in each function's deployment package.

---

### Q20. **Scenario:** The `getStudentData` function returns data successfully but the response is very slow (5+ seconds). How would you optimize it?

**Answer:**
Optimization strategies:
1. **Increase Lambda memory** - More memory = more CPU. Bumping from 128 MB to 512 MB can significantly reduce execution time.
2. **Use `ProjectionExpression`** to return only needed attributes:
   ```python
   response = table.scan(ProjectionExpression='studentid, #n, #c, age',
                          ExpressionAttributeNames={'#n': 'name', '#c': 'class'})
   ```
3. **Enable DynamoDB Accelerator (DAX)** for in-memory caching.
4. **Implement API-level caching** in API Gateway.
5. **Use Provisioned Concurrency** to eliminate cold starts.
6. **Paginate results** instead of returning all records at once.

---

## Section 3: API Gateway (Questions 21-28)

### Q21. What is Amazon API Gateway? What role does it play in this project?

**Answer:**
Amazon API Gateway is a fully managed service for creating, publishing, and managing RESTful and WebSocket APIs. In this project:
- It serves as the **entry point** for all HTTP requests from the frontend.
- **POST requests** are routed to the `insertStudentData` Lambda function.
- **GET requests** are routed to the `getStudentData` Lambda function.
- It handles request validation, CORS, throttling, and response formatting.

---

### Q22. What is CORS and why is it needed in this project?

**Answer:**
**CORS (Cross-Origin Resource Sharing)** is a browser security mechanism that restricts web pages from making requests to a different domain than the one that served the page.

It is needed because:
- The frontend is hosted on S3 (e.g., `http://bucket-name.s3-website.amazonaws.com`)
- The API is on API Gateway (e.g., `https://api-id.execute-api.region.amazonaws.com`)
- These are **different origins**, so the browser blocks the requests unless CORS headers are present.

Required CORS headers:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: Content-Type
```

---

### Q23. What is the difference between REST API and HTTP API in API Gateway?

**Answer:**
| Feature | REST API | HTTP API |
|---------|----------|----------|
| **Cost** | Higher (~$3.50/million requests) | Lower (~$1.00/million requests) |
| **Latency** | Higher | Up to 60% lower |
| **Features** | Full (caching, WAF, request validation, API keys) | Basic (simpler, faster) |
| **Auth** | IAM, Cognito, Lambda Authorizers, API Keys | IAM, Cognito, JWT |
| **Use Case** | Enterprise APIs needing full features | Simple, cost-effective APIs |

For this project, **HTTP API** is sufficient since we only need basic GET/POST routing to Lambda.

---

### Q24. What are API Gateway stages? How did you use them?

**Answer:**
Stages are named references to a deployment of the API (e.g., `dev`, `staging`, `prod`). Each stage has its own:
- Invoke URL
- Stage variables
- Throttling settings
- Caching configuration

In this project, I created a `prod` stage. The invoke URL becomes:
```
https://api-id.execute-api.eu-west-2.amazonaws.com/prod
```

---

### Q25. **Scenario:** Your API Gateway is receiving 10,000 requests per second and users are getting 429 errors. What do you do?

**Answer:**
The 429 error means **Too Many Requests** (throttling). Steps to resolve:
1. **Check default throttle limits** - API Gateway has a default limit of 10,000 requests/second per account per region and 5,000 burst.
2. **Request a limit increase** through AWS Support.
3. **Enable API Gateway caching** to reduce the load on Lambda and DynamoDB.
4. **Implement usage plans and API keys** to rate-limit individual consumers.
5. **Add CloudFront** in front of API Gateway for edge caching.
6. **Optimize Lambda and DynamoDB** to reduce response time, freeing up concurrent connections faster.

---

### Q26. How does API Gateway handle request/response transformation?

**Answer:**
API Gateway can transform requests and responses using **mapping templates** (Velocity Template Language for REST API):

- **Request transformation:** Convert or enrich incoming data before passing to Lambda.
- **Response transformation:** Format Lambda's output before sending to the client.

In this project, the API Gateway passes the POST body directly to Lambda as the event object and returns Lambda's response as JSON to the browser.

---

### Q27. What is API Gateway throttling and how does it protect the backend?

**Answer:**
Throttling limits the number of API requests to prevent abuse and protect backend resources:
- **Account-level:** 10,000 requests/second (default).
- **Per-method/per-client:** Configurable via Usage Plans.
- **Burst limit:** 5,000 requests.

When limits are exceeded, API Gateway returns a `429 Too Many Requests` response. This prevents:
- DynamoDB from being overwhelmed with write operations.
- Lambda concurrency limits from being exhausted.
- Unexpected cost spikes from malicious or accidental traffic surges.

---

### Q28. **Scenario:** A POST request to add a student works from Postman but fails from the browser. What could be wrong?

**Answer:**
This is almost certainly a **CORS issue**. The browser enforces CORS, but Postman does not.

Steps to fix:
1. **Enable CORS in API Gateway** for the POST method.
2. Ensure the **OPTIONS preflight method** is configured (browsers send an OPTIONS request before POST).
3. Verify the Lambda response includes CORS headers:
   ```python
   return {
       'statusCode': 200,
       'headers': {
           'Access-Control-Allow-Origin': '*',
           'Access-Control-Allow-Headers': 'Content-Type',
           'Access-Control-Allow-Methods': 'POST, OPTIONS'
       },
       'body': json.dumps('Student data saved!')
   }
   ```
4. Check the browser console for the specific CORS error message.

---

## Section 4: AWS S3 (Questions 29-35)

### Q29. What is Amazon S3? How is it used for static website hosting?

**Answer:**
Amazon S3 (Simple Storage Service) is an object storage service. For static website hosting:
- Upload HTML, CSS, JS files to a bucket.
- Enable **Static Website Hosting** in bucket properties.
- Set the **Index document** (e.g., `index.html`).
- Apply a **Bucket Policy** to allow public read access.
- Access the site via the S3 website endpoint URL.

S3 provides 99.999999999% (11 nines) durability and scales automatically to handle any traffic.

---

### Q30. Explain the S3 Bucket Policy used in this project.

**Answer:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
        }
    ]
}
```

- **Effect: Allow** - Permits the specified action.
- **Principal: \*** - Anyone (public access).
- **Action: s3:GetObject** - Only allows reading objects (not listing, deleting, or uploading).
- **Resource: .../*\*** - Applies to all objects within the bucket.

This policy makes the website files publicly readable while preventing unauthorized modifications.

---

### Q31. What is the difference between S3 Bucket Policy and IAM Policy?

**Answer:**
| Feature | Bucket Policy | IAM Policy |
|---------|--------------|------------|
| **Attached to** | S3 bucket (resource-based) | IAM user/role/group (identity-based) |
| **Controls** | Who can access this specific bucket | What this user/role can access |
| **Cross-account** | Yes (can grant access to other accounts) | No (only within the same account) |
| **Anonymous access** | Yes (Principal: "*") | No |

In this project, the **Bucket Policy** grants public read access, while the **IAM Policy** on `LambdaDynamoDBRole` grants Lambda access to DynamoDB.

---

### Q32. What are S3 Storage Classes? Which one is appropriate for this project?

**Answer:**
| Storage Class | Use Case | Cost |
|---------------|----------|------|
| **S3 Standard** | Frequently accessed data | Highest |
| **S3 Intelligent-Tiering** | Unknown/changing access patterns | Auto-optimized |
| **S3 Standard-IA** | Infrequent access, rapid retrieval | Lower |
| **S3 One Zone-IA** | Infrequent access, single AZ | Even lower |
| **S3 Glacier** | Long-term archival (minutes to hours retrieval) | Very low |
| **S3 Glacier Deep Archive** | Rarely accessed (12+ hours retrieval) | Lowest |

**S3 Standard** is appropriate for this project since the website files are accessed frequently by users.

---

### Q33. **Scenario:** After uploading files to S3 and enabling static hosting, the website shows "403 Forbidden." How do you fix it?

**Answer:**
Common causes and fixes:
1. **Missing Bucket Policy** - Add the public read policy (`s3:GetObject` for `Principal: *`).
2. **Block Public Access enabled** - Go to the bucket's **Permissions** tab → Disable **Block all public access**.
3. **Incorrect Index document** - Verify the **Static website hosting** setting has `index.html` as the index document.
4. **Wrong URL** - Use the **S3 website endpoint** (e.g., `http://bucket.s3-website-region.amazonaws.com`), not the S3 object URL.
5. **Object ACLs** - If using ACLs, ensure objects are set to public-read (though Bucket Policy is preferred).

---

### Q34. How would you improve the security of the S3-hosted frontend?

**Answer:**
1. **Use CloudFront** as a CDN in front of S3:
   - Enables HTTPS (S3 website endpoints only support HTTP).
   - Use Origin Access Control (OAC) to keep the bucket private.
   - CloudFront serves the content; S3 is not directly accessible.
2. **Remove public bucket policy** and restrict access to CloudFront only.
3. **Enable S3 versioning** to recover from accidental overwrites.
4. **Enable S3 access logging** to monitor who accesses the files.
5. **Use AWS WAF** with CloudFront to block malicious requests.

---

### Q35. What is S3 versioning and when would you enable it?

**Answer:**
S3 Versioning keeps multiple versions of an object in the bucket. When enabled:
- Uploading a file with the same name creates a new version instead of overwriting.
- Deleted objects are marked with a delete marker and can be restored.
- Protects against accidental deletions and overwrites.

**Use case:** Enable versioning on the website bucket to roll back to a previous version of `scripts.js` or `index.html` if a bad deployment occurs.

---

## Section 5: IAM and Security (Questions 36-40)

### Q36. What is the Principle of Least Privilege? Is it followed in this project?

**Answer:**
The Principle of Least Privilege states that entities should have **only the minimum permissions** required to perform their tasks.

In this project, the `LambdaDynamoDBRole` uses `AmazonDynamoDBFullAccess`, which grants **full access** to all DynamoDB tables. This violates least privilege.

**Better approach - Custom policy:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:Scan"
            ],
            "Resource": "arn:aws:dynamodb:eu-west-2:ACCOUNT-ID:table/studentData"
        }
    ]
}
```
This allows only `PutItem` and `Scan` on the specific `studentData` table.

---

### Q37. What is an IAM Role vs. an IAM User? Why does Lambda use a Role?

**Answer:**
| Feature | IAM User | IAM Role |
|---------|----------|----------|
| **Credentials** | Long-term (access keys, password) | Temporary (STS tokens) |
| **Used by** | People | AWS services, applications |
| **Authentication** | Access key/secret key | Assumed via `sts:AssumeRole` |

Lambda uses a **Role** because:
- It is an AWS service, not a person.
- Roles provide temporary security credentials that are automatically rotated.
- No need to embed or manage access keys in code.

---

### Q38. **Scenario:** A developer accidentally pushes the AWS API endpoint with hardcoded credentials to GitHub. What should you do?

**Answer:**
Immediate actions:
1. **Rotate the credentials** - Go to IAM and deactivate/delete the compromised access keys immediately.
2. **Check CloudTrail** for any unauthorized activity using the leaked credentials.
3. **Revoke active sessions** associated with the compromised credentials.
4. **Remove the sensitive data** from the Git history using `git filter-branch` or BFG Repo Cleaner.
5. **Enable AWS Config rules** to detect publicly accessible resources.

Preventive measures:
- Use **environment variables** or **AWS Secrets Manager** instead of hardcoding.
- Set up **git-secrets** to prevent committing credentials.
- Enable **GuardDuty** for threat detection.

---

### Q39. What is AWS CloudTrail and how would it help debug issues in this project?

**Answer:**
AWS CloudTrail records all API calls made in your AWS account, providing an audit trail. It helps:
- **Debug permission issues** - See exactly which API call failed and why.
- **Track changes** - Know who modified the DynamoDB table, Lambda function, or S3 bucket.
- **Security auditing** - Detect unauthorized access or unusual activity.
- **Compliance** - Maintain logs for regulatory requirements.

Example: If `insertStudentData` fails, CloudTrail would show the `dynamodb:PutItem` API call, the IAM role used, and the error message.

---

### Q40. **Scenario:** You need to give a new developer access to only the `studentData` DynamoDB table and nothing else. How would you set this up?

**Answer:**
1. Create a **new IAM User** for the developer.
2. Create a **custom IAM policy:**
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "dynamodb:GetItem",
                   "dynamodb:PutItem",
                   "dynamodb:UpdateItem",
                   "dynamodb:DeleteItem",
                   "dynamodb:Scan",
                   "dynamodb:Query"
               ],
               "Resource": "arn:aws:dynamodb:eu-west-2:ACCOUNT-ID:table/studentData"
           }
       ]
   }
   ```
3. Attach this policy to the developer's IAM user.
4. Optionally, add a **condition** to restrict access by IP address or time.

---

## Section 6: Frontend & JavaScript (Questions 41-44)

### Q41. How does the frontend communicate with the backend in this project?

**Answer:**
The frontend uses the **Fetch API** in JavaScript to make HTTP requests to the API Gateway endpoint:

- **Adding a student (POST):**
  ```javascript
  fetch(API_ENDPOINT, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(inputData),
  })
  ```

- **Retrieving students (GET):**
  ```javascript
  fetch(API_ENDPOINT)
      .then(response => response.json())
      .then(data => { /* populate table */ })
  ```

The flow: Browser → HTTPS → API Gateway → Lambda → DynamoDB → Lambda → API Gateway → Browser.

---

### Q42. What is the Fetch API? How is it different from XMLHttpRequest?

**Answer:**
| Feature | Fetch API | XMLHttpRequest |
|---------|-----------|----------------|
| **Syntax** | Promise-based (`.then()/.catch()`) | Callback-based (`onreadystatechange`) |
| **Readability** | Clean and modern | Verbose and complex |
| **Streaming** | Supports `ReadableStream` | No native streaming |
| **Error Handling** | Does not reject on HTTP errors (404, 500) | Fires `onerror` for network failures |
| **Browser Support** | Modern browsers | All browsers including legacy |

Fetch is used in this project for its cleaner syntax and native Promise support.

---

### Q43. **Scenario:** A user submits the add student form but sees "Error saving student data." The browser console shows `net::ERR_CONNECTION_REFUSED`. What could be wrong?

**Answer:**
`ERR_CONNECTION_REFUSED` means the browser cannot reach the API endpoint at all. Possible causes:
1. **Incorrect API_ENDPOINT URL** in `scripts.js` - Verify the URL is correct and the API is deployed.
2. **API Gateway not deployed** - The API exists but hasn't been deployed to a stage.
3. **API Gateway deleted or misconfigured** - The endpoint no longer exists.
4. **Network/firewall issue** - The user's network is blocking HTTPS requests to AWS.
5. **Wrong region in the URL** - The API was deployed in a different region.

**Debug steps:** Open the API URL directly in the browser. If it returns a response (even an error), the endpoint is reachable.

---

### Q44. How would you add input validation to the Add Student form?

**Answer:**
Both **frontend** and **backend** validation should be implemented:

**Frontend (in `scripts.js`):**
```javascript
function validateInput(data) {
    if (!data.studentid || data.studentid.trim() === '') {
        alert('Student ID is required');
        return false;
    }
    if (!data.name || data.name.trim() === '') {
        alert('Name is required');
        return false;
    }
    if (isNaN(data.age) || data.age < 1 || data.age > 100) {
        alert('Age must be a number between 1 and 100');
        return false;
    }
    return true;
}
```

**Backend (in Lambda):**
```python
def lambda_handler(event, context):
    required_fields = ['studentid', 'name', 'class', 'age']
    for field in required_fields:
        if field not in event or not event[field]:
            return {
                'statusCode': 400,
                'body': json.dumps(f'Missing required field: {field}')
            }
```

---

## Section 7: Architecture & Scenario-Based (Questions 45-50)

### Q45. **Scenario:** Your manager asks you to make this application production-ready. What changes would you make?

**Answer:**
1. **Security:**
   - Add AWS Cognito for user authentication.
   - Apply Principle of Least Privilege to IAM roles.
   - Use CloudFront with HTTPS instead of direct S3 website hosting.
   - Enable AWS WAF for request filtering.

2. **Reliability:**
   - Add input validation on both frontend and backend.
   - Implement error handling and retry logic in Lambda.
   - Enable DynamoDB point-in-time recovery (PITR).
   - Set up CloudWatch alarms for errors and throttling.

3. **Performance:**
   - Enable API Gateway caching.
   - Use CloudFront as a CDN for the frontend.
   - Implement pagination for large datasets.
   - Consider DynamoDB DAX for caching reads.

4. **Operations:**
   - Set up CI/CD with AWS CodePipeline or GitHub Actions.
   - Use Infrastructure as Code (CloudFormation or Terraform).
   - Enable CloudWatch Logs and X-Ray tracing.
   - Implement different environments (dev, staging, prod).

---

### Q46. **Scenario:** The application needs to support 10 schools, each with their own students. How would you redesign the DynamoDB table?

**Answer:**
**Option 1: Composite Primary Key**
- **Partition Key:** `schoolid` (String)
- **Sort Key:** `studentid` (String)

This allows:
- **Query all students in a school:** `KeyConditionExpression: schoolid = :schoolid`
- **Get a specific student:** `Key: { schoolid: 'school1', studentid: 'S001' }`
- Natural data isolation per school with efficient queries.

**Option 2: Single Table Design**
Keep `studentid` as partition key but prefix it with school: `school1#S001`. Add a GSI on `schoolid` for school-based queries.

**Option 1 is preferred** as it provides natural partitioning and efficient queries without additional indexes.

---

### Q47. **Scenario:** You need to send an email notification to the admin whenever a new student is added. How would you implement this?

**Answer:**
Use **Amazon SNS (Simple Notification Service)** with Lambda:

1. Create an **SNS Topic** (e.g., `NewStudentNotification`).
2. Subscribe the admin's email to the topic.
3. Modify the `insertStudentData` Lambda to publish to SNS after saving:
   ```python
   import boto3

   sns = boto3.client('sns')

   def lambda_handler(event, context):
       # ... save to DynamoDB ...

       sns.publish(
           TopicArn='arn:aws:sns:eu-west-2:ACCOUNT-ID:NewStudentNotification',
           Subject='New Student Added',
           Message=f"New student registered:\nID: {event['studentid']}\nName: {event['name']}"
       )

       return {'statusCode': 200, 'body': json.dumps('Student saved!')}
   ```
4. Update the Lambda IAM role to include `sns:Publish` permission.

---

### Q48. **Scenario:** The application needs to export all student data as a CSV file daily. How would you architect this?

**Answer:**
Use **Amazon EventBridge** (scheduled rule) + **Lambda** + **S3**:

1. **EventBridge Rule** - Create a scheduled rule that fires daily at midnight:
   ```
   cron(0 0 * * ? *)
   ```
2. **Lambda Function** - Triggered by EventBridge:
   ```python
   import csv
   import io
   import boto3
   from datetime import datetime

   def lambda_handler(event, context):
       dynamodb = boto3.resource('dynamodb')
       s3 = boto3.client('s3')
       table = dynamodb.Table('studentData')

       response = table.scan()
       students = response['Items']

       output = io.StringIO()
       writer = csv.DictWriter(output, fieldnames=['studentid', 'name', 'class', 'age'])
       writer.writeheader()
       writer.writerows(students)

       filename = f"exports/students_{datetime.now().strftime('%Y-%m-%d')}.csv"
       s3.put_object(Bucket='student-exports-bucket', Key=filename, Body=output.getvalue())

       return {'statusCode': 200, 'body': 'Export complete'}
   ```
3. **S3 Bucket** - Store the CSV files with lifecycle policy to delete after 30 days.

---

### Q49. **Scenario:** You need to add a "Delete Student" feature. Walk through the full implementation from frontend to backend.

**Answer:**

**1. Frontend (fetch_all_students.html):**
Add a Delete button in each table row:
```javascript
row.innerHTML = `
    <td>${student.studentid}</td>
    <td>${student.name}</td>
    <td>${student.class}</td>
    <td>${student.age}</td>
    <td><button onclick="deleteStudent('${student.studentid}')">Delete</button></td>
`;
```

**2. Frontend (scripts.js):**
```javascript
function deleteStudent(studentId) {
    if (!confirm('Are you sure you want to delete this student?')) return;

    fetch(API_ENDPOINT, {
        method: "DELETE",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ studentid: studentId })
    })
    .then(response => response.json())
    .then(() => {
        alert('Student deleted successfully');
        document.getElementById("getstudents").click(); // Refresh table
    })
    .catch(() => alert('Error deleting student'));
}
```

**3. Lambda Function (`deleteStudentData`):**
```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('studentData')

def lambda_handler(event, context):
    student_id = event['studentid']

    table.delete_item(Key={'studentid': student_id})

    return {
        'statusCode': 200,
        'headers': {'Access-Control-Allow-Origin': '*'},
        'body': json.dumps('Student deleted successfully')
    }
```

**4. API Gateway:**
- Add a **DELETE** method to the existing API.
- Integrate it with the `deleteStudentData` Lambda.
- Enable CORS for the DELETE method.
- Redeploy the API to the `prod` stage.

---

### Q50. **Scenario:** Your company wants to deploy this same application across multiple AWS regions for global users with low latency. How would you architect this?

**Answer:**

**Multi-Region Architecture:**

1. **Frontend (S3 + CloudFront):**
   - Host the S3 website bucket in one region.
   - Use **Amazon CloudFront** (global CDN) to cache and serve the frontend from 400+ edge locations worldwide.
   - Users in any region get fast page loads.

2. **Backend (API Gateway + Lambda):**
   - Deploy API Gateway and Lambda in multiple regions (e.g., `us-east-1`, `eu-west-2`, `ap-southeast-1`).
   - Use **Amazon Route 53** with **latency-based routing** to direct users to the nearest API region.

3. **Database (DynamoDB Global Tables):**
   - Enable **DynamoDB Global Tables** to auto-replicate the `studentData` table across selected regions.
   - Provides multi-region, multi-active database with single-digit millisecond reads/writes locally.
   - Automatic conflict resolution using last-writer-wins.

4. **Architecture Diagram:**
   ```
   User (US)  ──► Route 53 ──► API Gateway (us-east-1) ──► Lambda ──► DynamoDB (us-east-1)
                                                                              ↕ (Global Tables replication)
   User (EU)  ──► Route 53 ──► API Gateway (eu-west-2)  ──► Lambda ──► DynamoDB (eu-west-2)
                                                                              ↕
   User (Asia)──► Route 53 ──► API Gateway (ap-southeast-1)──► Lambda ──► DynamoDB (ap-southeast-1)
   ```

5. **Benefits:**
   - Sub-100ms API latency for global users.
   - Automatic failover if a region goes down.
   - 99.999% availability with multi-region active-active setup.
   - Consistent data across all regions via Global Tables.

---

## Quick Reference: AWS Services Used

| Service | Purpose in This Project |
|---------|------------------------|
| **S3** | Static website hosting (HTML, CSS, JS) |
| **DynamoDB** | Student data storage (NoSQL) |
| **Lambda** | Backend logic (insert/retrieve students) |
| **API Gateway** | REST API endpoints (GET/POST) |
| **IAM** | Role-based access control for Lambda |
| **CloudWatch** | Monitoring and logging (automatic with Lambda) |

---

*Prepared for the AWS Student Management System project by Khushal Ravindra Bhavsar*
