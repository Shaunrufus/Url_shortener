# System_Design


# Serverless URL Shortener (AWS)

A highly scalable, serverless URL shortener built on AWS, designed to handle high-read traffic with single-digit millisecond latency.

## 🏗 Architecture
<img width="1904" height="474" alt="image" src="https://github.com/user-attachments/assets/5ca765a2-c55f-4188-99c5-258cb6a11316" />


**Traffic Flow:**
1. **Client:** Makes a request via browser or API.
2. **Amazon CloudFront (CDN):** Caches popular short links globally for sub-10ms redirects.
3. **Amazon API Gateway:** Routes incoming HTTP requests (POST for creation, GET with path parameters for redirects).
4. **AWS Lambda:** Serverless compute running Python 3.x to execute core logic (Base62 encoding).
5. **Amazon DynamoDB:** NoSQL key-value store using `short_code` as the partition key for lightning-fast lookups.

## ⚙️ System Design 

### 1. Requirements
* **Functional:** Takes a long URL, returns a 7-character short link, and securely redirects users.
* **Non-Functional:** High availability (99.99% uptime) and extremely low latency (under 50ms per redirect).

### 2. Scale Estimation
* **Traffic:** Designed for 10 million requests/day (~116 requests per second).
* **Ratio:** Highly read-heavy system (estimated 100:1 read-to-write ratio).

### 3. Deep Dive: Collision Prevention
The system generates a random 7-character Base62 string, providing roughly 3.5 trillion unique combinations. To prevent the rare event of a collision, the Lambda function utilizes a `ConditionExpression` when writing to DynamoDB to ensure the `short_code` does not already exist before saving. 

### 4. Architectural Trade-Offs
* **DynamoDB (NoSQL) vs. RDS (SQL):** A URL shortener is fundamentally a massive key-value store. I chose DynamoDB over a relational database like PostgreSQL because we do not need complex table JOINs or strict ACID transactions (like moving money between bank accounts). DynamoDB guarantees single-digit millisecond reads at any scale and requires zero server maintenance.
* **Lambda vs. EC2:** URL shortener traffic is extremely spiky. EC2 requires provisioning servers that sit idle, whereas Lambda scales instantly to handle spikes and scales to zero when traffic stops, optimizing cost. The accepted trade-off here is the occasional "Cold Start" latency for the compute layer.

## 🚀 How to Run (API Endpoints)
**Create a Short Link (POST):**
`curl -X POST https://wfg22c5emk.execute-api.ap-south-2.amazonaws.com/prod -H "Content-Type: application/json" -d '{"long_url": "https://www. XYZ"}'`

**Use a Short Link (GET):**
Navigate to `https://<YOUR_API_GATEWAY_URL>/prod/{short_code}` in any browser to be redirected.






# Massive-Scale Notification System (AWS)

A highly decoupled, asynchronous event-driven architecture designed to handle massive traffic spikes and deliver notifications (SMS, Email) with zero data loss.

## 🏗 Architecture (Fan-Out Pattern)
<img width="1405" height="497" alt="image" src="https://github.com/user-attachments/assets/3cc1c0ca-2bf8-473d-8bb5-a1fef1e98c1f" />


**Traffic Flow:**
1. **Producer:** The core banking microservice publishes a single event payload to an Amazon SNS Topic.
2. **Fan-Out:** Amazon SNS pushes identical copies of the event to multiple subscribed SQS Queues simultaneously.
3. **Queuing (Durability):** Amazon SQS holds the messages in separate, dedicated queues (`SMS_Queue`, `Email_Queue`) to protect against downstream failures.
4. **Compute:** Independent AWS Lambda functions poll their respective queues, process the data, and format it for external API delivery.
5. **Delivery:** Notifications are dispatched via third-party providers (e.g., Twilio for SMS, Amazon SES for Email).

## ⚙️ System Design 

### 1. Requirements
* **Functional:** Accept triggering events from microservices and fan them out to multiple communication channels simultaneously.
* **Non-Functional:** Zero data loss (high durability), asynchronous processing (decoupled), and the ability to handle severe traffic spikes (high availability).

### 2. Scale Estimation
* **Throughput:** Designed for 100+ million notifications daily (averaging ~1,157 RPS, but capable of scaling instantly to 10,000+ RPS during spiky periods like payroll days).

### 3. Deep Dive: Fault Tolerance and Retries
To ensure zero data loss if a downstream provider (like Twilio) goes down, the system relies on the durability of Amazon SQS. If a Lambda worker fails to process a message or times out, the message remains in the SQS queue. SQS respects a `VisibilityTimeout` and will allow the Lambda to retry. If the message fails consecutively (e.g., 3 times), SQS automatically routes the payload to a Dead Letter Queue (DLQ) for engineering review, ensuring no alerts are permanently lost and the main queue is not bottlenecked by poison-pill messages.

### 4. Architectural Trade-Offs
* **Asynchronous SQS vs. Synchronous API Calls:** By placing SQS between the publisher and the workers, we trade immediate delivery confirmation for systemic reliability. The core application does not wait for the SMS to send before completing the user's transaction, drastically reducing latency for the end-user.
* **Single Responsibility Principle:** We utilize separate queues and separate Lambda functions for SMS and Email. The trade-off is slightly more infrastructure to manage, but the immense benefit is fault isolation. If the Email provider API crashes, the SMS pipeline continues to operate at 100% efficiency without coupling failures.

## 🚀 How to Test
1. Publish a JSON payload to the central SNS `BankingAlertsTopic`.
2. Monitor the automated fan-out to the attached SQS queues.
3. Verify the isolated processing via AWS CloudWatch logs for both the `SMS_Worker` and `Email_Worker` Lambdas.
