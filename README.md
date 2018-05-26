# Workshop JenkinsX

## Prerequisites

You will need:

* A browser
* A ssh client (putty, ssh) and connection to the internet
* A github.com account - if you have none we will provide one

The following clusters and ssh users have been created. Please substitute YOURCLUSTER below, with your chosen cluster. 

I created the following clusters for you

* alpha
* beta
* gamma
* delta
* epsilon
* zeta

## Initial checks

### SSH into jumphost

use your ssh client and the provided credentials to log into our provided jumphost

### Check if your cluster is up and running

```
kops validate cluster
```

Should return "Your cluster YOURCLUSTER.k8s.hackaburg.be.continental.cloud is ready"

Please note down your cluster domain name, because we will need this domain later for jenkins installation.

### Check if your AWS account is setup correctly

```
aws s3 ls
```

If it returns a bucket "infra..." everything is ok

### Retrieve your credentials to access the cluster

You'll need a set of credentials and a token to access the kubernetes cluster dashboard:

1. Basic auth credentials for the api

   ```
   cat ~/.kube/config
   ```

   look for the entry username/password. The username is 'admin' and the password is your api password.

2. Retrieve the access token 

   ```
   kops get secrets --type secret admin -oplaintext
   ```

Open the cluster dashboard

https://api.YOURCLUSTER.k8s.hackaburg.be.continental.cloud/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

First provide your admin credentials and in the second dialogue choose "Token" and paste in your token. You should now have access to the cluster.

Everything ok? Then continue

## Install JenkinsX

First we need to install JenkinsX. The jx executable is already installed on the jumphost.

```
jx install
```

Select **kubernetes**. Answer the question "No ingress controller found..." with **Yes** to install a ingress controller in the kube-system namespace.

Choose **Yes** again to wait for jenkins for the new AWS load balancer to be created. This takes a few moments.

While we wait, we have to do a manual step in the AWS console. Currently jenkinsx is not able to create DNS entries in AWS Route53. So we have to add it. First copy the name of the loadbalancer that is shown (e.g. ac51aac...-34223...) then navigate to the AWS console in your browser.

https://649125951619.signin.aws.amazon.com/console

* Go to Services
* Choose Route53
* Select hosted Zone **k8s.hackaburg.be.continental.cloud**
* Choose **Create Record Set** 
  * Enter as Name: **\*.YOURCLUSTER** (e.g. *.alpha)
  * Select Alias **Yes** 
  * Paste the Loadbalancer name as the Alias Target. If the Loadbalancer is not listed you'll have to wait a few seconds. Try reloading the page if it is not showing

No we can go back to your terminal session.

For the domain please enter: **YOURCLUSTER.k8s.hackaburg.be.continental.cloud**

Now enter a github username (you can use the provided account) because jenkins needs access to github. You'll have to create a token. Just follow the link provided by jx and paste the newly created token.

Jenkins will now launch all the necessary pods in kubernetes. You can check the creation in the kubernetes dashboard.

JenkinsX now needs a API Token from Jenkins. It will show you the Url and print the admin password. Log in to Jenkins and select **Show API Token** paste the Token.

Your all set. The jenkins installation is up and running.

## Create our first application

Now let's create a simple spring demo application.

```
jx create spring -d web -d actuator
```

You will be asked to fill in your prefered language, Group and Artifact. Just keep the default value. **Please choose a unique name other than _demo_ as the artifact and repo name**. Just use the provided default values for the rest of the questions.

After everything is pushed, Jenkins will start its first build which you can follow by issuing

```
jx get build logs
```

When the build has finished you can get the url of the newly created environment

```
jx get app
```

If you follow the link, you'll see a whitelabel error page, because there is no content to show.

### Add a homepage

Let's change this and add a homepage.

First we'll create a github issue for this...

```
cd YOUR_PROJECT_NAME
jx create issue -t 'add homepage'
```
...and do our work

```
git checkout -b feature/add-homepage
echo "<h1>It's working</h1>" > src/main/resources/static/index.html
git add .
git commit -m "Added homepage fixes #1"
git push origin feature/add-homepage
```

Now let's create a pull request for this. You can do this on the github website or via the command

```
hub pull-request
```

Jenkins will now trigger a ci job for this pullrequest automatically. This will take some time, but jenkins will update the pull-request page on github automatically. Again you can see the build logs with the ```jx get build logs``` command.

Switch to the github website and have a look at the pull request. When the build is ready you will see a link to the staging environment inside the pull-request. If you open up the link, you'll see our new index.html.

Now it's time to close our pull request. Just hit the green button to merge the pull request.

### Bring you app into production

As the last step we're going to promote this build to production.

You can see all environments with

```
jx get env
```

switch to the production env via

```
jx env
```

and select **production**. We'll promote the version and bring it live.

```
jx promote YOUR_PROJECT_NAME --env production --version 0.0.2
```

When the build is finished you should see with ```jx get apps``` that the application is now runnin in two environments staging and production. follow the production link from the terminal or find it as a comment on the github page and you should see our new homepage as well.

Well done!

# Now it's your turn

Try to create issues, explore the cluster, build stuff.
