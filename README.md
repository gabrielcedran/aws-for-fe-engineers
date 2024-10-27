# AWS For Frontend Engineers

## Notes:

### Account set up

**Root user vs IAM user:**

Pretty much like linux's users, you should not be going around with a root user.

Ideally, as soon as the account is set up with the root user, an administrator user should be created (stop being root).
Then when other processes are created - like CI/CD process - other users with less privilege should be created (e.g a user that
only has specific permission to put files in a specific bucket or just create an invalidation on Cloudfront).

_We should seek the principle of the least privilege / access_

MFA should be set up for root and administrator users.

Steps to create an administrator user:

1. Acces IAM service (from the root account)
2. Create an alias for the account (it creates a custom url for IAM users to log in to the account)
3. Open the users menu and click `Create User`
4. Fill in username
5. Decide if the user needs access to the console (or only programmatically)
6. Set the permissions (for administrator, set `AdministratorAccess`, which grants access to everything but billing, which should be exclusive for the root) - or create a group for admins
7. Tags are metadata that can be put on anything on aws for management (optinal) - ex: tag departments to easily identify on bills

_instead of creating users manually and individually, it's possible to use IAM Identity Center, which users the organasation identity source like OKTA, Google workspace, Microsoft Active Directory, etc_

**Alerts for costs:**

To set up alerts for when costs go above a certain threshold, go to `Billing and Cost Management` -> `Billing Preferences` ->
`Alert preferences` -> `Receive AWS Free Tier alerts` (we can also set `CloudWatch billings alerts`)

To set up a budget, go to `Billing and Cost Management` -> `Budgets` -> `Create a Budget` -> Choose/Setup one according to expected usage

### S3

It can store any type of files (formally known as objects), including javascript (and static assets) that consititutes an application or website
(thus it can be used to host web pages). Objects are stored in buckets.

Some other supported features are versioning, encruption and security.

Uploading to S3 is free, but the storage itself and serving the files (requests to the bucket) are charged (it's possible to set up limits to these to avoid abuse/misuse).

When using to host a website, the bucket name has to match the website's url (this requirement can be avoided by using cloudfront).

**Manual bucket creation**

When creating new buckets, the most important thing to pay attention is the permissions for public access (as this one could rack up a huge bill).
If you are hosting a public website, then accessing the bucket publicly is necessary (tho it's possible to expose the content via
cloudfront instead of makeing the bucket public, which is covered later).

Policies: user can have policies (what they have access and are allowed to do), resources can also have policies.
What would we want a public bucket hosting a website to be able to do?

1. people should be able to read from it
2. people shouldn't be able to put anything

So basically for this public bucket, we want to set a policy to allow people to read, but not put or delete:

Open the bucket and click on the `permissions` tab. Scroll to the `bucket policy section` and `edit`.

- Principal: who this policy applies to (`*` is anyone)
- Effect: allow or deny
- Action: what they should be able to do
- Resource: what they should be able to do it to

Example: this user (`principal`) can read files (`action`) from bucket x (`resource`).

_Policy Generator is a convenience tool to generate policies in a more friendly way. In the end it generates a json to be pasted in the policy input_

_it's possible to allow users to get any file or a particular file of a bucket_

_inside properties, it's possible to set up the bucket to host a static website, which provide relevants config for this purpose_

#### Redirecting from one bucket to another (www to no www domain)

When a bucket is created without www prefix, in order to have the website working with both versions (with or without), it's necessary to create a second bucket with the desired name.

Instead of managing 2 buckets and deploying assets to both, it's possible to simple point the second one to the main bucket.

To do so, 1. create a new bucket (with the secondary domain name) - leave all the configs as the default, 2. open bucket properties, 3. go to `static website hosting`, enabled it and choose the hosting type `redirect requests for an object`, 4. point to the other bucket's website address

_at this point, the new bucket has an amazon's branded url. To have the custom one, create a DNS `a record` for the new domain (starting www) and point it to its bucket - more detail in section Route 53_

### Route 53

It's amazon's DNS service. It offers features like detecting servers that are down to re-route to one that is up, or detecting servers that are slow from a given place to re-route to another that is faster.

It's possible to connect (point) domains bought outside amazon to route 53. However buying new domains via Route 53 makes it way easier to hook up with other amazon's services.

#### Set up custom domain for s3 bucket hosting

Open Route 53 -> Registered Domains -> Manage DNS -> Select domain and click `view details` -> Create record -> Enable `alias` (advantage of buying the domain from aws\*\*) -> On the Route Traffic to dropdown, select `Alias to S3 website endpoint` -> Select region and website entry point -> Create records

\*\* It allows the possibility of alias a domain to another amazon resource

### Certificate Manager

To create a ceritificate and enable `https` on websites (ps it's not possible to request a certificate for amazon's owned domain names):

1. Click `request a certificate` button
2. Set the fully qualified domain name (create 2, one for the url with www and one without at least)
3. Prove that you own the domains (little secret added to the DNS record - if done via route 53, it's automatic step #5)
4. Click `request` button
5. Open the created certificate and hit the `Create records in Route 53` button (if domain purchased via aws)

### AWS Cli

To set up the `aws cli`, install it and then execute `aws configure`. It's going to ask the access key and token (which can be generated directly on IAM service).

To have multiple profiles, just run the commands with `--profile {profile_name}`. E.g `aws configure --profile abc`.

Credentials are stored in `~/.aws/credentials`.

Each AWS has a command under aws cli. Example: `aws s3 ls` and `aws s3 ls s3://{bucket_name}`.

#### Deploying aps via aws cli

1. build the project `npm run build`
2. copy the built assets into s3 `aws s3 cp {dir/assets} s3://{bucket} --recursive`. Eg `aws s3 cp build s3://{bucket_name} --recursive`

### Cloudfront

When you have a bucket in a given region, the further away you get from that location, the slower the objects take to load. This is where cloudfront comes into play - it's amazon's CDN solution. It sets up the assets on a CDN around the world.

Naming: Cloudfront distribution is simply a cloudfront set up (S3 has buckets, Cloudfront distributions)

_cloudfront can wrap virtually anything: s3 bucket, apis, datacenters, etc. Whatever goes through that route, gets cached_

#### Set up

1. Open Cloudfront on console and hit `Create a Cloudfront distribution`
2. For the `Origin domain` add the actual website url (if it's hosted in a S3 bucket, copy the actual amazon's provided url, do not pick from the list)
3. Origin Shield is a good feature to protect the actual server (but if the static are hosted on s3, it's not the most important thing)
4. In the cache policy, it's possible to determine different policy for different resources (like images have a different cache policy to javascript)
5. Allowed HTTP Methods: it's possible to put cloudfront in front of an API (either via api gateway or an ec2 instance, own data centre, etc), thus when enabling all http methods is useful
6. (Optional) Cache key and origin requests [recommended might be good enough]
7. (Optional) Function associations `lambda@edge`
8. (Optional) Web Application Firewall - useful but comes with a cost
9. Price class - according to where users are expected to be located
10. When using custom domain, add Alternate Domain Name (CNAME) according to custom domains
11. For custom SSL HTTPS certificate, choose the certificate created on `Certificate Manager` and the Security Policy
12. Choose http versions that you want to support
13. Define the default root object: `/index.html` (default file for the app)
14. (optional) if route 53 is pointing to the s3 bucket, change it to point to cloudfront: in the alias part, select cloudfront (here there is the option for setting up smart routing)

#### Cloudfront and S3

Buckets are stored in a given location, which might be far away from (some) end users and therefore inefficient -
replicating buckets in a different regions could be a solution but it's not efficient either. Cloudfront is
geographically distrubuted out of the box, closing this gap.

Bonus 1: serving files out of S3 has a cost per request. Cloudfront caches the resources, reducing the number of hits on S3
(and its price per request is cheaper).

Bonus 2: When hitting S3 directly, the request goes through the public internet, when using cloudfront the request hits the cloudfront edge node (which is closer) and then the request goes through amazon's private network (which is generally faster than going through the public internet).

_ps: usually the first user pays the price to load the object from the bucket and load into all the caches in that route. (regional edge case and edge node)_

**OAI (Origin Access Identity)** It's a feature that allows buckets to be private (not accessed publicly) but still allow cloudfront to access it.

1. Open the wanted distribution and click on `Origins` tab.
2. Select (or create) the S3 bucket and hit `edit`.
3. On the origin input, select the bucket from the dropdown (here, select from the dropdown as the OAI is being set up).
4. Then select `Origin access control settings` on the `Origin Access`.
5. Assign OAI so that only this user has access to the S3 bucket (if it doesn't exist, create one)
6. Update the s3 bucket access policy with the assigned user\*\*
7. Hit save

\*\* Follow the next steps to update the s3 bucket policy:

1. Open the bucket and go to permissions tab
2. Block all public access
3. In bucket policy, update it with the one provided in step 6 - if using the legacy, get the user OAI id under Security -> Origin Access. The principal should be ` { "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity {id}" }`
4. turn of static website hosting (as now it's cloudfront's responsibility - main difference between having the origin as a forward url or the actual s3 service access)

Ps1: if the default root ins't working, go back to cloud front settings and remove the root slash `/`from the `default root object`.
Ps2: at this point, client side routing for not found routes is failing, which makes sense as cloudfront is in charge now (thus that not found directive set on s3 bucket is ineffective) - on cloudfront this can be done under the directive's tab `Error pages`.

#### Headers

Cloudfront ignores most of the request headers and adds its own ones.

#### Cache

Cache policy is set under the `behaviour tab`. The recommended settings cache files by 24 hours. What happens when a new version of the app is released?

Potential solutions (with their trade-offs) are:

1. Have the index.html have a much shorter TTL (5 seconds, 1 minute, 5 minutes, whatever is suitable given the audience):

To do so, create a new behaviour and set the path pattern to `/index.html` and set a different policy for it

The downside of this approach is that no matter the amount of time, if a bad version is pushed, that bad version with live for that amount of time (rollback)

2. Use invalidations (under `Invalidations` tab): when creating an invalidation, simply add all the patterns that cloudfront should invalidate (forget).

The limitation here is the amount of validations that are provided for free (1000 per year) - but even above this, it's super cheap (`/*` counts as one invalidation).

### CI Pipeline with GH Actions

First of all, it's importanto to have a user that has the bear minimum to build and deploy the application, nothing more - you don't want to use an admin user. In this application, the user has to (1) upload files to the S3 bucket (2) issue invalidations on cloudfront.

Steps:

1. Go to IAM and create a new policy (policies -> create policy):

- In the service dropdown, select cloudfront
- In the actions allowed combobox search for CreateInvalidation and select it
- In resources leave the radio Specific selected and add the cloudfront resource distribution id (pick it up from cloudfront)

- Hit the Add more permissions button
- In the service dropdown, select s3
- In the actions allowed combobox search for and select ListBucket and PutObject

- Add a meaningful and create the policy

2. Go to users and hit `Create User` (this one won't need access to console, only an api key). Assign the Policy created in the step above and complete the creation.

3. Create the github action (`checkout`, `setup aws client`, `build project`, `deploy to s3 using aws cli` and `run invalidation on cloudfrom using aws cli`). Ps at least `access key` and `secret access key` should be parameterised via GH secrets (but adding bucket name and distribution id is also a good idea).

4. Create the github secrets

### Lambda Functions

The basic concept of lamnda functions is that instead of managing servers, you simply write some code in your favourite programming language, hand it over to aws and they take care of it for you from management perspective (run my code, update my linux, auto-scale the settings, etc).

**Lambda functions execute code in response to events**

#### Lambda@edge

Simply put, it's a lambda function deployed and executed on Cloudfront edge nodes.

For every request and every response, if you need to change anything, you can do on the fly (e.g modify request headers or cookies for ab testing, bluegreen deploy, set csp policies, etc). It can even fetch a database.

There are 4 places where lambda@edge can be added: (1) on the viewer request (between the user's browser and cloudfront, before cache is checked), (2) origin request (between cloudfront and the origin server, e.g s3 - this is only applied in case of cache miss), (3) origin response (the other way around, obviously also in case of cache miss), (4) Viewer response (the other way around).

**setup**: The set up of lambdas@edge follows pretty much the same process of regular lambda (in this example, it'll add rewrite some headers):

1. Go to lambda service and hit `create function` button
2. fill in the details (lambda name, runtime programming language, enable networtk, etc)
3. complete lambda creation
4. add the code and hit `deploy` button (at this point it's deployed to lambda but not cloudfront - _this creates a role for the lambda that will be edited in the next step_)
5. go to IAM -> roles -> and look for the recently created lambda role. Then open the `Trust Relationships` tab and edit the policy -> change the `Service` attribute to be an array and add `edgelambda.amazonaws.com` - by default, only lambda.amazonaws.com is added.
6. go to back to the lambda and open `actions` (top left corner) and click `deploy to Lambda@edge`
   5.1 alternatively: add trigger for cloudfront and hit `deploy to Lambda@edge` - (ps it's gotta be in us-east-1, otherwise you won't see cloudfront)
7. set up the next modal by choosing the distribution, cache behaviour, `CloudFront event (Viewer Request, Origin Request, etc)` and click `deploy`

_To see all lambda@edge from a cloudfront distribution, simply head to the Behaviours tab of the cloudfront console and hit edit - it'll be under a section called `Function associations`._

#### CloudFront functions

It's a completely different functionality to `lambda@edge`. It can do way less - it cannot call external things, use external libs, it's a subset of js, and has a shorter timeout, but it's way faster and cheaper (it can be seen a subset of lambda@edge).

It lives directly within CloudFront console but creation is similar to lambda@edge (create the function and assign to the cloudfront distribution at the stage that you want).

#### CSP (content security policy)

It's possible to set CSP header to improve the web application security. Mozilla observatory provides a nice and convenient tool to analyse the website / app - https://developer.mozilla.org/en-US/observatory
where is scans the provided url and give a score with a report with what is missing to improve it.

CloudFront doesn't set the CSP automatically, therefore setting them as a lambda@edge or CloudFront function is a good solution. Creation is pretty similar tho:

1. On CloudFront console, navigate to functions and hit create function.
2. Give it a name and choose the runtime js version
3. Add code and test
4. Hit publish
5. Go to the distribution and open the target behaviour (or create one)
6. Scroll down to Function Associations and assign the CF function to the desired stage (the same stages as Lambda@edge: `Viewer Request`, `View Response`, `Origin Request`, `Origin Response`) - be aware of the limitations when mixing cf functions and lambda@edge.

_restriction when mix and matching lambda@edge and CloudFront functions: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-function-restrictions-all.html_

### Other aws services to have a look:

- Cognito: authn and authz solution like auth0 and firebase auth (the cool thing about it is that it integrates neatly with s3 like give permission to a file only to the user that uploaded it)
- Dynamodb: learn more advanced features and the architectural style / mindset around it
- AppSync: a solution that speeds up the process of creating an infra to store, access, manipulate data (from one or multiple sources) leveraged by GraphQL (given the schema it can create the storage [DynamoDB?]). It also supports real time upadtes with Websockets
- Amazon Location Service: solution to add geolocation for applications
- Amplify

### AWS for Frontend Engineers

- [00 - Introduction](content/00%20-%20Introduction.md)
- [01 - The AWS Free Tier](content/01%20-%20The%20AWS%20Free%20Tier.md)
- [02 - Account Set Up](content/02%20-%20Account%20Set%20Up.md)
- [03 - Billing](content/03%20-%20Billing.md)
- [04 - IAM](content/04%20-%20IAM.md)
- [05 - S3](content/05%20-%20S3.md)
- [06 - Registering a Domain Name](content/06%20-%20Registering%20a%20Domain%20Name.md)
- [07 - S3 Policies](content/07%20-%20S3%20Policies.md)
- [08 - AWS CLI](content/08%20-%20AWS%20CLI.md)
- [09 - Route 53](content/09%20-%20Route%2053.md)
- [10 - Routing](content/10%20-%20Routing.md)
- [11 - CloudFront](content/11%20-%20CloudFront.md)
- [12 - Using OAI](content/12%20-%20Using%20OAI.md)
- [13 - Creating a Custom Cache Policy](content/13%20-%20Creating%20a%20Custom%20Cache%20Policy.md)
- [14 - Build Pipeline](content/14%20-%20Build%20Pipeline.md)
- [15 - Lambda@Edge](content/15%20-%20Lambda@Edge.md)
