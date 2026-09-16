Hello pwners! This is my first writeup in the world of cloud penetration testing specifically AWS. My goal is to consolidate the knowledge I gained from this lab and to share that knowledge with others

The goal of this lab is to teach us the initial phase of AWS penetration testing which is the enumeration with a focus on S3 enumeration. If you are unfamiliar with S3, you should learn the fundamentals of AWS before proceeding.

The site starts by providing a web link, and the visible URL indicates that it is hosted on AWS.

The site begins by providing a link to a webpage that, upon opening, appears to be a completely ordinary webpage.

<img width="1920" height="1035" alt="image" src="https://github.com/user-attachments/assets/07c977a5-6498-41b3-aa24-26e9288fda58" />

So, let's take a look at the page's source code.

<img width="1920" height="1035" alt="image" src="https://github.com/user-attachments/assets/36360bb8-e5d4-4aa5-9b1e-39f3ac13eeac" />

```html
<meta charset="UTF-8">
<title>Huge Logistics</title>
<link rel="stylesheet" href="https://s3.amazonaws.com/dev.huge-logistics.com/static/">
```

Interestingly, it now appears to be using an S3 bucket named `dev.huge-logistics.com` to store static files.

When I tried to play with that URL in the browser, I didn't find anything useful; everything just resulted in "access denied messages."

<img width="1920" height="1035" alt="image" src="https://github.com/user-attachments/assets/02417dbf-d900-4521-acb2-285a07621725" />

So let's try using AWS CLI

```shell
sudo apt install awscli
```

After installing it, come with me and let's see what we can do with it

```shell
aws s3 ls s3://dev.huge-logistics.com --no-sign-request
```

Let's break it down:

aws: is the tool name s3: is the name of the service that we want to interact with ls: is the action that we want to do (here we are listing the contents) --no-sign-request: means that the request is anonymous (think of it like the FTP anonymous login)

<img width="1877" height="299" alt="image" src="https://github.com/user-attachments/assets/a24902dd-aa9a-414a-b19b-f32438d7a0bf" />

Here we go! My request succeeded, returning a list of the contents.

We can now explore each one.

```shell
aws s3 ls s3://dev.huge-logistics.com/admin/ --no-sign-request
aws s3 ls s3://dev.huge-logistics.com/migration-files/ --no-sign-request
aws s3 ls s3://dev.huge-logistics.com/shared/ --no-sign-request
```

Actually, I am permitted to see the content of the `/shared` folder only using the anonymous login.

<img width="1631" height="87" alt="image" src="https://github.com/user-attachments/assets/ffde364d-6db0-482f-bee9-2220fa04a828" />

It has one zip file called `hi_migration_project.zip`. So let's get it on my machine:

```shell
aws s3 cp s3://dev.huge-logistics.com/shared/hl_migration_project.zip . --no-sign-request
```

Here we replaced `ls` with `cp` and you can observe its role.

After unzipping the file, we see a PowerShell script.

<img width="1808" height="321" alt="image" src="https://github.com/user-attachments/assets/15d85ded-5977-4707-b6ee-6c2841ff3bd7" />

Let's see what it hides:

```powershell
# AWS Configuration

$accessKey = "AKIA3SFMDAPOWMWJSZZZ"

$secretKey = "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"

$region = "us-east-1"

  

# Set up AWS hardcoded credentials

Set-AWSCredentials -AccessKey $accessKey -SecretKey $secretKey

  

# Set the AWS region

Set-DefaultAWSRegion -Region $region

  

# Read the secrets from export.xml

[xml]$xmlContent = Get-Content -Path "export.xml"

  

# Output log file

$logFile = "upload_log.txt"

  

# Error handling with retry logic

function TryUploadSecret($secretName, $secretValue) {

$retries = 3

while ($retries -gt 0) {

try {

$result = New-SECSecret -Name $secretName -SecretString $secretValue

$logEntry = "Successfully uploaded secret: $secretName with ARN: $($result.ARN)"

Write-Output $logEntry

Add-Content -Path $logFile -Value $logEntry

return $true

} catch {

$retries--

Write-Error "Failed attempt to upload secret: $secretName. Retries left: $retries. Error: $_"

}

}

return $false

}

  

foreach ($secretNode in $xmlContent.Secrets.Secret) {

# Implementing concurrency using jobs

Start-Job -ScriptBlock {

param($secretName, $secretValue)

TryUploadSecret -secretName $secretName -secretValue $secretValue

} -ArgumentList $secretNode.Name, $secretNode.Value

}

  

# Wait for all jobs to finish

$jobs = Get-Job

$jobs | Wait-Job

  

# Retrieve and display job results

$jobs | ForEach-Object {

$result = Receive-Job -Job $_

if (-not $result) {

Write-Error "Failed to upload secret: $($_.Name) after multiple retries."

}

# Clean up the job

Remove-Job -Job $_

}

  

Write-Output "Batch upload complete!"

  
  

# Install-Module -Name AWSPowerShell -Scope CurrentUser -Force

# .\migrate_secrets.ps1
```

Wow! The script contains hardcoded AWS keys.

```powershell
# AWS Configuration

$accessKey = "AKIA3SFMDAPOWMWJSZZZ"

$secretKey = "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"

$region = "us-east-1"

```

You can consider the `accessKey` as a `username` and the `secretKey` as a `password`.

Now let's configure the AWS CLI to use these credentials instead of an anonymous connection.

<img width="1019" height="141" alt="image" src="https://github.com/user-attachments/assets/dfc824c6-4581-46e1-8e91-912ad806f05c" />
 ``
 
Now the AWS CLI has credentials.

```bash
aws sts get-caller-identity
```

This is like `whoami`; it allows us to find out our execution context.

<img width="1218" height="178" alt="image" src="https://github.com/user-attachments/assets/215f7847-efc2-48a5-8809-cd0e8185c336" />

The IAM user whose credentials we used is named `pam-test`.

Now, let's try listing the contents of the paths that I couldn't see using the anonymous login: `/admin`, `/migration-files`.

<img width="1716" height="144" alt="image" src="https://github.com/user-attachments/assets/101d8d7c-1dbf-477b-ba25-c574e26c4d03" />

In the `/admin` directory, I am able to see the content but I can't dump it.

Let's try with `/migration-files`.

<img width="1874" height="209" alt="image" src="https://github.com/user-attachments/assets/34b741d9-ff2f-47e9-b794-82f3c5a70700" />

```plain
2023-10-16 18:08:47          0 
2023-10-16 18:09:26    1833646 AWS Secrets Manager Migration - Discovery & Design.pdf
2023-10-16 18:09:25    1407180 AWS Secrets Manager Migration - Implementation.pdf
2023-10-16 18:09:27       1853 migrate_secrets.ps1
2026-01-30 21:53:58       2494 test-export.xml

```

In the `migration-files` directory, I am able to see and download the files.

Let's take a look at `test-export.xml`:

```bash
aws s3 cp s3://dev.huge-logistics.com/migration-files/test-export.xml .
```

<img width="1247" height="556" alt="image" src="https://github.com/user-attachments/assets/4b702bb3-b236-46d3-b983-0836f032905e" />

```xml
<CredentialEntry>

<ServiceType>AWS IT Admin</ServiceType>

<AccountID>794929857501</AccountID>

<AccessKeyID>AKIA3SFMDAPO6DGDLJAG</AccessKeyID>

<SecretAccessKey>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</SecretAccessKey>

<Notes>AWS credentials for production workloads. Do not share these keys outside of the organization.</Notes>

</CredentialEntry>

```

Wow again!

It seems that we now have the `AWS IT Admin` credentials.

Now let's use `aws configure` again to set the new keys.

<img width="1150" height="158" alt="image" src="https://github.com/user-attachments/assets/6a5fcaf8-f89d-49f6-89c8-a71eefa5c4df" />

and `aws sts get-caller-identity` reveals that we are the IAM user `it-admin` 

<img width="1285" height="188" alt="image" src="https://github.com/user-attachments/assets/ab4e325d-275c-4672-a4ad-b11f5e712914" />

Let's now try to get the flag!

<img width="1620" height="124" alt="image" src="https://github.com/user-attachments/assets/d25cf4f7-374a-427b-891e-3f2afde0ff36" />

Yep, and here we go! I could retrieve the flag.

And that's a wrap, pwners!, one misconfigured bucket was all it took to go from anonymous to IT Admin. Stay curious, keep enumerating!
