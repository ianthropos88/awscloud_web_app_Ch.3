# **Bespoke AWS Cross-Region CI/CD Pipeline (CloudFormation)** :pushpin: #

AWS CodePipeline includes a number of actions that help you configure build, test, and deploy resources for your automated release process. You can add actions to your pipeline that are in an AWS Region different from your pipeline. When an AWS service is the provider for an action, and this action type/provider type are in a different AWS Region from your pipeline, this is a cross-Region action.

**Note:** Cross-region actions are supported and can only be created in those AWS Regions where CodePipeline is supported. For a list of the supported AWS Regions for CodePipeline, see Quotas in AWS CodePipeline.

You can use the console, AWS CLI, or AWS CloudFormation to add cross-Region actions in pipelines.

**Note:** Certain action types in CodePipeline may only be available in certain AWS Regions. Also note that there may be AWS Regions where an action type is available, but a specific AWS provider for that action type is not available.

When you create or edit a pipeline, you must have an artifact bucket in the pipeline Region and then you must have one artifact bucket per Region where you plan to execute an action. For more information about the ArtifactStores parameter, see CodePipeline pipeline structure reference.

**Note:** CodePipeline handles the copying of artifacts from one AWS Region to the other Regions when performing cross-region actions.

If you use the console to create a pipeline or cross-Region actions, default artifact buckets are configured by CodePipeline in the Regions where you have actions. When you use the AWS CLI, AWS CloudFormation, or an SDK to create a pipeline or cross-Region actions, you provide the artifact bucket for each Region where you have actions.

**Note:** You must create the artifact bucket and encryption key in the same AWS Region as the cross-Region action and in the same account as your pipeline.

You cannot create cross-Region actions for the following action types:

- Source actions

- Third-party actions

- Custom actions

When a pipeline includes a cross-Region action as part of a stage, CodePipeline replicates only the input artifacts of the cross-Region action from the pipeline Region to the action's Region.

**Note:** The pipeline Region and the Region where your CloudWatch Events change detection resources are maintained remain the same. The Region where your pipeline is hosted does not change.

## > :rocket: **Thank you for your interest in my work.** :blush: ##

This solution aims at deploying a web application with a PostgreSQL database on AWS in 3 environments (Development, Staging, & Production).

The project is supported by several managed services including AWS RDS, PostgreSQL,Amazon ElastiCache, Amazon Elasticsearch Service, Amazon Pinpoint and Amazon Personalize.

# **Cross-Region Action in CodePipeline** :hourglass_flowing_sand::clock10: #

#### **PROBLEM** ####

Cross-region deployment is natively supported by AWS CodePipeline. At the build stage, builds can be processed by already created CodeBuild projects in supported regions. Native Cross-region deployment with deploy stage providers like CloudFormation, CodeDeploy, and ECS is possible when operating within supported regions.

However, some Regions do not have native support for cross-region deployment. Below explanation provides 2 ways to implement a bespoke AWS CI/CD pipeline in such regions. In regions where AWS natively supports cross-region or multi-region deployments, the default integrated solution should be used to achieve a cleaner solution.

#### **SOLUTION** ####

When there is no full support for builtin code connection in a region, a pipeline is set up in an AWS region which already supports this feature.

This pipeline will get triggered with set actions (merge/push) on a specified branch, then pushes the source code to CodeBuild. CodeBuild zips the source code, copies it to an S3 bucket. With Cross-region replication set, the zipped source code is replicated to our desired region.

The CodePipeline in the 1st region region as per the architecture above, has its source provider set to S3. The next problem arises, how can we trigger CodePipeline when there is a new object in the S3 bucket set as its source provider?

This article approaches this problem using CloudWatch events to trigger CodePipeline. Of course, we’ll have to disable the PollForSourceChanges property, since we want to trigger the pipeline and not wait for it to poll for changes.

For CodePipeline resoucrce created with AWS CloudFormation, theConfiguration property in the source stage called PollForSourceChanges should be set to false. If your template doesn't include that property, then PollForSourceChanges is set to true by default.

```yaml
CodePipelineTrigger:
    Type: 'AWS::S3::Bucket'
    Properties:
      VersioningConfiguration:
        Status: Enabled

CodePipeline:
    Type: 'AWS::CodePipeline::Pipeline'
    Properties:
      Name: !Sub 'codepipeline-${AWS::StackName}'
      RoleArn: !GetAtt CodePipelineServiceRole.Role.Arn
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Version: '1'
                Owner: AWS
                Category: Source
                Provider: S3
              Configuration:
                S3Bucket: !Ref CodePipelineTrigger
                S3ObjectKey: ! Ref PipelineSourceObjectKey
                PollForSourceChanges: 'false'
              OutputArtifacts:
                - Name: SourceCode
```

**The Amazon S3 source event-based change detection using Cloudwatch Events.**

The following CloudFormation resources are required to implement CloudWatch Events, CodePipeline trigger:

1. CloudWatch IAM role (with CodePipeline StartPipelineExecution policy attached)

```yaml
AmazonCloudWatchEventRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Principal:
              Service:
                - events.amazonaws.com
            Action: sts:AssumeRole
      Path: /
      Policies:
        -
          PolicyName: cwe-pipeline-execution
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              -
                Effect: Allow
                Action: codepipeline:StartPipelineExecution
                Resource: "*"
```

2. CloudWatch Event Rule

```yaml
AmazonCloudWatchEventRole:
    Type: AWS::Events::Role
    Properties:
      EventPattern:
        source:
          - aws.s3
        detail-type:
          - 'AWS API Call via CloudTrail'
        detail:
          eventSource:
            - s3.amazonwas.com
          eventName:
            - PutObject
            - CompleteMultipartUpload
          resources:
            ARN:
              - !Join [ '', [ !GetAtt CodePipelineTrigger.Arn, '/', !Ref PipelineSourceObjectKey ] ]
      Targets:
        -
          Arn:
            !Join [ '', [ 'arn:aws:codepipeline:', !Ref 'AWS::Region', ':', !Ref 'AWS::AccountId', ':', !Ref CodePipeline ] ]
          RoleArn: !GetAtt AmazonCloudWatchEventRole.Arn
          Id: codepipeline-Rule
```

3. CloudTrail trail, S3Bucket (to store CloudTrail event log files) and Bucket policy (Amazon S3 uses to log the events that occur)

```yaml
#CloudTrail Bucket Policy
  AWSCloudTrailBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref AWSCloudTrailBucket
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Sid: AWSCloudTrailAclCheck
            Effect: Allow
            Principal:
              Service:
                - cloudtrail.amazonaws.com
            Action: s3.GetBucketAcl
            Resource: !GetAtt AWSCloudTrailBucket.Arn
          -
            Sid: AWSCloudTrailWrite
            Effect: Allow
            Principal:
              Service:
                - cloudtrail.amazonaws.com
            Action: s3:PutObject
            Resource: !Join [ '', [ !GetAtt AWSCloudTrailBucket.Arn, '/AWSLogs/', !Ref 'AWS::AccountId', '/*' ] ]
            Condition:
              StringEquals:
                s3:x-amz-acl: bucket-owner-full-control

#CloudTrail log Bucket
  AWSCloudTrailBucket:
    Type: AWS::S3::Bucket

#CloudTrail
  AWSCloudTrail:
    DependsOn:
      - AWSCloudTrailBucketPolicy
    Type: AWS::CloudTrail::Trail
    Properties:
      S3BucketName: !Ref AWSCloudTrailBucket
      EventSelectors:
        -
          DataResources:
            -
              Type: AWS::S3::Object
              Values:
                - !Join [ '', [ !GetAtt CodePipelineTrigger.Arn, '/', !Ref PipelineSourceObjectKey ] ]
          ReadWriteType: WriteOnly
      IncludeGlobalServiceEvents: true
      IsLogging: true
      IsMultiRegionTrail: true
```
When these CloudFormation resources are pieced together, the seond pipeline is triggered when files matching the set object key, are created or updated to the S3 source bucket (this also includes objects created through cross-region replication).
