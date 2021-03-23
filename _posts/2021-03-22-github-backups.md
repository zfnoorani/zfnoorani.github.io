---
layout: single
classes: wide
title:  "Simple Automated GitHub Backups"
date:   2021-03-22 12:00:00 -0600
categories: tricks
---

If you're as reckless with your personal projects as me, you're just one `git push --force` away from catastrophe.

Alternatively, maybe your [pass](https://www.passwordstore.org/) database is on GitHub.
Perhaps GitHub is on the brink of folding and we don't know it.
Either way, backing up your GitHub repositories is wise.

Thanks to a generous GitHub Actions free tier, this is remarkably simple.

My backup target is AWS S3, but there are certainly cheaper backup repositories available.
Still, assuming your repos are small and you make use of lifecycle rules, several months of backups should easily fit into the 5GB S3 free tier.

My requirements are simple:
- Git-level backup of all public/private repositories under my name.
- Taken every three days.
- 30-days retention.
- GPG encrypted.
- Notify on failure.
- Switching to use a different S3-compatible provider should be easy.

While there is a [built-in profile exporter](https://docs.github.com/en/github/understanding-how-github-uses-and-protects-your-data/requesting-an-archive-of-your-personal-accounts-data) in GitHub, it is very slow. At the time of this writing it's not available as an API and thus cannot be reasonably automated.

### GitHub Action
Stick this in a public repo, for that sweet GitHub Actions free tier.

We'll need the following [secrets](https://docs.github.com/en/actions/reference/encrypted-secrets):
- `GITHUB_TOKEN`: A GitHub PAT with `repo` and `read:org` scopes.
- `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`: Credentials for your bucket.
- `AWS_S3_BUCKET`: The bucket.
- `GPG_PUBLIC_KEY`: Your GPG public key.
- `EMAIL`: The GPG target. For backups, you generally use the email you used to create your GPG key.

{% raw %}
```yaml
name: backup
on:
  workflow_dispatch:
  schedule:
    - cron: '0 6 */3 * *'
jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: backup
        run: |
          export AWS_DEFAULT_REGION=us-west-2
          
          # Import GPG public key
          echo "$GPG_PUBLIC_KEY" | gpg --import

          gh repo list --source -L1000 | awk '{print $1}' |
          	parallel git clone "https://user:$GITHUB_TOKEN@github.com/{}"

          tar cvzf - ./ |
            gpg --encrypt --trust-model always -r "$EMAIL" |
            aws s3 cp - "s3://$AWS_S3_BUCKET/github/github-$(date +"%Y%m%d_%H%M%S").tar.gz.gpg"
        env:
          GITHUB_TOKEN: ${{ secrets.GH_TOKEN }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_S3_BUCKET: ${{ secrets.AWS_S3_BUCKET }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          EMAIL: ${{ secrets.EMAIL }}
          GPG_PUBLIC_KEY: ${{ secrets.GPG_PUBLIC_KEY }}
```
{% endraw %}

Let's break the more obscure bits down.

```yaml
    - cron: '0 6 */3 * *'
```
Trigger this job at 0600 every 3 days.
[Crontab Guru](https://crontab.guru/) is your friend.

```sh
export AWS_DEFAULT_REGION=us-west-2
```
It doesn't really matter what region you specify, but setting this prevents `aws` from trying to discover the region.
I found in the actions environment, this discovery fails.

```sh
gh repo list --source -L1000 | awk '{print $1}' |
  parallel git clone "https://user:$GITHUB_TOKEN@github.com/{}"
```
We're using the [GitHub CLI](https://cli.github.com/), which is [helpfully installed](https://github.com/actions/virtual-environments/blob/main/images/linux/Ubuntu2004-README.md) on `ubuntu-latest`.
Fetch all repos, then clone them in parallel.
Doing so with [GNU Parallel](https://www.gnu.org/software/parallel/) lets us speed this job up substantially.

If you don't want to use GNU Parallel, a simple for loop could do the trick:
```sh
for r in $(gh repo list --source -L1000 | awk '{print $1}'); do
  git clone "https://user:$GITHUB_TOKEN@github.com/$r" &
done
wait
```

Then, let magic happen.
```sh
tar cvzf - ./ |
  gpg --encrypt --trust-model always -r "$EMAIL" |
  aws s3 cp - "s3://$AWS_S3_BUCKET/github/github-$(date +"%Y%m%d_%H%M%S").tar.gz.gpg"
```
Pardon the long chain.
`tar` compresses the current directory, outputting to `stdout` to begin a stream.
Remember the `czvf` [mnemonic](https://twitter.com/_tessr/status/626076327133577216?lang=en): Compress Ze Vucking Files

Then, encrypt the tar stream with `gpg`.
The `--trust-model always` flag tells `gpg` to implicitly trust our public key.
We provided it, after all.

Finally, upload the stream to AWS S3 under the `github/` subdirectory with the format `github-19700101_120000.tar.gz.gpg`.

The rest of this action is importing the secrets as environment variables.
This job will run automatically, or let you trigger it manually.
If it fails, and assuming your notifications are at their defaults, you'll be notified through email.

### GPG Encryption
There's lots of available on how [GPG works](https://docs.github.com/en/github/authenticating-to-github/generating-a-new-gpg-key).
If you don't need your backups encrypted, you can take out the `gpg` part of the action.

### AWS S3
Feel free to skip the rest of this article, if you already have your own S3-compatible infrastructure set up.

I manage all my infrastructure with [Terraform](https://www.terraform.io/).
Hopefully, you're using or considering using some kind of infrastructure-as-code solution yourself.
[This](https://blog.gruntwork.io/a-comprehensive-guide-to-terraform-b3d32832baca#.b6sun4nkn) guide is enough to get you started.

For this project, let's start with a bucket:
```terraform
resource "aws_s3_bucket" "bucket" {
  bucket = "bucket"
  
  # Private bucket!
  acl    = "private"

  # Enable versioning, so rogue overwrites or removals aren't as destructive.
  versioning {
    enabled = true
  }

  # Enable server side encryption.
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }

  # Lifecycle rules let you automate cleanup tasks.
  lifecycle_rule {
    id      = "delete-old-github-backups"
    enabled = true
    
    # Target the `github/` folder.
    prefix  = "github/"

    # Mark files as expired after 30 days.
    expiration {
      days = 30
    }

    # Delete expired files 30 days after being marked expired.
    noncurrent_version_expiration {
      days = 30
    }
  }
}

# Ensure the `github/` folder exists.
resource "aws_s3_bucket_object" "github" {
  bucket = aws_s3_bucket.bucket.id
  key    = "github/"
  source = "/dev/null"
}
```

Lock it down with more paranoia.
This will prevent any new uploads from having permissive ACLs applied to them.
```terraform
resource "aws_s3_bucket_public_access_block" "bucket" {
  bucket                  = aws_s3_bucket.bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

Then we need a service account for our action.
Don't use your main credentials!
GitHub Actions is [not a trusted store](https://blog.teddykatz.com/2021/03/17/github-actions-write-access.html) of superuser credentials.

```terraform
resource "aws_iam_user" "github_put" {
  name = "github-put"
}

resource "aws_iam_policy" "github_put" {
  name = "github-put"
  path = "/"

  policy = <<EOT
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:PutObject"
      ],
      "Effect": "Allow",
      "Resource": "${aws_s3_bucket.bucket.arn}/github/*"
    }
  ]
}
EOT
}

resource "aws_iam_user_policy_attachment" "github_put" {
  user       = aws_iam_user.github_put.name
  policy_arn = aws_iam_policy.github_put.arn
}
```
Again, paranoia is key.
This service account can only write to the `github/` subdirectory in the `bucket` bucket.
Even if its credentials are leaked, any other backups in the repo are safe, whether through inaccessibility or versioning.

One `terraform apply` later, all infrastructure should be provisioned.
Use `aws iam create-access-key --user-name github-put` to get credentials for the service account that can be populated into GitHub Actions.

### Restores
Download the latest backup.
Using the AWS console for this isn't cheating.

Then, Extract Ze Vucking Files:
```bash
gpg --decrypt FILENAME.tar.gz.gpg | tar xzvf
```

### Summary
This solution fits all my needs and fits into the free tiers of GitHub Actions and AWS.

Where it may fall flat is the inability to capture issues and other repository metadata.
If you have this need, I'd suggest messing with the `gh` command for exporting issues.
