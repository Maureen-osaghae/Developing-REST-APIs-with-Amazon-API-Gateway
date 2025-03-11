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

<h3>Analysis:</h3>
To make it easier during this initial API development phase, you will use mock data. When you test the API call, it will not actually connect to the database. Instead, it will return the data that's hardcoded in the responseTemplate part of the code (lines 67-91). 
This approach reduces the scope of potential errors during testing. You can stay focused in this lab on ensuring that the REST API logic is well defined. 
However, the structure of this mock data intentionally matches the data structure that will appear in the next lab when Lambda will be interacting with the database table.
The key values will be mapped to the attributes that are defined in the DynamoDB table (which Icreated in the previous lab).
Note: attributes in DynamoDB are not primatives. Instead, they are wrapper objects (as shown in the example code below). This is why there is a slight difference between the key names in the JSON and the attribute names in DynamoDB. 

In the API Gateway console, choose the  TEST link, then scroll to the bottom and choose the Test button. 

In the panel on the right, you should see the following response body, response headers, and log information. 

<img width="733" alt="image" src="https://github.com/user-attachments/assets/934c7ff5-891f-40bc-9d35-3e5ee9181262" />

Congratulations! You have now successfully created and tested a REST API with a resource that makes a GET request. 

<h2>Task 3: Creating the second API endpoint (GET)</h2>
In this task, you will define another API endpoint of type GET. This endpoint will eventually support calls to/products/on_offer from the cafe website and it will return in stock items. 
In the AWS Cloud9 navigation pane, expand the python_3 directory and open the file named create_on_offer_api.py.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/4322a6f8-6d0f-41b8-b025-f71f791f1f2b" />

  <ol>
        <li>In a browser tab, go to the API Gateway console and choose the ProductsApi API that you created a moment ago.</li>
        <li>In the panel on the left, choose Resources.</li>
        <li>Choose GET under products</li>
        <li>In the breadcrumb navigation at the top of the screen (above the Actions menu), you can see APIs > ProductsAPI followed by an id in parenthesis. </li>
        <li>This is the api_id.</li>
        <li>On the same line, you will see /products, followed by another id in parenthesis. </li>
        <li>This is the resource parent_id </li>
  </ol>

<img width="959" alt="image" src="https://github.com/user-attachments/assets/9cb77570-0551-40f5-b653-ba839c1696a2" />

Create the API resource. Save the change to the file.
Then in the Bash terminal, verify that the current directory is python_3 and run the code.
python create_on_offer_api.py

<h3>Observe the results.</h3>
Return to the AWS Management Console browser tab, and open the API Gateway console. Choose the APIs link in the breadcrumb navigation above, then on the left, open the ProductsApi by choosing the link.
<img width="959" alt="image" src="https://github.com/user-attachments/assets/b3f04eac-7d26-4b92-9389-4d13f74046ae" />

Notice that there is now a nested resource called /on_offer under the /products resource. 

<img width="959" alt="image" src="https://github.com/user-attachments/assets/9ac82ea9-847a-406f-b3fa-e714cdc0c2dd" />

Test the /on_offer resource. Use the  Test link, the same way you tested the first resource in the previous task. You should receive a 200 HTML status code response. 

<img width="959" alt="image" src="https://github.com/user-attachments/assets/3bb1228f-cdff-4798-a4f8-576477bb8bb8" />

Congratuations! You now have two API resources the website will be able to use. 

<h2>Task 4: Creating the third API endpoint (POST)</h2>
In this task, you will create a third resource for the API, /create_report. This resource will be configured at the same level as /products (not as a nested resource under products).
Café staff who are logged in (authenticated) will later use this API resource to request an inventory report.
The report details will be discussed in later labs. However, for now, you will configure the API to support this feature. You will also test that the website can make an Asynchronous JavaScript and XML (AJAX) request.
It is fully expected that the AJAX request will fail when you test it because you haven't configured an authentication mechanism yet. However, you will configure authentication in a later lab.

In the AWS Cloud9 IDE, if the create_products_api.py file is not already open, open it (you ran this file in Task 2). Next, in the python_3 directory, also open the create_report_api.py file.
In the main code editor window, right-click the create_report_api.py file tab and choose Split Pane in Two Columns 

<img width="959" alt="image" src="https://github.com/user-attachments/assets/ca6594e7-70f2-40f9-8df0-b1dbb41dd477" />

Analyze and update the create_report_api.py code. Be sure to compare the code in this file to the create_products_api.py code while you do the analysis and updates. 

Tip: You could use the console as you did before to discover the api_id. However you can also use the AWS Command Line Interface (AWS CLI) to find the value of the API ID by running the following command: aws apigateway get-rest-apis --query items[0].id --output text

Analyze the rest of the create_report_api code while comparing it to the create_products_api code. The code in the two files looks similar, but they have some differences:

<ol>
      <li>The httpMethod that's invoked is POST (instead of GET).</li>
      <li>This code creates a new resource with a pathPart of create_report, instead of products.</li>
      <li>The product_integration_response defines three responseParameters. In create_report_api these parameters do not allow Cross-Origin Resource Sharing (CORS), whereas in 
          create_products_api they do allow it.</li>
      <li>The product_integration_response also hardcodes a response for testing purposes, though the user is not authenticated. (The purpose of the test is to ensure that the client can 
           receive a response.)</li>
</ol>

Save the changes to the file. In the next step, you will run the code in create_report_api.py. In the terminal, confirm that you are in the python_3 directory, and then run the code to create the third endpoint.

python create_report_api.py

In the API Gateway console, view the details of the report API that you configured. Return to the API Gateway console tab and refresh the page. Confirm that you are in ProductsApi.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/60706493-3b58-4103-b36c-75c17f64b06f" />

In the navigation pane, confirm that Resources is selected, and choose /create_report > POST.
You should see the details of the POST method execution.
Choose the  TEST link, then choose the Test button at the bottom of the screen. 
In the panel on the right, you should see the following response body, response headers, and log information. 

<img width="959" alt="image" src="https://github.com/user-attachments/assets/5bd32836-ff7c-4d67-aad7-63c72c09eac6" />

In the panel on the right, you should see the following response body, response headers, and log information. 

<h2>Task 5: Deploying the API</h2>
Now that you have defined all three resources in the API, the next step is to deploy the API.
Deploy the API.
<ol>
      <li>Still in the API Gateway console where you have the ProductsApi details open, under Resources select the root /</li>
      <li>From the Actions menu, choose Deploy API and then fill in the details:.</li>
      <li>Deployment stage: [New Stage].</li>
      <li>Stage name: prod</li>
      <li>Stage description: (leave blank)</li>
      <li>Deployment description: (leave blank)</li>
      <li>Choose Deploy</li>
</ol>

<img width="957" alt="image" src="https://github.com/user-attachments/assets/c5b67669-7f9f-4bf9-8152-4d21fc1ae4ee" />

<img width="959" alt="image" src="https://github.com/user-attachments/assets/89471b66-7798-428e-98ec-40344a3c529a" />

Copy the Invoke URL value to your clipboard. You will use it next.

<h2>Task 6: Updating the website to use the APIs</h2>
In this final task in the lab, you will update and then test the website files that are hosted on Amazon S3. After you complete these updates, the website will invoke the REST API that you just created. 
Update the website's config.js file. In the Cloud9 IDE, open resources/website/config.js
On line 2, replace null with the Invoke URL value you copied a moment ago. Be sure to surround it in quotes. Verify that prod appears at the end of the URL with no trailing slash.
Save the change to the file.

<img width="553" alt="image" src="https://github.com/user-attachments/assets/83c9d1e7-103f-4d8c-abe7-346c3f3c36ec" />

<img width="533" alt="image" src="https://github.com/user-attachments/assets/cc42f1d7-1cbb-4643-b831-146aa30e6c70" />

Load the latest café webpage with the developer console view exposed.

      If you still have the cafe website open in a browser tab, return to it. If you do not still have it open, reopen it now by following these steps:

<ol>
      <li>In the S3 console, choose the bucket that contains your website files.</li>
      <li>Choose index.html and then copy the Object URL.</li>
      <li>Load the Object URL in a new browser tab.</li>
      <li>Test and observe details about the website and the application logic. </li>
      <li>Scroll to the top of the Cafe website and choose login.</li>
</ol>
You will receive a message of "No API to call". This is expected. The authentication logic will be implemented in a later lab. The generate report call from the website will also be implemented in a later lab, from the webpage that will load after a successful login.

<img width="956" alt="image" src="https://github.com/user-attachments/assets/4b3a8da4-f06b-4a2e-910a-9fba8d804ed3" />

Notice that on offer is chosen by default.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/0aad50cd-e1a4-4b85-a2f7-33eff4c36152" />

Choose view all. You should now see three products listed (these match the mock data you set in the /products resource you configured in Task 2). 

<img width="959" alt="image" src="https://github.com/user-attachments/assets/55fd5acf-c21e-479e-ac81-038a4d7d0b6a" />

Congratulations! Your website is now making calls to the API that you created and deployed. 
Sofía is satisfied that she has made progress! 
After she successfully set up the website on Amazon S3, Sofía has been excited to improve the website's functionality. Her larger plan is to build a serverless dynamic website with a database backend. Sofía's plan has three major milestones.
<ol>
      <li>The first milestone was to create a database backend to store café data. She accomplished that in the previous lab by using DynamoDB.</li>
      <li>The second milestone is to create a REST API so that the webpages that are hosted on Amazon S3 can interact with the backend database. Sofía just completed the most difficult part of 
          that task during this lab.</li>
</ol>

The following diagram summarizes the features that Sofía has built in the last lab and in this lab:

<img width="424" alt="image" src="https://github.com/user-attachments/assets/2810fc98-b2e5-4b48-bf88-2ce18e30b512" />

Though the API currently uses mock data, it should be straightforward to replace the mock endpoints with actual endpoints that can communicate with the database. 
The third milestone will be accomplished in the next lab. Sofía will create AWS Lambda functions. The REST API resources that she created in this lab will trigger those Lambda functions to query the DynamoDB table. This database table contains the actual data that she stored in the previous lab.

<img width="430" alt="image" src="https://github.com/user-attachments/assets/89377d56-6154-4c47-b31e-4f39a3fa0d0e" />

Finally, in later labs in the course, Sofía will use Amazon Cognito to implement the authentication logic that the create_report API call expects.
Sofía knows that she has work to do. For now, though, Sofía decides to celebrate her most recent accomplishment by relaxing with her friends.

<h2>Lab complete </h2>

© 2024 Amazon Web Services, Inc. and its affiliates. All rights reserved. This work may not be reproduced or redistributed, in whole or in part, without prior written permission from Amazon Web Services, Inc. Commercial copying, lending, or selling is prohibited.













































