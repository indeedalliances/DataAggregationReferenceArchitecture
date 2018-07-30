This demo is for use by Indeed's ATS partners to illustrate how to aggregate multiple data feeds in to a single XML file. While it is only a demo, it contains the vast majority of code and infrastructure that would be required to deploy a production system, regardless of size.

The architecture is designed to be deployed with Amazon Web Services. It uses two singleton autoscaling groups (i.e. they have both a minimum and maximum size of one server). One ASG manages the Feed Aggregator, and the other manages the FTP server that serves the aggregated file. For a detailed network diagram, refer to FeedAggregationReferenceArchitecture.pdf

In this demo, data feeds are represented by the three XML files stored in the "testdata" directory. The data feed endpoints are defined in the file "api_configs.json". If your ATS requires authentication or is a RESTful API, it will be your responsibility to modify the aggregation code as well as the configuration file format to accomodate your system. This should be the ONLY modification required to turn this demo in to a production system.

Once deployed, every hour the aggregation server will pull the data from the XML files on GitHub and combine them to a single XML file. Additionally, every 5 minutes a health check will execute on both the data feeds as well as the FTP server to ensure availability. If a health check fails, an email can be sent to a notification list, which is configured per data feed.

A CloudFormation template, "FeedAggregationReferenceArchitecture.yaml", is included that makes it straightforward to deploy the demo. Follow these steps to deploy it:

1. Download the contents of the "assets" directory to your local drive.
2. Create an AWS account.
3. Optional: In order to receive the health check failure emails, you will need register the both the "from" and "to" emails with AWS. The Simple Email Serivce is region-dependent, and currently the region is hard-coded in the HealthCheck code to use us-east-1. To register your email address, go to the AWS console here: https://console.aws.amazon.com/ses/home?region=us-east-1# Click on "Email Addresses" in the left-hand column, then click the "Verify a New Email Address". It is possible to use the same email for the "from" an "to" addresses. Note that if you use a corporate email address for the "from" address, your emails will likely go to the spam folder, becuase the sender domain differs from actual domain from which the email is sent. You will need to edit the "api_configs.json" to include the "from" and "to" emails. Note that you can include a comma-separated list of emails in the "to" address.
4. In any region, create an S3 bucket. This will hold the contents of your local "assets" directory.
5. Upload all the files from your local "assets" directory to the S3 bucket.
6. Create a KeyPair to use in order to SSH in to the EC2 instances. It must be created within the region that you intend to deploy the demo to. Note: you must deploy to a region that support AWS Elastic File System. As of this writing EFS is not supported in London, Paris, Singapore, Mumbai, Montreal, or São Paulo. For an up-to-date list of services supported by region, see: https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/

You are now ready to upload the CloudFormation template to AWS, which will deploy the resources for the live demo. Go to the CloudFormation console, and upload "FeedAggregationReferenceArchitecture.yaml". You will be required to answer some questions in order to complete the deployment.

Network Configuration questions:

	- The VPC in to which to deploy the demo. Be sure to use the region's unmodified default VPC.
	- The default subnets for the VPC availability zones be used for auto-scaling targets. You should select all of the default subnets in the VPC, which should be all of the subnets on the list, assuming no subnets have been added or removed.
	- The HTTP access, which is the CIDR block of IP addresses allowed to access the aggregatuon server via HTTP. Typically, you would set this the be the IP address of your workstation, so that you can send the aggregation server commnands over HTTP for testing (i.e., /HealthCheck and /GetJobs). For the demo, it's fine to leave this open (0.0.0.0/0).
	- The SSH access, which is the CIDR block of external IP addresses allowed to access the EC2 instances via SSH. In production, this would be the IP address of a Site Reliability Engineer's workstation, so they can inspect that state of the server. For the demo, it's fine to leave this open (0.0.0.0/0).
	- The FTP access, which is the CIDR block of IP external addresses allowed to access the FTP server. In production, this would be the IP address of Indeed's FTP robot. For the demo, it's fine to leave this open (0.0.0.0/0).

EC2 Configuration questions:

	- The name of the S3 Resources Bucket to which you uploaded the files in your "assets" directory.
	- EC2 instance type for the Aggregation server. For the demo, nano will work.
	- EC2 instance type for the FTP server. For the demo, nano will work.
	- The name of the SSH KeyPair to be used to access the EC2 instances.

FTP Configuration questions:

	- The FTP User Name that will be created in order to connect to the FTP server.
	- The FTP Password for the FTP user.
	- Be sure to make notes of both the above to connect to the FTP server!


Now click "Next" to go to the options page. You do not need to change any options, so click "Next" again to go to the Review page. On this page, check the box at the bottom that says "I acknowledge that AWS CloudFormation might create IAM resources." Then click "Create", and wait.

After the stack reached the CREATE_COMPLETE state, click on the Outputs tab. Here you will see the public IP addresses of the Aggregation and FTP servers, as well as the username and password that were chosen for the FTP account.

You can now log in to the FTP server and take look. Since the aggregation process executes every hours, there may not yet be any XML data available. But you will at least see two files: aggtouch.txt and ftptouch.txt. The reason these files are there is that they are created as the FTP and Aggregation servesr are brought online. THey are used to ensure that the FTP health check (which tests for a minimum number of files in the FTP directory) succeeds.

Now let's manually trigger the HealthCheck on the Aggregation server. Copy the aggaddress value and paste it in a browser directed at the HealthCheck on port 8080, i.e. http://xxx.xxx.xxx.xxx:8080/HealthCheck The server should respond immediately with the message "I am Healthy".

Now let's manually trigger the Aggregation process on the Aggregation server. Copy the aggaddress value and paste it in a browser directed at the GetJobs methods on port 8080, i.e. http://xxx.xxx.xxx.xxx:8080/GetJobs. The server should respond immediately with the message "Success!".

Now go back to the FTP server and confirm that you see the following XML files:

- one.xml is the job data from the first endpoint.
- two.xml is the job data from the second endpoint.
- three.xml is the job data from the third endpoint.
- jobs.xml is the data from one.xml, two.xml, and three.xml combined in to a single file.
- one more file in the form xxxxxxxxxxxxx_jobs.xml, where xxxxxxxxxxxxx is the UNIX date-time at which the file was created. This file will also have been run throuygh the prettyprint utility, to ensure that the XML is properly formatted.

At any given time, the files one.xml, two.xml, three.xml, and jobs.xml will walys be there. They serve as a way to audit the exact data that came back from the endpoints during the most recent aggregation process.

There will also be up to 40 files of the form xxxxxxxxxxxxx_jobs.xml; the aggregation process will delete the oldest files over the number 40. The number 40 is currently hard-coded in to the template, but might be parameterized in a future version.

Keep in mind that the names of the work files are all configured in the file api_configs.json.






