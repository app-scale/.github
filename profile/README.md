## Overview
- This project involves receiving events (install/reattribution/event) generated from mobile advertising, storing them in MongoDB, and allowing clients to download and review the events that occurred over a specific period through a CSV file.
- The system was designed with independent scaling of each component, asynchronous processing for throttling, and cost-efficiency in mind, using an event-driven architecture.
    - **relay-api**: Designed to easily scale by adding servers in response to increased requests.
    - **relay-client-api**: Tasks that can lead to significant server resource usage are throttled, and in case of increased CSV requests, independent scale-up/out mechanisms can be employed.
    - **relay-processor**: Minimizes database load and optimizes performance by consuming Kafka messages in batches for bulk insert into database.

## System Design Diagram
![architecture](https://github.com/app-scale/.github/assets/16813807/aa93acf8-8274-4706-afba-43928e2bbee4)

## Monitoring Dashboard Example
![dashboard](https://github.com/yb-toy-relay/.github/assets/16813807/a2ae40b6-6921-4161-a319-a912279d314c)

## Skills Used
| category             | skills                                                                                                       |
|----------------------|--------------------------------------------------------------------------------------------------------------|
| Language             | Java17 / JavaScript / HTML / CSS                                                                             |
| Framework            | Springboot / React                                                                                           |
| Middleware           | Kafka / MongoDB / Schema Registry / Jenkins                                                                  |
| AWS                  | ElasticBeanstalk / S3 / MSK (kafka) / ALB / CloudWatch / CloudFront / S3 / EventBridge / Glue / CodePipeline |
| ETC                  | Git / Jira / Slack                                                                                           |

## Repository & Application Component
|       component              |                                                     description                                                    |
|------------------------------|--------------------------------------------------------------------------------------------------------------------|
| relay-api                    | API servers that receives callback URL requests. Each request is written to Kafka one by one.                      |
| relay-processor              | Consumes messages from the Kafka topic every second, written by relay-api, and performs bulk inserts into MongoDB. |
| relay-client                 | A webpage with a form for requesting CSV files.                                                                    |
| relay-client-api             | API server (receiving CSV file request) and kafka consumer (uploading generated CSV file to S3).                   |
| relay-schema                 | The Avro schemas for messages delivered to Kafka topics are defined.                                               |
| relay-docker-compose         | docker-compose settings to easily set up all the infrastructure needed for a local development environment.        |


## Non-functional Requirements
1. **Scalability**
    - Each component is designed for independent scaling.
    - The **relay-api** is designed to easily scale by adding servers in response to increased requests.
    - The **relay-client-api** utilizes Kafka to handle CSV requests, allowing for scale-up and scale-out options to support system scalability.
2. **Availability**
    - High availability is supported using automatic scaling features of **AWS Application Load Balancer** and **AWS Elastic Beanstalk**.
    - Cloud-based infrastructure enables rapid data recovery in the event of a disaster, supported by data backup and recovery plans.
3. **Performance & Resource Constraints**
    - The **relay-processor** minimizes database load and optimizes performance by consuming Kafka messages in batches for bulk insert into MongoDB.
    - The system is designed to efficiently utilize limited resources, particularly with asynchronous messaging through Kafka to prevent server resource overload.
4. **Reliability**
    - System components are isolated, ensuring that issues with one component do not significantly impact the operation of the entire system.
    - **AWS S3** and **Event Bridge** are used to manage file processing and event communication reliably.
    - **AWS Glue Schema Registry** ensures compatibility of data exchanged via Kafka and facilitates maintenance.
5. **Security**
    - Security is enhanced for data transmission and access using **VPC Peering** and **presigned URLs**.


## FAQ (Fairly Asked Questions) about Application Components
- **Q) How many EC2 instances are used for each application server?**
    - 2 for relay-api, 1 for relay-processor, 1 for relay-client-api.
- **Q) Why 1 for relay-processor? Shouldn't there be at least two instances for availability?**
    - Cost reduction is the primary reason.
    - Since messages are already stored in Kafka for seven days, there is no concern about data loss if the relay-processor fails; simply restarting it suffices. However, if the failure of the relay-processor goes unnoticed, the latest data cannot be accessed. Therefore, a monitoring and alerting system has been built to ensure immediate awareness through alerts in case of process downtime.
- **Q) Why 1 for relay-client-api? Shouldn't there be at least two instances for availability?**
    - Cost reduction is the primary reason.
    - Generating CSV files from large datasets requires substantial memory usage and can heavily load the server, which might make it seem insufficient to operate with just one server.
    - Several precautions have been taken to manage the load on a single server:
      1. Limiting the query period to a maximum of two weeks.
      2. Implementing throttling by processing only one CSV file creation request at a time. The relay-client-api does not create the file immediately upon receiving a request; instead, it sends the request to Kafka. It then consumes the message it has sent from Kafka to proceed with the CSV file creation. This approach allows the server to handle only one CSV file generation at a time, ensuring stable service operation.
      3. Establishing monitoring and alerting systems to promptly recognize process downtimes.
    - However, if multiple CSV creation requests are received simultaneously, clients who request later may receive their results later. If this becomes an issue, comparing the timestamps at the time of consumption with those at the time of production can identify cases that take excessively long. If many such cases occur, we can modify it to process more than one CSV file creation request at a time along with scale-up/out.
- **Q) Why does the relay-processor use batch consumption every second?**
    - If the relay-processor consumes messages from the Kafka topic one by one and inserts them individually, it can cause significant load on the database and result in multiple unnecessary round trips.
    - The strategy of consuming messages in batches every second and using a single insert statement for each call minimizes unnecessary round trips and reduces the load on the database. This is a throttling strategy designed to optimize performance.
- **Q) Ultimately, the relay-client-api only needs to receive CSV file requests and deliver them to the user's email. The question arises as to why the tasks of receiving CSV requests and generating the CSV data to upload to S3 are treated as separate operations?**
    - It was determined that the advantages of separating the CSV request events from the actual CSV data generation process outweigh the disadvantages.
    - As explained earlier regarding why a single server for the relay-client-api is sufficient, a key point to reiterate is that generating CSV files containing large amounts of data simultaneously can increase the load on both the server and the database.
    - By enabling only one CSV file creation at a time, it is possible to significantly reduce the load on the server and database. To achieve this throttling, the relay-client-api first sends the CSV file creation request to Kafka and then processes the task by consuming only one CSV file creation request at a time from Kafka.
- **Q) Why use a schema registry?**
    - This is answered in the Cloud Components FAQ below.
 

## Cloud Components FAQ (Fairly Asked Questions)
- **Q) Why NoSQL?**
    - Mobile advertising data often contains a lot of unstructured data, and the data structure does not require joins when querying. Therefore, NoSQL is used for its flexibility.
- **Q) Why is MongoDB hosted on Atlas when all other infrastructure is based on AWS, even though AWS also offers MongoDB services?**
    - I didn't choose AWS DocumentDB because it only emulates the MongoDB API and operates on Amazon's Aurora backend platform. This leads to notable architectural constraints, limitations in functionality, and compromised compatibility.
    - source: https://www.mongodb.com/atlas-vs-amazon-documentdb
- **Q) Are Atlas and AWS, being different cloud providers, communicating over the public internet? If so, does that mean the database is exposed to the public?**
    - No, VPC peering ensures private connectivity between the AWS infrastructure and Atlas.
- **Q) Why ALB?**
    - To leverage the characteristics of OSI Layer 7 for routing, I used an Application Load Balancer instead of a Classic Load Balancer.
- **Q) Why did you choose Elastic Beanstalk instead of container-based deployment?**
    - Firstly, my understanding of Kubernetes isn't extensive. Secondly, the system wasn't large enough to warrant adopting a container-based deployment architecture like Kubernetes. I believed that introducing such a system would have been over-engineering.
- **Q) Why use a message broker at all? Given the functionalities, it seems like you could just insert data directly into the database from the API server and return database queries for CSV requests immediately.**
    - The message broker is used for asynchronous processing and throttling of any requests that could potentially strain the server.
- **Q) Why use Kafka?**
    - Not only for its performance aspects (zero-copy, sequential I/O) but also because, unlike SQS or RabbitMQ, it allows the re-reading of messages once they've already been read.
- **Q) Why use schema registry?**
    - It's used to ensure compatibility of messages exchanged between Kafka producers and consumers. The ability to optimize data transmission size is a by-product.
- **Q) Could you provide a more detailed explanation of the process where the relay-client-api generates a CSV file, uploads it to S3, and then delivers this file to the user's email?**
    - Events occurring within S3 can be forwarded to EventBridge. In other words, EventBridge can detect when a new file is uploaded to S3.
    - Utilizing the API Destination feature supported by EventBridge, it's possible to specify an API endpoint to call when a specific event occurs.
    - In this way, the relay-client-api receives information about the CSV file uploaded to S3 through another API endpoint. It then generates a URL for accessing the CSV file and sends this URL to the user's email.
    - The access URL for the CSV file included in the user's email is a presigned URL that is valid for only 1 hour, ensuring that all CSV files in S3 are not publicly accessible, and only the user who requested it through the presigned URL can access it for 1 hour.
