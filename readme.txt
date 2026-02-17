This setup involves a bastion as a target to code deploy service.

The buildspec will copy all the data from codecommit as build artifact in s3.

Code deploy will get this build artifact from s3 and this will be available in code-deploy document root.

appspec.yml will copy thi data to /home/dev/node-app where the build will happen.

The the scripts will do their jobs which are referenced in appspec.yml


Bastion host is responsible for build and deployment.