

# Building Cloud Optimized GeoTIFFs without Servers

In this workshop we will:
* Learn how to build Cloud Optimized GeoTIFFs.
* Build VRT file using gdalbuildvrt utility
* Use the VRT in QGIS.

Prerequisites
* Create an IAM Role for EC2 and attach policy
* Start EC2 instance with IAM role and SSH to it.
* Create and S3 bucket and upload data to it.

How does it work?
**GeoTIFF** is a metadata standard that allows **georeferencing** information to be embedded within a ﻿TIFF﻿ image file. Cloud Optimized GeoTIFF, or COG, can be thought of a specific formulation of GeoTIFF that optimizes the in-situ use of GeoTIFF files in an Object Store such as Amazon Simple Storage Service (S3). COGs are useful because they allow systems to work with big geo-data files without having to first transfer typically files to local storage before accessing parts of those files. Being able to use data on S3, in-situ, allows many nodes in a cluster to be pointed at the same loosely coupled authoritative data. And even more importantly, any number of different use cases, such as tile servers, ML apps etc can be pointed at the same shared content.

You can see documentation on the GDAL trac site done by Even Rouault in late 2016 regarding COG ﻿here﻿. However, you might want to read Chris Holmes write up ﻿here﻿ first, then go back and look at Even's write-up.

The COG “recipe” does it's magic by organizing  the layout of pixels and image overviews in such a way as to optimize HTTP GET range requests. This enables client  applications to request just part of the image (HTTP range-gets),  rather than requesting the whole file from Amazon S3 and copying to the local file system before making it available to GDAL. Why is this important? It's because the server is often requested to build a small area, such as a 256 x 256 tile from a much larger image typically measured in thousands of pixels. The caveat here is that file system in user space (FUSE) based methods, such as used by NASA GIBS, a method that pre-dates COG, that do copy whole objects to local storage, can be more efficient when many requests are made against the same source GeoTIFFs. However, using COG alone is very simple and probably satisfies most use-cases.

Adding overviews to GeoTIFFs and rebuilding them with internal tiles and jpeg compression is an old technique used to improve performance of mosaic'd orthos served from WMS/WMTS servers. In this workshop we will start with 4-Band imagery stored on Amazon S3, and use AWS Lamba to compute the transformation into RGB COG.

This  workshop will show you how to run the ﻿gdal_translate﻿, ﻿gdaladdo﻿ utilities using the ﻿AWS Lambda﻿ execution environment. It follows the general technique for running arbitrary executables as outlined by Tim Wagner in the AWS Compute Blog ﻿here﻿. You will learn  how to run a distributed batch operation, a single line of which might look like this were it to run from the desktop command line.

```bash
gdal_translate -b 1 -b 2 -b 3 -of GTiff -outsize 50% 50% -co tiled=yes -co BLOCKXSIZE=512 -co BLOCKYSIZE=512' -co PHOTOMETRIC=YCBCR -co COMPRESS=JPEG -co JPEG_QUALITY='85' input.tif output.tif
```

We will do the same, but using ﻿AWS Lambda﻿, in a server-less way, without the limitations imposed by the desktop environment. 
The good news is that the AWS Lamba version of the above invocation of gdal_translate is is actually simpler, because the specifics of the transform, the gdal command line arguments, are kept as Lambda environment variables, rather than used on CLI.

A single command on Lambda looks like the following:

```bash
aws lambda invoke --function-name lambda-gdal_translate --region us-east-1 --invocation-type Event --payload '{"sourceBucket": "aws-naip", "sourceKey": "wi/2015/1m/rgbir/47090/m_4709061_sw_15_1_20150914.tif"}' log
``` 

Note how we are only passing the source geotiff location as a JSON string.

﻿Rather than being limited by the number of cores on your workstation, AWS Lambda﻿ allows you to run your operation on many execution threads.  Lambda makes it easy to access large amounts compute, but serverless compute is only part of  the architecture. This script works in conjunction with ﻿Amazon Simple Storage Service﻿ (S3), rather than a traditional file system or block storage such as EBS or EFS. Both compute and storage is serverless. 

The specific methods described in this blog were used to transform the CONUS collection of USDA NAIP data in the aws-naip, and ESRI curated naip-visualization S3 buckets into COG format. USDA NAIP is a part of the AWS Earth on AWS collection  that you can find described ﻿here﻿.

We will be using 3 Lambda functions that are based on ﻿﻿mwkorver﻿/lambda-gdal_translate.﻿

lambda-gdal_translate-cli
lambda-gdaladdo-evnt
lambda-gdal_translate-evnt.

Note: If you compile your own GDAL binaries, ensure that they’re either statically linked or built for the matching version of Amazon Linux. However, for this workshop binaries are provided.

Create 2 IAM Roles
First a little housekeeping. We will need 2 AWS Identity and Access Management (IAM) Roles. IAM is a web service that helps you securely control access to AWS resources. You use IAM to control who is authenticated (signed in) and authorized (has permissions) to use resources. 
We will need one Role to run AWS Lambda, and one for use with EC2 to invoke Lambda and use S3.

Lambda Role creation
Login to the AWS Management Console and confirm that you are in the Virginia Region (N.Virginia)

Goto IAM console.
﻿https://console.aws.amazon.com/iam/﻿
On the Menu on the left
Click Roles
Click Create Role Button
﻿https://console.aws.amazon.com/iam/home?region=us-east-1#/roles$new?step=type﻿
“AWS Service” should be selected, if not select it then select Lambda.

Under Choose the service that will use this role
Select Lambda
Click the blue button “Next: Permissions”

In the Filter Search box type Lambda.
Select AWSLambdaFullAccess
Click the blue button “Next: Review”

Give your new IAM Role this name. 
lambdaUmgeocon

When it says your new Role has been created click on the Role name to go to properties of that role and  keep that tab open You will need the Role arn that looks something like this.
arn:aws:iam::23098090808080:role/lambdaUmgeocon

EC2 Role creation
For use on EC2 we need a role that allows access to Lambda and S3. An administrative role or power-user role will work, but here is how you create a role that is more specific to what we need today.

Goto IAM console.
﻿https://console.aws.amazon.com/iam/﻿
On the Menu on the left
Click Roles
Click Create Role Button
﻿https://console.aws.amazon.com/iam/home?region=us-east-1#/roles$new?step=type﻿
“AWS Service” should be selected, if not select it.

Under Choose the service that will use this role 
Select EC2
Click the blue button “Next: Permissions”
In the Filter Search box type Lambda.
Select AWSLambdaFullAccess
Click the blue button “Next: Review”
Give your new IAM Role this name. 
ec2Umgeocon

SSH to EC2 instance
Start an EC2 instance in the Virginia region and ssh to it. A small free-tier one can work, but you if you are not interested in waiting, better to pick something with more pep, such as an m5.large ($0.10/hour). 
This workshop assumes familiarity with EC2, but if that is not the case, see ﻿here﻿ for a quick-start tutorial. To use this workshop you will need to be logged into EC2 with the ﻿AWS CLI﻿ installed on that instance along with IAM Role associated with that EC2 instance that gives you access to Lambda and the ability to read/write to S3.

The easiest way to do this is to run Amazon Linux instance configured with an ﻿IAM Role﻿ that can read/write to S3. You can alternatively just run this part locally, but generally speaking, it makes more sense to be closer to your data, on a VM in the same AWS region as the data you are working with. The idea is to go to the data, not the other way around.

From the command line, create a working bucket in the Virginia region.
$ aws s3 mb s3://<yourBucketHere> --region us-east-1

Make sure it works, upload something small to your new S3 bucket. 
```bash
$ aws s3 cp myTestfFile s3://<yourBucketHere>/myTestFile
```
then test with
```bash
$ aws s3 ls s3://<yourBucketHere>/
```

Lets go look at our source data. We are going work with smallest state in the NAIP collection which is Rhode Island.
Run. 
```bash
$ aws s3 ls --request-payer requester --recursive s3://aws-naip
```

Ok, there is a lot of data in this bucket.
Hit ctrl-c to stop it

Narrow it down to just Rhode Island 2014 source RGBIR iles.
```bash
$ aws s3 ls --request-payer requester --recursive s3://aws-naip/ri/2014 | grep rgbir/
```
Write keynames to a list.
```bash
$ sudo aws s3 ls --request-payer requester --recursive s3://aws-naip/ri/2014 | grep rgbir | awk -F" " '{print $4}' > mylist
```

Check your list
$ cat mylist

Creating AWS Lambda functions

Here is a skeleton of the CLI command we are going to use to create a Lambda Function using our sample code.

```bash
aws lambda create-function --region us-east-1
    --function-name yourFunctionName
    --description 'Your Lambda Function' 
    --code S3Bucket=demo-bucket-name,S3Key=zippedCodeFile
    --role role-arn
    --memory-size 640
    --environment Variables="{LD_LIBRARY_PATH=/usr/bin/test/lib64}"
    --handler index.handler
    --runtime nodejs6.10
```
               
Note: The “—role” part of the command. In order to use Lambda with S3, you will need an Identity Access Management (IAM) Role with permissions that allow your Lambda function to read/write to Amazon S3.
This is where you use the IAM Role arn created earlier that looks like this.
arn:aws:iam::23098090808080:role/lambdaUmgeocon

## Create function lambda-gdal_translate-cli
You will need to copy the script below to a text editor and replace both the IAM Role arn and the S3 bucket name to yours.

```bash
aws lambda create-function --region us-east-1 \
--function-name lambda-gdal_translate-cli \
--description 'Runs gdal_translate on invocation from AWS CLI' \
--code S3Bucket=korver.us.east.1,S3Key=lambdaCode/lambda-gdal_translate.zip \
--role arn:aws:iam::<yourAccntNumberHere>:role/lambdaUmgeocon \
--memory-size 960 \
--timeout 120 \
--environment Variables="{gdalArgs='-b 1 -b 2 -b 3 -co tiled=yes -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=ALL_CPUS -co COMPRESS=DEFLATE -co PREDICTOR=2', \
      uploadBucket= '<yourBucketHere>', \
      uploadKeyAcl= 'private', \
      uploadKeyPrefix= 'cloud-optimize/deflate', \
      find01= 'rgbir/', \
      find02= '1m/', \
      replace01= 'rgb/', \
      replace02= '100cm/', \
      largeTiffArgs='-b 1 -b 2 -b 3 -of GTiff -co TILED=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co COMPRESS=JPEG -co JPEG_QUALITY=85 -co PHOTOMETRIC=YCBCR', \
      smallTiffArgs='-b 1 -b 2 -b 3 -co tiled=yes -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=ALL_CPUS -co COMPRESS=DEFLATE -co PREDICTOR=2'}" \
--handler index.handler \
--runtime nodejs6.10 
```


## Create function lambda-gdaladdo-evnt

Copy the script below to a text editor and replace both the IAM Role arn and the S3 bucket name to yours.

```bash
aws lambda create-function --region us-east-1 \
    --function-name lambda-gdaladdo-evnt \
    --description 'Runs gdaladdo to create .ovr file on tif creation event' \
    --code S3Bucket=korver.us.east.1,S3Key=lambdaCode/lambda-gdaladdo-evnt.zip \
    --role arn:aws:iam::<yourAccntNumberHere>:role/lambdaUmgeocon \
    --memory-size 640 \
    --timeout 30 \
    --environment Variables="{targetBucket='<yourBucketHere>', \
      gdaladdoLayers='2 4 8 16 32 64',\
      gdaladdoArgs='-r average -ro'}" \
    --handler index.handler \
    --runtime nodejs6.10 
```
  
## Create function lambda-gdal_translate-evnt
you will need to copy this to a text editor and replace both the IAM Role arn and the S3 bucket name to yours.

```bash
aws lambda create-function --region us-east-1 \
    --function-name lambda-gdal_translate-evnt \
    --description 'Runs gdal_translate on event from S3' \
    --code S3Bucket=korver.us.east.1,S3Key=lambdaCode/lambda-gdal_translate.zip \
    --role arn:aws:iam::<yourAccntNumberHere>:role/lambdaUmgeocon \
    --memory-size 960 \
    --timeout 120 \
    --environment Variables="{gdalArgs='-of GTiff -co TILED=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co COMPRESS=JPEG -co JPEG_QUALITY=85 -co PHOTOMETRIC=YCBCR -co COPY_SRC_OVERVIEWS=YES --config GDAL_TIFF_OVR_BLOCKSIZE 512', \
          uploadBucket= '<yourBucketHere>', \
          uploadKeyAcl= 'authenticated-read', \
          find01= 'cloud-optimize/deflate', \
          replace01= 'cloud-optimize/final'}" \
    --handler index.handler \
    --runtime nodejs6.10 \
		
```

## Add tests events to Lambda functions.
Goto Lambda Console
﻿https://console.aws.amazon.com/lambda/home?region=us-east-1#/functions﻿

Find lambda-gdal_translate-cli
﻿https://console.aws.amazon.com/lambda/home?region=us-east-1#/functions/lambda-gdal_translate-cli﻿

Hit the drop-down by the “Test” button and click “Configure Test Event”.
Add this json text. 

```javascript
{
"sourceBucket": "aws-naip",
"sourceKey": "ct/2014/1m/rgbir/41072/m_4107243_nw_18_1_20140721.tif"
} 
``` 

Give the test json the name smalltiff.
Click Test Button to run the test event.

Once finished you should see
Execution result: succeeded(﻿logs﻿)
Click details that should include this string.
Command to run: AWS_REQUEST_PAYER=requester GDAL_DISABLE_READDIR_ON_OPEN=YES CPL_VSIL_CURL_ALLOWED_EXTENSIONS=.tif ./bin/gdal_translate -b 1 -b 2 -b 3 -co tiled=yes -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=ALL_CPUS -co COMPRESS=DEFLATE -co PREDICTOR=2 /vsis3/aws-naip/ct/2014/1m/rgbir/41072/m_4107243_nw_18_1_20140721.tif /tmp/output.tif 


Go to your EC2 SSH session and list your working bucket.

```bash
$ aws s3 ls --recursive s3://<yourBucketHere>/
```

if all went well you should have just one tif file that looks like this.

cloud-optimize/deflate/ct/2014/100cm/rgb/41072/m_4107243_nw_18_1_20140721.tif

Now lets add eventing to your S3 bucket to wire up the other 2 lambda functions.

Goto lambda-gdaladdo-evnt Function.
﻿https://console.aws.amazon.com/lambda/home?region=us-east-1#/functions/lambda-gdaladdo-evnt﻿

On the left, under 'Designer' scroll down to find S3 and click it. You should get a prompt configure the S3 trigger and area in the GUI call 'Configure Triggers”.
Bucket: <yourBucketHere>
Event type: ObjectCreated
Prefix: cloud-optimize/deflate/
Filter pattern: tif

Similarly for the 3rd and final step, lambda-gdal_translate-evnt function.
﻿https://console.aws.amazon.com/lambda/home?region=us-east-1#/functions/lambda-gdal_translate-evnt﻿

On the left, under 'Designer' scroll down to find S3 and click it. You should get a prompt configure the S3 trigger and area in the GUI call 'Configure Triggers”.
Bucket: <yourBucketHere>
Event type: ObjectCreated
Prefix: cloud-optimize/deflate/
Filter pattern: ovr
Now once again go back to lambda-gdal_translate-cli function
﻿https://console.aws.amazon.com/lambda/home?region=us-east-1#/functions/lambda-gdal_translate-cli﻿
And run the same test again.

Then back to your EC2 SSH session list your working  bucket.

```bash
$ aws s3 ls --recursive s3://<yourBucketHere>/
```

This time output should look like this.
2018-05-23 08:19:00 96271284 cloud-optimize/deflate/ct/2014/100cm/rgb/41072/m_4107243_nw_18_1_20140721.tif
2018-05-23 08:19:16 47455892 cloud-optimize/deflate/ct/2014/100cm/rgb/41072/m_4107243_nw_18_1_20140721.tif.ovr
2018-05-23 08:19:25 12527265 cloud-optimize/final/ct/2014/100cm/rgb/41072/m_4107243_nw_18_1_20140721.tif

The last file under final/  prefix is your completed COG file.


Now let's do a US State worth of data.
```bash
$cat mylist
```

Using the list, this is how you run lambda multiple times while feeding source geotiff locations.

Run this and look at the —payload is json of 2 keypairs, Bucket and key name.
```bash
$ cat mylist | grep 2012/ | awk -F"/" '{print "lambda invoke --function-name lambda-gdal_translate-cli --region us-east-1 --invocation-type Event --payload \x27{\"sourceBucket\":\"aws-naip\",\"sourceKey\":\""$0"\"}\x27 log" }' 
```

Running the above one-by-one would be time consuming for a long list. This is how you run the list, but in parallel.

The -P arg controls the # of threads.

```bash
 $ cat mylist | grep 2012/ | awk -F"/" '{print "lambda invoke --function-name lambda-gdal_translate-cli --region us-east-1 --invocation-type Event --payload \x27{\"sourceBucket\":\"aws-naip\",\"sourceKey\":\""$0"\"}\x27 log" }' | xargs -n11 -P64 aws
 ```

Once you run the above command, after a few moments, you should see many HTTP 202s returned. That means that Lambda is running your gdal jobs in asynchronous or non-blocking mode because you used  --invocation-type Event in your Lambda invocation.

Check to see progress of COG generation by listing your bucket.

```bash
$ aws s3 ls --recursive s3://<yourBucketHere>/
```


## Build VRT file using [gdalbuildvrt](http://www.gdal.org/gdalbuildvrt.html) utility

or [gdaltindex](http://www.gdal.org/gdaltindex.html)


## Use the VRT in QGIS.

















Sample user-data file for EC2

#cloud-boothook
#!/bin/bash
set -x

exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
# fix for "unable to resolve host" error even when dns hostnames/resolution are turned on for VPC

echo "127.0.0.1 $(hostname)" >> /etc/hosts

# grab shapefiles from public bucket
aws s3 sync s3://umgeocon/shpfl /home/ec2-user/mapfiles
# grab map file, this file includes keys
aws s3 cp s3://umgeocon/mapfiles/test.map /home/ec2-user/mapfiles/test.map

if [ -x "$(command -v docker)" ]
then
echo "This is a reboot"
# grab map and shapefiles

else
echo "Running first time install scripts."
yum update -y
DEBIAN_FRONTEND=noninteractive
# install docker
yum install -y docker
service docker start
docker run --detach -v /home/ec2-user/mapfiles:/mapfiles:ro --publish 8080:80 --name mapserver geodata/mapserver
# log file setup in the now running mapserver container
# the location of the log file is dependent on what you specify in your map file.
# looks like - CONFIG "MS_ERRORFILE" "/var/log/ms_error.log" in map file
docker exec mapserver touch /var/log/ms_error.log
docker exec mapserver chown www-data /var/log/ms_error.log
docker exec mapserver chmod 644 /var/log/ms_error.log
fi



test query
﻿http://ec2-184-72-85-184.compute-1.amazonaws.com:8080/wms/?map=/mapfiles/test.map&SERVICE=WMS&LAYERS=naip-index-20170815&SRS=epsg:3857&BBOX=-9798309.78182,5438953.18466,-9798004.03371,5439258.93277&STYLES=&VERSION=1.1.1&REQUEST=GetMap&FORMAT=image/jpeg&WIDTH=256&HEIGHT=256﻿


$ sudo docker run -v $(pwd):/data geodata/gdal gdalbuildvrt







