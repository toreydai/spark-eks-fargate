# Run Spark Job on EKS Fargate
### 1.	准备终端环境
1) 创建并配置1台Cloud9实例，作为操作终端；
2) 打包容器镜像时需要更多的EBS容量，在AWS EC2控制台将Cloud9实例的EBS卷扩展为50GB，并使用如下命令扩展其文件系统：

```
sudo growpart /dev/nvme0n1 1
sudo xfs_growfs -d /
df -h
```
### 2.	测试数据上传到S3
```
#创建S3存储桶
aws s3 mb s3://spark-eks-test-001

#在S3存储桶下创建文件夹
aws s3api put-object --bucket spark-eks-test-001 --key bin/
aws s3api put-object --bucket spark-eks-test-001 --key output/

#将测试数据和jar包上传到S3
aws s3 cp green_tripdata_2018-1.csv s3://spark-eks-test-001/
aws s3 cp yellow_tripdata_2018-1.csv s3://spark-eks-test-001/
aws s3 cp taxi_zone_lookup.csv s3://spark-eks-test-001/
aws s3 cp spark-eks-assembly-3.1.2.jar s3://spark-eks-test-001/bin/
```
### 3.	打包镜像
执行如下命令，打包Spark容器镜像并推送到ECR镜像仓库：

```
#设置AWS默认区域
aws configure set default.region ap-northeast-2

#登陆到ECR
aws ecr-public get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin public.ecr.aws

#在ECR中创建Spark容器镜像使用的镜像仓库
SPARK_REPO=$(aws ecr-public create-repository \
  --repository-name spark-eks \
  --region us-east-1 \
  --query 'repository.repositoryUri' \
  --output text)

#构建docker镜像
docker build -t spark-eks:v3.1.2 .
#对docker镜像打tag
docker tag spark-eks:v3.1.2 $SPARK_REPO:v3.1.2
#将本地的镜像推送到ECR中
docker push $SPARK_REPO:v3.1.2
```

### 4.	创建EKS集群
参照如下步骤，创建EKS集群：

```
#集群配置文件创建EKS集群
eksctl create -f eksctl_cluster.yaml

#创建K8S自动弹性伸缩组件cluster autoscaler
kubectl apply -f cluster_autoscaler.yaml

#创建Spark运行时使用的serviceaccount
S3_POLICY_ARN=arn:aws:iam::aws:policy/AmazonS3FullAccess

eksctl create iamserviceaccount \
  --name spark \
  --namespace spark \
  --cluster spark-eks-fargate \
  --attach-policy-arn $S3_POLICY_ARN \
  --approve --override-existing-serviceaccounts

eksctl create iamserviceaccount \
  --name spark-fargate \
  --namespace spark-fargate \
  --cluster spark-eks-fargate \
  --attach-policy-arn $S3_POLICY_ARN \
  --approve --override-existing-serviceaccounts
```

### 5.	运行Spark作业
对spark-job-fargate.yaml需要进行修改后方能使用：
<br>1) 修改容器镜像地址为当前账号中ECR的镜像仓库地址

```
image: public.ecr.aws/b0b2i9t9/spark-eks:v3.1.2
--conf spark.kubernetes.container.image=public.ecr.aws/b0b2i9t9/spark-eks:v3.1.2 \
```
<br>2) 将s3存储桶路径spark-eks-test-001修改为当前使用账号中的实际路径

```
            --conf spark.hadoop.fs.s3a.committer.name=magic \
            --conf spark.hadoop.fs.s3a.committer.magic.enabled=true \
            --conf spark.hadoop.fs.s3a.fast.upload=true \
            \"s3a://spark-eks-test-001/bin/spark-eks-assembly-3.1.2.jar\" \
            \"s3a://spark-eks-test-001/\" \
            \"2018\" \
            \"s3a://spark-eks-test-001/taxi_zone_lookup.csv\" \
            \"s3a://spark-eks-test-001/output/\"
            \"spark_eks_fargate\""
```
            
接下来即可在EKS Fargate中运行Spark任务。

```
#执行如下命令，启动运行在EKS Fargate上的Spark任务
kubectl apply -f spark-job-fargate.yaml

#查看Spark Pod运行情况
kubectl get pods -n spark-fargate

#查看Spark任务日志
kubectl logs spark-eks-fargate-c8ad8281154d6352-driver -n spark-fargate
```
任务完成后，进入S3存储桶output路径下，即可看到输出文件。
