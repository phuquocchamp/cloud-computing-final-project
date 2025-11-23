## This is a system consisting of 3 components: Frontend, Backend & Database for practicing system deployment on AWS using the technologies & knowledge learned.
## Before you begin, ensure the project can run with docker-compose:
- `docker-compose -f docker-compose.yaml up -d`
- Then access `localhost:3000` to view the website. Try adding a few users and viewing the user list.

## Project Structure Explanation
### Frontend
- Node.js project responsible for listing, adding, and deleting users.
- Dockerfile builds an image based on Node20, exposing port 3000.
- Docker image accepts environment variable: REACT_APP_API_URL which is the Backend API URL. e.g.: localhost:8080
### Backend
- Java Spring Boot responsible for providing API to list, add, and delete users.
- Dockerfile builds an image running on Java OpenJDK, exposing port 8080.
- Docker image accepts environment variable: MONGO_URL which is the MongoDB URL. e.g.: mongodb://database:27017/dev (*since local Mongo doesn't set password, if using DocumentDB the connection URL will be different.)
### Database
- Uses Mongo:5.0 image, port 27017.

### Assignment Requirements: Deploy to AWS & configure CI/CD according to one of the following 2 options:
### Note: For the CI/CD part specifically, you can deploy as a monorepo or split frontend and backend into 2 separate repos.

### Option 1:
- Frontend: Server-side Rendering on ECS, ECR.
- Backend: ECS, ECR.
- Database: Document DB.
- Load Balancer: ALB
- CI/CD using one of the solutions: Jenkins, GitHub Actions, or CodePipeline.
- Deployment strategy for backend: Rolling update or Blue-Green.
### Option 2:
- Frontend: S3 + CloudFront.
- Backend: ECS, ECR.
- Database: Document DB.
- Load Balancer: ALB
- CI/CD using one of the solutions: Jenkins, GitHub Actions, or CodePipeline.
- Deployment strategy for backend: Rolling update or Blue-Green.



### Suggested approach for Architecture 1: Both Frontend & Backend deployed on ECS, DB: Document DB running Mongo, combined with ALB.

### Implementation Steps:
#### 1. Create network (VPC, Subnet), Security Group & ECS Cluster, ECR repository for FE, BE.
#### 2. Create Document DB (Mongo Engine version 5.0)
* Create a Custom Parameter group from Mongo 5.0, disable TLS=Disabled, and save. *Reason: The provided source code doesn't work with TLS Enabled mode of Mongo.
* Create a Document DB Cluster using Mongo Engine 5.0. *In the Document DB creation interface, select option: ```Instance Based Cluster```
* Choose instance size db.t3.medium, Engine: 5.0.0, Number of instances: 1
* Parameter group: Select the Parameter group created in the previous step.
* Set username/password for the Cluster. Note that username must be different from ```admin```
* Configure necessary Security group for the Cluster (Port 27017)
* Use an EC2 instance with mongosh pre-installed to test connection to the Database. Troubleshoot if there are any issues.  
<span style="color: red;">*Note: AWS DocumentDB currently does not support connections from local machines (via internet), so you must create an EC2 instance in the same VPC as MongoDB, install mongosh on it, then try connecting with the command</span> e.g.:  
`mongosh --host linh-test-db.cluster-cwpdzas1s9oa.ap-southeast-1.docdb.amazonaws.com:27017 --username linhadmin --password`  
Enter password, press Enter
* Reference AWS link: `https://docs.aws.amazon.com/documentdb/latest/developerguide/troubleshooting.connecting.html#troubleshooting.cannot-connect.public-endpoints`


#### 3. Create an Application Load Balancer
- Create Application Load Balancer, listener port 80 (or 443 if SSL is available).
- Create 2 target groups: 
- frontend-tg: Type IP, port 3000, default Healthcheck. 
- backend-tg: Type IP, port 8080, Healthcheck: /api/students overwrite health check port 8080
- Configure the Application Load Balancer so that rule /api/* points to backend-tg, and the rest defaults to frontend-tg

#### 4. Deploy Backend
- Build Docker image and push to ECR. 
- Create Backend Task definition, note to overwrite `MONGO_URL` for backend (note that password is stored in plaintext, needs improvement in the future using Secret Manager)
- Example: ```mongodb://linhadmin:thisismypassword@linh-mongo.cluster-cwpdzas1s9oa.ap-southeast-1.docdb.amazonaws.com:27017/dev```
- Create Backend Service, select backend-target-group, corresponding listener.
- Test API e.g. GET ```<alb-domain>:80/api/students```, result should return a list of students in JSON format.

#### 5. Deploy Frontend
- Build Frontend to create Docker image, push to ECR.

- Create Frontend Task definition, note to overwrite `REACT_APP_API_URL` so frontend can recognize the backend API with structure: ```<alb-domain>:80```
- Example: ```http://linh-test-alb-581342174.ap-southeast-1.elb.amazonaws.com:80```

- Create Frontend Service, select frontend-target-group, corresponding listener.

#### 6. Test connection to ALB & access the application, try adding/deleting users
- Sample URL: ```http://linh-test-alb-581342174.ap-southeast-1.elb.amazonaws.com:80```
#### 7. Optional: Configure CI/CD for the repo (monorepo or split into 2 repos FE, BE) using the knowledge learned.
- You can use Jenkins or CodePipeline to configure CI/CD for frontend & backend repositories.
- For each repository, configure the following 2 pipelines:
  + Pipeline 1: Automatically build & deploy whenever code is merged/pushed to the develop branch.
  + Pipeline 2: Manual build & deploy with specified branch/tag.
- Due to resource limitations, both pipelines will deploy to the same target environment. In real-world projects, the 2 pipelines would deploy to 2 different target environments.
