---
layout: post
title: Hassle-Free Python Lambda Deployment
categories: [AWS, AWS_Lambda, Python, Shell_Script, Backend, Serverless]
---

#### Tutorial + Script

If you are new to AWS Lambda, you will probably find a pain in the a** to update your Lambda code when you are working with deployment packages.

<!--more-->


The usual steps are: change the code in your development environment, create the lambda package (with all dependencies in it), compress it, upload to S3, and finally update your Lambda function. Yeah, a pain in a**, as I said.

If you are doing this manually and making frequent changes (specially during development), it's time consuming and very error-prone. That's why I will provide a **shell script** at the end of the post so you will be able to **automate** the steps above, but I strongly recommend you understand how this stuff works, so you can use it properly, hence the **tutorial**.

## Before You Begin

This tutorial uses Python as Lambda runtime, and you will need to have installed on your system:

- Python2.7
- virtualenv
- pip
- AWS CLI


On the AWS side, you will need to have created:

- An s3 bucket (to store the deployment package)
- An IAM user (with programmatic access)
- The Lambda function

It's assumed you are familiar with the basics of AWS Lambda, S3 and IAM. This tutorial won't cover that part. If you are begginer and need help with that, you can Google or check out this [great introductory book](http://amzn.to/2hrotRP)

## Step 1: Set Up the (Virtual) Environment

First, we will create our environment where we will have the Python interpreter and its dependencies. Navigate to your Lambda project root, and create a python2.7 environment called env.

```
virtualenv -p /usr/bin/python2.7 env
```

[Since April 2017, Lambda also supports Python 3.6](https://aws.amazon.com/about-aws/whats-new/2017/04/aws-lambda-supports-python-3-6/). Finally, AWS! 🎉🎉 You should be able to use this tutorial as well for 3.6 with minor or no changes, even tough I didn't test it.

Create a file called **requirements.txt** and add all your pip dependencies, one per line. Then activate the virtual env. and install the requirements by running:

```
source env/bin/activate
pip install -r requirements.txt
```

**Note**: Lambda already includes the AWS SDK for Python (Boto 3), so you don’t need to include it in your deployment package, hence neither in your requirements files, unless you need a different version.

## Step 2: Source and Entry Point

In your project root, create a directory to hold your actual Python source code, let’s name it src. You will write code here as if you were building a simple Python module. There’s just one important thing, your function’s entry point.


In the Lambda configuration console this is called the handler, and it defaults to `lambda_function.lambda_handler`, which means that Lambda expects a file in your deployment package called `lambda_function.py` and a Python function with the following signature:

{% highlight python %}
def lambda_handler(event, context):
    # your code here
{% endhighlight %}

Your are free to choose any file or function name you like, as long as your configure accordingly in the Lambda console. Since our code is under src, we will have to define the handler as:

```
src.lambda_function.lambda_handler
```

If this is misconfigured, you might get errors like:

```
Unable to import module 'src.lambda_function'
```

## Step 3: Native Libs

As you may know, Lambda functions runs on pre-configured [Amazon Linux machines](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html). They ship with very few native libraries, so if you need additional ones on your execution environment (such as Numpy, OpenCV, MySQL, lxml, Pillow etc), you should pre-compile them and then copy to your deployment package.

Creating those platform-specific libraries is beyond the scope of this post. Also there are some nice people out there that may already have done that for the libraries you need, [like this one](https://github.com/Miserlou/lambda-packages). Don't reinvent the wheel, google first to find out if the one you are looking for has been already compiled by a trusted source.

For the purpose of this tutorial, create a subdirectory called `native_libs` in your lambda root and place all your pre-compiled libraries in there.

## Step 4: Configure AWS CLI

If you don't have already it installed, [please do it now](https://aws.amazon.com/cli/). This is key to save you tons of time while updating your Lambda function, as you will see later.

If you are like me, that work on different projects, for different clients, on the same development machine, you will find handy to set up AWS CLI with multiple profiles. Assuming you already have an IAM user you control (and created it with programatic access, so we can use with AWS CLI) this is how you will configure it:


{% highlight shell %}
$ aws configure --profile my_profile_name
AWS Access Key ID [None]: AKIAI44QH8DHB-EXAMPLEID
AWS Secret Access Key [None]: je7MtGbClwBF2Zp9Uh3yCo8nvb-EXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: 
{% endhighlight %}

## Step 5: Attach IAM Policy

Here, we need to make sure the IAM user configured in the previous step has enough privileges to (i) upload files to a certain S3 bucket, where we will temporarily store our deployment package; (ii) update our lambda function code with the package stored on S3.

For (i), assuming the name of the S3 bucket that will hold our deployment package is *awesome-lambda-code*, then you should attach the following custom policy to the user.

{% highlight json %}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1494441459000",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::awesome-lambda-code/*"
            ]
        }
    ]
}
{% endhighlight %}

To create a custom policy in the IAM Console go to *policies > create policy > create your own policy*. Then don't forget to attach the policy (under *Policy actions*) to the right user.

**Note**: Mind the trailing '/*', otherwise you will be facing the error:

```
A client error (AccessDenied) occurred when calling the CreateMultipartUpload operation: Access Denied
```

As for (ii), assuming the name of your lambda function is awesome-lambda-function, then this is what your policy should look like:

{% highlight json %}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1494441719000",
            "Effect": "Allow",
            "Action": [
                "lambda:PublishVersion",
                "lambda:UpdateFunctionCode"
            ],
            "Resource": [
                "arn:aws:lambda:us-west-2:575808965339:function:awesome-lambda-function"
            ]
        }
    ]
}
{% endhighlight %}

## Step 6: Create a Distribution Folder

In your lambda project folder, create a `dist` directory, where we will copy all necessary files that will make up our distribution package. You might ask, "but the content of the project root isn't everything we need ? " Yes, but it's not organized in the way Lambda requires.

Let’s take a look at our project structure:


![Project Template Structure](/images/hfld-project-structure.png){: .center-image }

First off, we will copy the src folder and any other file or directory that your code uses, including configuration files, data files, and so on. Just copy as is to `dist`:

```
cp -rf src conf data dist
```

Now here it comes the trick most people miss. Our virtual environment shall go into the root of dist as well, but not under any intermediate folder, so you should copy like this:

```
cp -rf env/lib/python2.7/site-packages/* dist
```

Notice that we don't copy the env directory, but only the content of site-packages.

For your pre-compiled native libs, we will proceed similarly. If we assume that all binaries are under native_libs, then copy like this:

```
cp -rf native_libs/* dist
```

## Step 7: Zip Zip

Time to create our deployment package, a zip file. You might say, "Dude, are you really going to teach me how to zip a folder ?". This might seems quite easy, but there's a big *gotcha* that a lot of people fall for. **You should zip your file so that when Lambda extracts it, it will have no intermediate folder, that is, the result is the content of your root dist folder.**


Therefore if you zip the dist folder directly, the unzip result will be a folder called dist. We don't want that. We want the **content** of dist to be the result of the decompression. Here's how you will do:

```
cd path/to/dist
zip -r path/to/deployment_bundle.zip .
```

If you fail to properly set up Steps 6 or 7, you might get runtime errors like:

```
Unable to import module XXX : No module named YYY
```


## Step 8: Deploy Time!

Now we are ready to upload our package to S3 and update our lambda function code. These are accomplished with:

```
aws s3 cp path/to/deployment_bundle.zip s3://awesome-lambda-code --profile my_profile_name

aws lambda update-function-code --function-name awesome-lambda-function --s3-bucket awesome-lambda-code --s3-key deployment_bundle.zip --publish --profile my_profile_name

```


**Note**: There's a possibility to update the Lambda code via direct upload, instead of going through S3, but I found in past that the direct upload to Lambda to be unreliable. That's why I prefer to perform the update via s3.


## Script and Project Template


Ok, if you understand what it has been done in the previous steps, you will have an easy time customizing the script and organize your project structure, and you will be doing quick and hassle-free Lambda deployments.


You can get the script [HERE](https://github.com/joarleymoraes/aws_lambda_deploy). Feel free to contribute. :)


Best way to use it is to start writing your code from the project_template provided.
Leave a comment below if you run into issues.







