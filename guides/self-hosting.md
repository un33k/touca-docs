# Self-Hosting Instructions

Customers on our Enterprise Plan have the option to deploy an instance of the
Touca server on their own infrastructure. The instructions in this section are
intended for DevOps engineers and system administrators who may want to perform
this deployment.

{% hint style="success" %}

Our Enterprise Plan includes dedicated support and professional services for
deploying and upgrading self-hosted instances of the Touca server. Please feel
free to contact us if you had any question or needed any help during this
process.

{% endhint %}

## Prerequisites

We provide a dedicated _access token_ to our enterprise customers. You will need
this token to download any stable release of the on-premise version of the Touca
server.

We recommend that you deploy Touca on a machine with at least 2 GB of memory.

There is no restriction for the choice of Unix distribution. However, the
instructions that follow are written for and tested on Ubuntu 18.04 LTS
distribution.

## Prepare your server

{% tabs %}

{% tab title="Overview" %}

Self-hosting Touca involves downloading Touca Docker images from our AWS
Container Registry and running them via docker-compose on your machine.

The tabs in this section provide instructions for installing these required
tools on your machine. You can skip this section, if you already have Docker,
docker-compose, and AWS CLI installed.

{% endtab %}

{% tab title="Initial Server Setup" %}

{% hint style="info" %}

This section provides general best practices for setting up a virtual machine.
They are not specific to self-hosting Touca and are presented for completeness.

{% endhint %}

#### Create a New User

```bash
sudo adduser touca
sudo usermod -aG sudo touca
```

#### Add Public Key Authentication

```bash
rsync --archive --chown=touca:touca ~/.ssh /home/touca
```

#### Disable Password Authentication

```bash
sudo vim /etc/ssh/sshd_config
```

And set `PasswordAuthentication` to no. Finally, reload SSH daemon for your
changes to take effect.

```bash
sudo systemctl reload sshd
```

#### Setup Basic Firewall

Use the UFW firewall to make sure only connections to certain services are
allowed:

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

If configured correctly, you should see an output similar to the following:

```bash
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
```

{% endtab %}

{% tab title="Install Docker" %}

Update the apt package index:

```bash
sudo apt-get update
```

Install packages to allow apt to use a repository over HTTPS:

```bash
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
```

Add Docker’s official GPG key:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Setup the stable docker repository.

```bash
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Update the apt package index:

```bash
sudo apt-get update
```

Install the latest version of _Docker Engine - Community_ and _containerd_:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Since we do not want to preface every `docker` command with `sudo`, create a
Unix group called `docker`.

```bash
sudo groupadd docker
```

Add current user to the newly created docker user group.

```bash
sudo usermod -aG docker $USER
```

Now log out and log back in again and check if you can successfully run docker
without using sudo.

```bash
docker run hello-world
```

{% endtab %}

{% tab title="Install docker-compose" %}

Download `docker-compose` executable from artifacts of their latest GitHub
release:

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

Fix permissions of the downloaded binary:

```bash
sudo chmod +x /usr/local/bin/docker-compose
```

{% endtab %}

{% tab title="Install AWS CLI" %}

Download and install official AWS command line tools.

```bash
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./install
```

Once you verify that AWS CLI is installed, you can remove the downloaded
archive.

```bash
aws --version
rm awscliv2.zip
```

{% endtab %}

{% endtabs %}

## Obtain Docker Images

Assuming you have installed Docker, docker-compose, and AWS CLI on your machine,
we can start with obtaining Touca docker images from Touca container registry on
AWS. The commands in this section reference the following parameters that should
be replaced with credentials that we provide to you upon your purchase.

- `TOUCA_AWS_ACCESS_KEY_ID`
- `TOUCA_AWS_SECRET_ACCESS_KEY`
- `TOUCA_AWS_REGION`
- `TOUCA_AWS_REPO`

### Authenticate to AWS Container Registry

Run the following command to create an AWS profile.

```bash
aws configure
```

This command opens an interactive prompt to let you provide credentials for
Access Key, Secret, Region, and Output Format.

```bash
<TOUCA_AWS_ACCESS_KEY_ID>
<TOUCA_AWS_SECRET_ACCESS_KEY>
<TOUCA_AWS_REGION>
json
```

Now that your profile is set up, run the following to authenticate to the AWS
Container Registry.

```bash
mkdir ~/touca
aws ecr get-login-password --region <TOUCA_AWS_REGION>
```

The expected output of this command is a long text. We do not need to store it
anywhere.

Now that things are set up with AWS, we can login to the container registry via
Docker.

```bash
aws ecr get-login-password --region <TOUCA_AWS_REGION> | docker login --username AWS --password-stdin <TOUCA_AWS_REPO>
```

Now we can pull our images from the new registry.

```bash
docker pull <TOUCA_AWS_REPO>/touca-api:1.3
docker pull <TOUCA_AWS_REPO>/touca-app:1.3
docker pull <TOUCA_AWS_REPO>/touca-cmp:1.3
```

## Deploy Docker Containers

Download Touca deployment scripts from Touca admin dashboard, move it to the
production machine and install it in the appropriate path.

```bash
ssh touca@your-machine
mkdir touca; cd touca;
scp devops.tar.gz touca@your-machine:~/
tar -zxf ../devops.tar.gz
rm ../devops.tar.gz
```

Before running the docker containers, create the local directories \(volumes\)
to which they bind.

```bash
mkdir -p local/logs/backend local/logs/comparator
sudo chown 8002:touca local/logs/backend local/logs/comparator
mkdir -p local/data/minio local/data/mongo local/data/redis
```

Modify values of the following environment variables in
`devops/docker-compose.prod.yaml` file. Do not wrap the values in single or
double quotations.

- `AUTH_JWT_SECRET`, `AUTH_COOKIE_SECRET`

  We recommend a randomly generated string of 32 characters length.

- `MAIL_TRANSPORT_HOST`, `MAIL_TRANSPORT_USER`, `MAIL_TRANSPORT_PASS`

  Set these values based on your mail server configurations.

- `WEBAPP_ROOT`

  Root URL of the Touca server. Can be of the form
  `https://touca.your-company.com` or `http://172.129.29.29`.

Now run `devops/deploy.sh` to deploy Touca via `docker-compose`.

```bash
~/touca/devops/deploy.sh -r <TOUCA_AWS_REPO> -u AWS
```

Monitor standard output of docker containers to check that everything is running
as expected:

```bash
docker-compose -f ~/touca/devops/docker-compose.prod.yml --project-directory ~/touca logs --follow
```

At this time, you should be able to verify that Touca is up and running by
navigating to your machine address on a browser.

{% hint style="success" %}

Did we miss out a required step? We'd love to hear about your experience. Share
your thoughts with [support@touca.io](mailto:support@touca.io).

{% endhint %}