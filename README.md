# Url_shortener


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
