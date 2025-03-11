<h1>Developing REST APIs with Amazon API Gateway<h/1>
  
<h2>Lab overview</h2>

In this lab, you will create a REST application programming interface (API) by using Amazon API Gateway.
<h3>What I learned:</h3>
<ol>
      <li>Create simple mock endpoints for REST APIs and use them in the cafe website.</li>
      <li>Enable Cross-Origin Resource Sharing (CORS)</li>
</ol>
<h2>Business Scenario</h2>

In the previous lab, I play the role of Sofía to build a web application for the café. As part of this process, I created an Amazon DynamoDB table that was named FoodProducts, where I stored information about café menu items.
I then loaded data that was formatted in JavaScript Object Notation (JSON) into the database table. The table structure looked similar to the following table:

<img width="959" alt="image" src="https://github.com/user-attachments/assets/81abef1a-fd40-43ac-87f1-6d888ed3a450" />

In the previous lab I also configured code that used the AWS SDK for Python (Boto3) to: Scan a DynamoDB table to retrieve product details. Return a single item by product name using get-item as a proof of concept. Create a Global Secondary Index (GSI) called special_GSI that I use to filter out menu items that are on offer and not out of stock.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/fa64f8c2-17ef-4650-abec-0ad02ad20a3a" />

      
In this lab, I will continue to play the role of Sofía. I will use Amazon API Gateway to configure mock data endpoints. There are three that I will create:
<ol>
    <li>[GET] /products (which will eventually invoke a DynamoDB table scan)</li>
    <li>[GET] /products/on_offer (which will eventually invoke a DynamoDB index scan and filter)</li>
    <li>[POST] /create_report (which will eventually trigger a batch process that will send out a report)</li>
</ol>
Then in the lab that follows this one, I will replace the mock endpoints with real endpoints (Lambda fountion), so that the web application can connect to the DynamoDB backend.
When I start the lab, the following resources were pre-created for me in the account. 

<img width="401" alt="image" src="https://github.com/user-attachments/assets/260db08a-acb7-41a5-87d2-6785980400f3" />

However, by the end of this lab, you will have created the following architecture: 

<img width="437" alt="image" src="https://github.com/user-attachments/assets/bfc9105c-bf2a-4fdf-ad0d-dfd4837983c3" />

Lets get started

<h2>Task 1: Preparing the development environment</h2>
In this first task, you will configure your AWS Cloud9 environment so that you can create the REST API.
Before I can start this lab, I must import some files and install some packages in the AWS Cloud9 environment that was prepared for me.
       Connect to the AWS Cloud9 IDE.
<ol>
          <li>From the Services menu, search for and select Cloud9. </li>
          <li>In the Cloud9 Instance pane, choose Open IDE. The AWS Cloud9 IDE loads in a new browser tab.</li>
</ol>
Download and extract the files that you will need for this lab.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/53791900-b93c-4053-883a-22273f0b7ca1" />

      In the terminal, run the following command:

wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCDEV-2-91558/04-lab-api/code.zip -P /home/ec2-user/environment

<img width="959" alt="image" src="https://github.com/user-attachments/assets/c1adb52e-366e-4e52-a9b6-9ef767382849" />

Notice that a code.zip file was downloaded to the AWS Cloud9 instance. The file is listed in the Environment window.
Extract the file: unzip code.zip

<img width="794" alt="image" src="https://github.com/user-attachments/assets/8bd95a04-6239-4bb7-8397-903131053e9b" />

Run the script that upgrades the versions of Python and the AWS CLI installed in your IDE environment, and also creates the cafe website in your AWS account. 
chmod +x resources/setup.sh && resources/setup.sh

The script will prompt you for the IP address by which your computer is known to the internet.
Use www.whatismyip.com to discover this address and then paste the IPv4 address into the command prompt and finish running the script.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/66068460-6fec-4674-ae28-dbbad41364cb" />

Verify the version of AWS CLI installed. In the AWS Cloud9 Bash terminal (at the bottom of the IDE), run the following command: aws –version  The output indicate that version 2 is installed. 

<img width="959" alt="image" src="https://github.com/user-attachments/assets/9ba6f6c5-0173-49dd-a0c1-1fbe69379596" />

Verify that the SDK for Python is installed. Run the following command:
pip show boto3

<img width="959" alt="image" src="https://github.com/user-attachments/assets/aba98194-adfd-4929-8fef-da50420de554" />

To Verify that the cafe website can be loaded in a browser tab.
<ol>
      <li>Load the website in a browser tab.</li>
      <li>In a browser tab, open the Amazon S3 console.</li>
      <li>Choose your bucket name, and then choose Objects.</li>
</ol>

If the files that the script just uploaded do not display, choose the refresh icon to view them.

Choose the index.html file.
<ol>
    <li>Copy the Object URL. It will be in the following format.</li>
    <li>https://<bucket-name>.s3.amazon.com/index.html</li>
    <li>Verify that the website displays by pasting the full URL into your browser.</li>
</ol>

<img width="959" alt="image" src="https://github.com/user-attachments/assets/af7045ab-d2c0-41de-b9a9-2138d6565ce6" />

Notice in the Browse Pastries section that there are two buttons. The "on offer" view displays by default and it shows six menu items.

<img width="956" alt="image" src="https://github.com/user-attachments/assets/9ba67a44-4855-4273-98e5-7c9846aeaad5" />

Select the "view all" view. Notice that many more menu items display 

<img width="959" alt="image" src="https://github.com/user-attachments/assets/730fc067-357a-4707-acc1-11810e9509dd" />

<h2>Task 2: Creating the first API endpoint (GET)</h2>
In this task, you will create a REST API called ProductsApi. You will also create the first of three resources for the API.
The first API resource will be called products. It will make a GET request so that the website can retrieve all rows from the FoodProducts DynamoDB database table. You will then deploy it in an API Gateway stage that's named prod. When a user visits the website, it will make an AJAX request and return a list of café menu items from API gateway (it will return mock data for now).
To complete all these tasks, you will use the SDK for Python.
In the AWS Cloud9 navigation pane, expand the python_3 directory and open the file named create_products_api.py.
<img width="959" alt="image" src="https://github.com/user-attachments/assets/60bfa753-a895-4b0d-8171-086f60be7b2a" />

On line 3, replace the (fill me in) with the correct value that will create an API Gateway client. Which is ‘apigateway’
Take a moment to analyze the first part of what this code will do when you run it (nameproductapi code): 
<ol>
          <li>Lines 5-24 create a REST API that's named ProductsApi, and a resource that's named products.</li>
          <li>Lines 28-33 create a method request of type GET in the products resource.</li>
</ol>
You will analyze what the additional lines of code accomplish later in this task.

Run the code.

Save the change to the file. Then, in the Bash terminal, go to the directory that contains the Python code file, and run the code.
      cd python_3
      python create_products_api.py
      
<img width="709" alt="image" src="https://github.com/user-attachments/assets/4acd8bc1-8e13-41a2-8c03-4a0225fd0e21" />

Return to the AWS Management Console browser tab, and open the API Gateway console.
Open the ProductsApi that you just created by choosing the link.
       Choose the GET method that you defined.
       You should see the details of the GET method execution in a graphical format.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/802dab19-cd65-4dc3-841e-f2c451989e77" />

<img width="959" alt="image" src="https://github.com/user-attachments/assets/9c01044c-859b-4f4d-b8ac-e54f7462eaf7" />

Take a moment to study the data flow in the GET method that you defined. 
On the left is the Client.
<ol>
      <li>Lines 28-33 - When you run the Test, the Method Request is sent to the URL in the Amazon Resource Name (ARN) detail. The request doesn't require any authorization to invoke it.</li>
      <li>Lines 50-58 - The Integration Request of type MOCK is invoked, and the mock endpoint receives the data.</li>
      <li>Lines 35-48 - The mock endpoint invokes the Integration Response, which invokes the Method Response.</li>
      <li>Lines 61-92 - The Method Response returns the REST API response back to the Client that the request originated from.</li>
<ol>

Analysis: To make it easier during this initial API development phase, you will use mock data. When you test the API call, it will not actually connect to the database. Instead, it will return the data that's hardcoded in the responseTemplate part of the code (lines 67-91). 
This approach reduces the scope of potential errors during testing. You can stay focused in this lab on ensuring that the REST API logic is well defined. 
However, the structure of this mock data intentionally matches the data structure that will appear in the next lab when Lambda will be interacting with the database table.
The key values will be mapped to the attributes that are defined in the DynamoDB table (which Icreated in the previous lab).
Note: attributes in DynamoDB are not primatives. Instead, they are wrapper objects (as shown in the example code below). This is why there is a slight difference between the key names in the JSON and the attribute names in DynamoDB. 

In the API Gateway console, choose the  TEST link, then scroll to the bottom and choose the Test button. 

In the panel on the right, you should see the following response body, response headers, and log information. 



























