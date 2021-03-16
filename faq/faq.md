# FAQ

## UNANSWERED QUESTIONS - please help to anwser!

Q: How do I use `crontab` on QHub to regularly execute notebooks?


## ANSWERED QUESTIONS:
## Q: How do I retrieve my user data from the EFS Share before I destroy my qhub on AWS?
Answered by: @jkellndorfer

We want to backup our user data from the EFS persistent volume associated with our qhub before we destroy it. Here is a possible avenue:

1. Login to the AWS console
2. Determine which Virtual Private Cloud your QHub EKS cluster is using.
3. From EFS find your cluster used EFS share and the ID, something like fs-9261c667.
3. Launch an AWS instance with Amazon Linux as OS. Choose the same VPC from Step 2. In security groups make sure you give rights to NFS and SSH.
4. Login to the AWS instance.
5. Install efs mount helper and mount the share (Instructions here: [https://docs.aws.amazon.com/efs/latest/ug/mounting-fs-mount-helper.html#mounting-fs-mount-helper-ec2](https://docs.aws.amazon.com/efs/latest/ug/mounting-fs-mount-helper.html#mounting-fs-mount-helper-ec2)): 

        sudo yum install -y nfs-utils
        sudo yum install -y amazon-efs-utils
        sudo mkdir /mnt/efs
        sudo mount -t efs fs-9261c667:/ /mnt/efs
        aws configure --profile efs_backup.  # enter your AWS KEY and SECRET_KEY WITH write rights to the backup bucket you want to use
     
6. Assume you mounted the EFS share on mountpoint /mnt/efs
        
        aws s3 cp --profile efs_backup --recursive /mnt/efs/home s3://<my-backup-bucket>/<my-backup-Prefix>
        
7. Terminate your ec2 AWS Linux instance.


## Q: How do I destroy my qhub deployment on AWS?
Answered by: @jkellndorfer

On your local qhub repo:

      cd <your-qhub-repo>
      git pull              # To get the latest changes 
      cd infrastructure
      terraform destroy
      

If `terraform destroy` does not complete and stops with an `unauthorized` error, try running it a second or several more times until you are stuck at the same `module`. Then try to manually remote the "offending" module with

      terraform state rm <module>

and the try `terraform destroy` again. Repeat that process possibly with other modules where the destroy process fails.

Final step: Login to your AWS console and look for everything related to the eks, loadbalancer, database tables, autoscaling groups, node groups, efs, volumnes, iam, etc. that may still be left and remove. (More details forthcoming!)


## Q: How do I use `nbconvert --execute` on qhub from commandline?
Answered by: @jkellndorfer

Assume the notebook `my-notebook.ipynb` uses the environment `myenv`.

1. Activate the environment used in the notebook

        conda activate myenv

2. Check that the environment is part of the string returned by `jupyter kernelspec list`

        jupyter kernelspec list
        python3    /home/conda/store/802e4196e4af0f9dbc000362cdb3bfde2df34aa9512bcfa6511c384ccef4518f-myenv/share/jupyter/kernels/python3
      
Interestingly, the kernel name is "python3", but contains,  ...-myenv/share ..., so we are ok.

3. Convert the notebook using the `python3` label:

        jupyter nbconvert --ExecutePreprocessor.kernel_name=python3 --execute --to html my-notebook.ipynb
      
You should get a `my-notebook.html` file that was executed with the myenv kernel. 

## Q How do I back up my user data using tar?

1. Create the pod.yaml for a ubuntu pod to back up from

        kind: Pod
        apiVersion: v1
        metadata:
          name: volume-debugger-ubuntu
          namespace: dev
        spec:
          volumes:
            - name: volume-to-debug-ubuntu
              persistentVolumeClaim:
               claimName: nfs-mount-dev-share
          containers:
            - name: debugger
              image: ubuntu
              command: ['sleep', '36000']
              volumeMounts:
                - mountPath: "/data"
                  name: volume-to-debug-ubuntu
   
2. deploy the pod

        kubectl apply -f pod.yaml -n dev
4. login to pod using k9s: Use arrow keys to highlight pod, then click "s".  You will be root.
5. install software needed for backup and transfer to S3
 
        apt update
        apt install vim curl zip unzip
        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
        unzip awscliv2.zip
        ./aws/install
        aws configure
6. run the tar command

        cd /data
        tar cvf 2021-03-home.tar home
8. transfer to s3
        
        aws s3 cp 2021-03-16-home.tar  s3://esip-qhub/backups/
