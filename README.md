
# 博客

英文： https://aws.amazon.com/cn/blogs/architecture/field-notes-building-an-automated-image-processing-and-model-training-pipeline-for-autonomous-driving/  

中文： 准备中。



# 1.环境配置
----------
1.Cloud9 权限 
- 绑定角色（这个必须操作，cdk不能使用aksk的方式）  
- 清理临时（如果没有清除，cdk将会执行失败）  
```
rm -vf ${HOME}/.aws/credentials
```

2.Set region
```
aws configure set region $(curl -s http://169.254.169.254/latest/meta-data/placement/region)
```

3.调大磁盘空间
```
wget http://container.bwcx.me/0-prepare/002-mgmt.files/resize-ebs.sh

chmod +x resize-ebs.sh

./resize-ebs.sh 2000
```

查看磁盘信息
```
df -hT

lsblk
```

使扩容生效
```
sudo growpart /dev/nvme0n1 1

sudo xfs_growfs -d /
```


# 2.部署步骤
----------

## 2.1 准备代码
```
git clone https://github.com/auto-bwcx-me/aws-autonomous-driving-data-lake-image-extraction-pipeline-from-ros-bagfiles.git

cd aws-autonomous-driving-data-lake-image-extraction-pipeline-from-ros-bagfiles
```


## 2.2 设置区域
----------

在开始之前，需要设定 Region，如果没有设定的话，默认使用新加坡区域 （ap-southeast-1）

```
# default setting singapore region (ap-southeast-1)
sh 00-define-region.sh

# sh 00-define-region.sh us-east-1
```



## 2.3 准备环境

```
pip install --upgrade pip
# python3 -m pip install --upgrade pip

python3 -m venv .env

pip3 install -r requirements.txt
```


## 2.4 安装CDK
```
npm install -g aws-cdk --force

cdk --version
```

如果是第一次运行CDK，先执行
参考文档 https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html
```
# cdk bootstrap aws://123456789012/us-east-1
# cdk bootstrap aws://123456789012/ap-southeast-1

cdk bootstrap
```

创建 ECR 存储库： `vsi-rosbag-image-repository`
```
aws ecr create-repository --repository-name vsi-rosbag-image-repository
```



## 2.5 CDK部署
```
bash deploy.sh deploy true
```



# 3.注意事项
----------
留空。




# 4.ROS bag image extraction pipeline and Model Training
----------
This solution describes a workflow that processes ROS bag files on Amazon S3, extracts the PNG files from a video stream using AWS Fargate on Amazon Elastic Container Services. The solution builds a DynamoDB table containing all detection results from Amazon Rekognition, which can be queried to find images of interest such as images containing cars. Afterwards, we want to label these images and fine-tune a Object Detection Model to detect cars on the road. For simplicity reasons, we have provided an example SageMaker Ground Truth Manifest File from a Bounding Boxes Labeling Job. In order to train the Object Detection Model we will convert the SageMaker Ground Truth Manifest file into the RecordIO file format, after we have visually inspected the annotation quality from our labelling job.

## Initial Configuration and Deployment of the CDK Stack

Note that deploying this stack may incur charges on your AWS account. See the section 'Cleaning Up' for instructions on 
how to remove the stack when you have finished with it.

    Define 3 names for your infrastructure in config.json:
    
    {
          "ecr-repository-name": "my-ecr-repository",
          "image-name": "my-image",
          "stack-id": "my-stack"
    }
   
   Ypu will need to ensure that you have also created an ECR repository matching the name used above (in this case my-ecr-rpository)
   
   Optionally (leave these as they are unless you know you need to change them), define other parameters for your Docker 
   container, such as number of vCPUs and RAM it should consume, in config.json:
    
          "cpu": 4096,
          "memory-limit-mib": 12288,
          "timeout-minutes": 2
          "environment-variables": {}
   
   [Fargate CPU and Memory Limit Documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)
     
   In deploy.sh, set REGION to teh region you are using for the deployment. The REPO_NAME and IMAGE_NAME should match 
   the values in your config.json:

    '''   
    REPO_NAME=vsi-rosbag-image-repository # Should match the ecr repository name given in config.json
    IMAGE_NAME=my-vsi-ros-image          # Should match the image name given in config.json
    REGION=eu-west-1
    '''
   
   

Extending the code to meet your use case:
    Extend the ./service/app/engine.py file to add more complex transformation logic
    
    Customizing Input
        Add prefix and suffix filters for the S3 notifications in config.json
        
    


deploy.sh with build=true will create an ecr repository in your account, if it does not yet exist, and push your docker image to that repository
Then it will execute the CDK command to deploy all infrastructure defined in app.py and ecs_stack.py 
          
          
The `cdk.json` file tells the CDK Toolkit how to execute your app.

This project is set up like a standard Python project.  The initialization
process also creates a virtualenv within this project, stored under the .env
directory.  To create the virtualenv it assumes that there is a `python3`
(or `python` for Windows) executable in your path with access to the `venv`
package. If for any reason the automatic creation of the virtualenv fails,
you can create the virtualenv manually.

To manually create a virtualenv on MacOS and Linux:

```
$ python3 -m venv .env
```

After the init process completes and the virtualenv is created, you can use the following
step to test deployment

```
$ bash deploy.sh <cdk-command> <build?>

$ bash deploy.sh deploy true
```


To add additional dependencies, for example other CDK libraries, just add
them to your `requirements.txt` or `setup.py` file and rerun the `pip install -r requirements.txt`
command.

## Fine-tuning of the Machine Learning Model

Once you have launched the stack explained above, you can clone the package `object-detection` from this repository into
the SageMaker Notebook Instance named `ros-bag-demo-notebook` that has been created in your account. Once you have 
cloned it, the next steps are outlined in the Notebook `Transfer-Learning.ipynb`.

## Cleaning up

To remove the resources from your account you can run:
'''
$ cdk destroy
'''

note that the S3 buckets will not be deleted unless you empty them first.

## Useful CDK commands

 * `bash deploy.sh default ls false eu-west-1`          list all stacks in the app
 * `bash deploy.sh default synth false eu-west-1`       emits the synthesized CloudFormation template
 * `bash deploy.sh default deploy true eu-west-1`      build and deploy this stack to your default AWS account/region
 * `bash deploy.sh default diff false eu-west-1`        compare deployed stack with current state
 * `bash deploy.sh default docs false eu-west-1`        open CDK documentation

