# Blue Green CD
Sample for who love「Blue Green CD」,「infras-as-code」and「immutable-infrastructure」.

* Please get started with following one first for going smoothly:
    * [Build Golden Image with packer and ansible](https://github.com/caophamtruongson/packer-ansible-cloudformation)

* Please read first if you don't know what is Blue Green CD
    * https://www.quora.com/What-is-blue-green-deployment
    * https://martinfowler.com/bliki/BlueGreenDeployment.html

* In my work, the task will contain folllowing items
    * AWS::AutoScaling::LaunchConfiguration
    * AWS::ElasticLoadBalancing::LoadBalancer
    * AWS::AutoScaling::AutoScalingGroup
    * In future
        * Including: SQS queue
        * Including: SecurityGroup
        * Blue Green switching by「weigth」
        * Do all thing with CI

# Index
1. [AWS console preparation](#aws-console-preparation)
1. [Credentials preparation](#credentials-preparation)
1. [Running](#running)
    * [Blue version](#blue-version)
    * [Green version](#green-version)
    * [Route53 Preparation](#route53-preparation)
    * [Switching DNS, point to「green」version](#switching-dns-point-togreenversion)
1. [Furthermore](#furthermore)
1. [Best practices](#best-practices)
1. [License](#license)


# AWS console preparation
1. Key pairs, please ignore if you finished at part: [Build Golden Image with packer and ansible](https://github.com/caophamtruongson/packer-ansible-cloudformation)
    * https://github.com/caophamtruongson/packer-ansible-cloudformation#aws-console-preparation
1. Security group, please ignore if you finished at part: [Build Golden Image with packer and ansible](https://github.com/caophamtruongson/packer-ansible-cloudformation)
    * https://github.com/caophamtruongson/packer-ansible-cloudformation#aws-console-preparation

# Credentials preparation
1. Create/modify「.bash_aws」for exporting environment variables
    * vi ~/.bash_aws
        * Please copy the content from this url first
            * https://github.com/caophamtruongson/packer-ansible-cloudformation#credentials-preparation
        * And then the new additional for this sample
            ```
            export AWS_SUBNET="PLACE_YOUR_DEFAULT_AWS_SUBNET_HERE" # e.g: "subnet-xyz123abc"
            export AWS_MIN_SIZE_ASG=3 # Three instances will be created as min value
            export AWS_MAX_SIZE_ASG=5 # Five instances will be created as min value
            ```
1. Exporting environment variables by following command
    * `. ~/.bash_aws`

# Running

## Blue version
1. Change directory to clone folder
    * `cd blue-green-continuous-delivery`
1. Export Golden Image value
    * Exporting value for「AWS_DATETIME_PARAMETER」variable
        ```
        export AWS_DATETIME_PARAMETER=`date +'%Y%m%d%H%M%S'`
        ```
    * `export AWS_EC2_GOLDEN_IMAGE=PLEASE_place_your_AMI_ID_here`
        * Because Golden Image AMI is usually changed, I'd better export it here instead of setting at「.bash_aws」
        * If you don't have any Golden Image, please refer here how to create
            * https://github.com/caophamtruongson/packer-ansible-cloudformation
1. Create new CloudFormation stack
    * `aws cloudformation create-stack --stack-name DemoBlueGreenCD-$AWS_DATETIME_PARAMETER --template-body file:///$PWD/cloudformation.stack.template --parameters ParameterKey=ImageIdParameter,ParameterValue=$AWS_EC2_GOLDEN_IMAGE ParameterKey=InstanceTypeParameter,ParameterValue=$AWS_EC2_INSTANCE_TYPE ParameterKey=KeyNameParameter,ParameterValue=$AWS_EC2_KEY_PAIR ParameterKey=SecurityGroupParameter,ParameterValue=$AWS_EC2_SECURITY_GROUP ParameterKey=NameTagParameter,ParameterValue=$AWS_EC2_NAME_TAG ParameterKey=SubnetsParameter,ParameterValue=$AWS_SUBNET ParameterKey=MinSizeOfASGParameter,ParameterValue=$AWS_MIN_SIZE_ASG ParameterKey=MaxSizeOfASGParameter,ParameterValue=$AWS_MAX_SIZE_ASG ParameterKey=DateTimeParameter,ParameterValue=$AWS_DATETIME_PARAMETER`
        * Output successfully
            ```
            {
                "StackId": "arn:aws:cloudformation:......"
            }
            ```
1. Copy「DNSNameInfo」of the stack after the stack was created successfully.
    * You can find「DNSNameInfo」at「Outputs」part of your stack detail
        * `AWS Console` → `Services` → `CloudFormation` → Click your stack name (to go to Detail page) → Scroll down to see `Outputs` word.
1. Create new「CNAME」record set for your「Hosted Zones」
    * Go to https://console.aws.amazon.com/route53/home, and create new record set
        * e.g: www.my-hostest-domain.com
            * And set value same as「DNSNameInfo」property of the created stack
1. You can connect to ELB by DNS name rigth now, it should be that
1. Phew,「blue」version has done. Let's move on creating「green」version


## Green version


1. Exporting new value for AWS_DATETIME_PARAMETER
    ```
    export AWS_DATETIME_PARAMETER=\date +'%Y%m%d%H%M%S'``
    ```
1. Creating new stack again (the command is same above one)
    ```
    aws cloudformation create-stack --stack-name DemoBlueGreenCD-$AWS_DATETIME_PARAMETER --template-body file:///$PWD/cloudformation.stack.template --parameters .....same a bove.....
    ```

## Route53 Preparation

You are going to update your above DNS record set, so that you need to prepare config file to update new「DNSNameInfo」value for that record set. I prepared config file for you:「change-resource-record-sets.json」

1. Replace your record set name
    * `sed -i -e "s/########_NAME_OF_RECORD_SET_########/your_record_set_name_here/" change-resource-record-sets.json`
        * e.g
            * `sed -i -e "s/########_NAME_OF_RECORD_SET_########/www.my-hostest-domain.com/" change-resource-record-sets.json`
1. Replace your DNSNameInfo
    * `sed -i -e "s/########_NAME_OF_RECORD_SET_########/your_new_DNSNameInfo/" change-resource-record-sets.json`
        * e.g
            * `sed -i -e "s/########_NEW_DNS_NAME_INFO_HERE_########/ELB-abcxyz-........elb.amazonaws.com/" change-resource-record-sets.json`

## Switching DNS, point to「green」version

1. Update DNS record set
    * `aws route53 change-resource-record-sets --hosted-zone-id YOUR_HOSTED_ZONE_ID --change-batch file:///$PWD/change-resource-record-sets.json`
        * To get your YOUR_HOSTED_ZONE_ID
            * `aws route53 list-hosted-zones | grep 'hostedzone' | awk '{print $2}'`
                * Reference more about「awk」
                    * `man awk`
                    * or: https://archive.org/details/pdfy-MgN0H1joIoDVoIC7
    * For me:
        * `aws route53 change-resource-record-sets --hosted-zone-id "/hostedzone/XYZ12345XYZ" --change-batch file:///$PWD/change-resource-record-sets.json`
            * Succesful output message
                ```
                {
                    "ChangeInfo": {
                        "Status": "PENDING",
                        "Comment": "Demo Blue Green CD",
                        "SubmittedAt": "2017-10-19T16:21:23.953Z",
                        "Id": "/change/XXXXXXXXXXXXXXXXX"
                    }
                }
                ```
1. Enjoy!

# Furthermore
1. AWS Resource Types Reference
    * http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html
1. CloudFormation Document
    * http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-as-group.html
    * http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/quickref-autoscaling.html
    * http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/example-templates-autoscaling.html
1. AWS CLI Route53
    * http://docs.aws.amazon.com/cli/latest/reference/route53/index.html#cli-aws-route53
1. Reference topics
    * https://www.thoughtworks.com/insights/blog/implementing-blue-green-deployments-aws
    * https://naftuli.wtf/2017/02/01/route53-bg-deployments/
    * https://minops.com/blog/2015/02/the-dos-and-donts-of-bluegreen-deployment/

# Best practices
1. Remember outputing DNSName of ELB
1. Remember set Name tag for EC2 instances
1. Do update stack after stack was created
    * `aws cloudformation update-stack --stack-name DemoBlueGreenCD-$AWS_DATETIME_PARAMETER --template-body file:///$PWD/cloudformation.stack.template --parameters SAME_AS_PARAMETER_WHEN_CREATING`

# License

[The MIT License](https://opensource.org/licenses/MIT)

Copyright 2017 Sơn Cao

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
