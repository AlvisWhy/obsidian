cloud deployment is used for containerized app

In artifact registry, we can sign the image. Later we can only work on the image with the signature.
We can use the org policy(**binary authorization** required for cloud run/ GKE), to only allow signed images. 

Use 
git tag to give a tag for certain version

```
git tag staging-v1.2
git push origin staging-v1.2
```
Jenkins find the tag and trigger run test.

Cloud build github app
a github plug-in to trigger cloud build, substitute the webhook to write token