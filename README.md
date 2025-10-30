# AWS Auto Scaling and Load Balancing Project

This project demonstrates how to build a scalable, fault-tolerant, and highly available web application infrastructure on AWS. It uses an Application Load Balancer (ALB) to distribute incoming traffic across a fleet of EC2 instances, which are managed by an Auto Scaling Group (ASG) that responds to CPU load.

This project is a practical implementation of "Task 7: Configure Load Balancing and Auto Scaling" for a cloud internship program.

---

## 🎯 Objective

* **High Availability:** Ensure the application remains online even if a single server (EC2 instance) or an entire Availability Zone fails.
* **Scalability:** Automatically add more servers when traffic increases (scale-out) to handle the load.
* **Elasticity & Cost-Efficiency:** Automatically remove servers when traffic is low (scale-in) to save costs.
* **Fault Tolerance:** The load balancer automatically detects and stops sending traffic to unhealthy instances.

---

## 🛠️ Technology Stack

* **AWS EC2 (t2.micro):** Virtual servers running the web application.
* **AWS Application Load Balancer (ALB):** Distributes HTTP traffic to the EC2 instances.
* **AWS Auto Scaling Group (ASG):** Manages the number of EC2 instances based on scaling policies.
* **AWS AMI (Amazon Machine Image):** A "master" template of the web server used by the ASG to launch new instances.
* **AWS VPC (Virtual Private Cloud):** The isolated network environment for all resources.
* **AWS Security Groups:** A virtual firewall to control inbound and outbound traffic.
* **Apache2:** The web server software installed on the EC2 instances.
* **HTML:** The simple webpage served by the application.

---

## 🏗️ Architecture

The infrastructure works as follows:

1.  A **User** accesses the application using the **ALB's public DNS name**.
2.  The **ALB** (in public subnets) receives the request. It checks its **Target Group** for a healthy instance.
3.  The ALB forwards the request to a healthy **EC2 Instance** (in private subnets) in its Target Group.
4.  The **Auto Scaling Group** monitors the average **CPU Utilization** of all instances.
    * **If CPU > 60% (Scale-Out):** The ASG launches a new EC2 instance from its Launch Template (using the custom AMI).
    * **If CPU < 20% (Scale-In):** The ASG terminates an instance.
5.  The **Target Group** constantly runs **Health Checks** on all instances. If an instance fails, the ALB stops sending traffic to it, and the ASG eventually replaces it.


*(Note: You can create a simple diagram using a tool like diagrams.net and embed the image here.)*

---

## ⚙️ Configuration Steps

### 1. Create the Server Template (AMI)

1.  **Launch a `t2.micro` EC2 instance** (Ubuntu) in a public subnet.
2.  **Create a Security Group** (`Web-Server-SG`) allowing:
    * `SSH` (Port 22) from `My IP`.
    * `HTTP` (Port 80) from `Anywhere (0.0.0.0/0)` (This is for the load balancer health checks).
3.  **SSH into the instance** and install the web server:
    ```bash
    sudo apt update
    sudo apt install apache2 -y
    # Create the simple webpage
    echo '<html><body><h2>Hello from my AWS Instance!</h2></body></html>' | sudo tee /var/www/html/index.html
    ```
4.  **Test:** Verify the webpage is working by visiting the instance's Public IP.
5.  **Create an AMI:** From the EC2 Console, create an image (AMI) from this instance. This will be the "master copy" for all future servers.
6.  Terminate the original instance.

### 2. Set Up the Load Balancer

1.  **Create a Target Group** (`My-Web-Targets`):
    * Target type: `Instances`
    * Protocol: `HTTP` on Port `80`.
    * Health checks: `HTTP` on path `/`.
2.  **Create an Application Load Balancer** (`My-Web-ALB`):
    * Scheme: `Internet-facing`.
    * VPC: Your default or custom VPC.
    * Subnets: Select **at least two public subnets** (for High Availability).
    * Security Group: Use the `Web-Server-SG` created earlier.
    * Listeners: Listen on `HTTP:80` and set the default action to **forward to `My-Web-Targets`**.

### 3. Configure the Auto Scaling Group

1.  **Create a Launch Template** (`My-Web-Launch-Template`):
    * Select your `AMI` (from Step 1.5).
    * Instance type: `t2.micro`.
    * Security Group: Select the `Web-Server-SG`.
2.  **Create the Auto Scaling Group** (`My-Web-ASG`):
    * Select the `My-Web-Launch-Template`.
    * Network: The same VPC and subnets as the load balancer (or private subnets for better security).
    * Load Balancing: Attach to the `My-Web-Targets` Target Group.
    * Group Size:
        * **Desired:** `2`
        * **Minimum:** `1`
        * **Maximum:** `4`
    * Scaling Policy:
        * Type: `Target tracking scaling`
        * Metric: `Average CPU utilization`
        * Target value: `60` (This will add instances when average CPU is over 60%).

---

## 🔬 How to Test

1.  **Check Application:** Find the **DNS Name** for `My-Web-ALB` (on the Load Balancer dashboard) and paste it into your browser. You should see your webpage.
2.  **Test Fault Tolerance:** Go to the EC2 Instances list. **Terminate** one of the two instances. Refresh your browser (you may not even notice a delay). The load balancer will detect the failure and only send traffic to the healthy instance. The ASG will then launch a new instance to bring the count back to `2`.
3.  **Test Scaling (Optional):** Use a load testing tool like Apache Benchmark (`ab`) to stress the servers.
    ```bash
    ab -n 500 -c 50 http://your-load-balancer-dns-name/
    ```
    Watch the ASG "Activity" tab. After a few minutes of high CPU, you will see it automatically launch a third instance.

Create a file named index.html and paste this in:
<!DOCTYPE html>
<html>
<head>
  <title>Cloud Auto Scaling Demo</title>
  <style>
    body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; }
    h2 { color: #333; }
    p { color: #555; }
  </style>
</head>
<body>
  <h2>Hello from my Cloud Instance!</h2>
  <p>This page is served from an AWS EC2 instance behind a load balancer.</p>
  <p>This system is designed to scale automatically based on traffic.</p>
</body>
</html>
