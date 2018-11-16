# Adding Anchore container image scanning to your Codefresh pipelines

## Background

Docker images themselves are commonly built with a continous integration tool like Codefresh. As with any CI pipeline building an artifact, it becomes important to employ the proper security testing tools to ensure a higher lever a confidence when deploying. In the case of Docker images, a mechanism for scanning and inspecting these artifacts.

Anchore is a service that conducts static analysis on container images, and applies user-defined acceptable polices to allow automated container image validation and certification. This means that not only can Anchore users gain a deep insight into the OS and non-OS packages contain within an image, but they have the ability to create governence around the artifact and it's contents via customizable polices. 

## Example

In this template, we will walk through how to configure a Codefresh pipeline to build an image from a Dockerfile, conduct an Anchore scan, evaluate the scanned image against an Anchore policy, and push it to a Docker registry.

Codefresh pipelines, are the core component of the Codefresh platform. These pipelines are workflows that contain user defined steps that are all executed inside a user chosen Docker containers. These steps are defined using a `codefresh.yaml` file. 

The Anchore scanning step will take place prior to the image being pushed to a Docker registry. Depending on the output of the policy evalution and pipeline configuration, the image may not be pushed into the registry.

## Setup

Prior to setting up our Codefresh pipeline, an Anchore Engine service needs to be accessible from the pipeline. Typically this is on port 8228. In this example, I have an Anchore Engine service on AWS EC2 with standard configuration. I also have a Dockerfile in a Github repository that I will build an image from during the first step of the pipeline. In the final step, I will be pushing the built image to an image repository in my personal Dockerhub.

Most typically, we advise on having a staging registry and production registry. Meaning, being able to push and pull images freely from the staging/dev registry, while maintaining more control over images being pushed to the production registry. In this example, I am using the same registry for both.

In the configuration section of the Codefresh pipeline, I've added the following environment variables: 

If `ANCHORE_FAIL_ON_POLICY` is set to true, the pipeline will fail, and the image will not be pushed to the registry. 

- `dockerhubUsername`
- `dockerhubPassword`
- `ANCHORE_CLI_URL`
- `ANCHORE_CLI_USER`
- `ANCHORE_CLI_PASS`
- `ANCHORE_CLI_IMAGE`
- `ANCHORE_RETRIES`
- `ANCHORE_FAIL_ON_POLICY`


## Build Image

In the first step of the pipeline, we build a Docker image from a Dockerfile as definied in our `codefresh.yaml`:

```
build_image:
    title: Building Docker Image
    type: build
    image_name: jvalance/sampledockerfiles
    working_directory: ./
    dockerfile: Dockerfile
```

## Conduct Anchore Scan

In the second step of the pipeline, we scan the built image with Anchore as definied in our `codefresh.yaml`:

```
anchore_scan:
    title: Scanning Docker Image
    image: anchore/engine-cli:latest
    description: Analyzing Image with Anchore...
    commands:
      - echo "Adding image to Anchore engine"
      - anchore-cli image add ${{ANCHORE_SCAN_IMAGE}}
      - echo "Waiting for image analysis to complete"
      - counter=0
      - while (! (anchore-cli image get ${{ANCHORE_SCAN_IMAGE}} | grep 'Status\:\ analyzed') ) ; do echo -n "." ; sleep 10 ; if [ $counter -eq ${{ANCHORE_RETRIES}} ] ; then echo " Timeout waiting for analysis" ; exit 1 ; fi ; counter=$(($counter+1)) ; done
      - echo "Analysis complete"
      - if [ "${{ANCHORE_FAIL_ON_POLICY}}" == "true" ] ; then anchore-cli evaluate check ${{ANCHORE_SCAN_IMAGE}}  ; fi 
```

Depending on the output of the policy evaluation, the pipeline may or may not fail. In this case, I have set `ANCHORE_FAIL_ON_POLICY` to true and exposed port 22. This is in violation of a policy rule, so the build will fail during this step.

## Push image

In the final step of the pipeline, we push the Docker image to a registry as defined in the `codefresh.yaml`:

```
push_image:
    title: Push Docker Image
    description: Pushing Docker Image...
    type: push
    candidate: '${{build_image}}'
    tag: latest
    registry: docker.io
    credentials:
      username: '${{dockerhubUsername}}'
      password: '${{dockerhubPassword}}'
```


## Conclusion

By following along with the example above, we can see how simply Anchore scanning can be configured and added to a Codefresh pipeline line in order to immediately bolster the security of the built container images. 

As a tip, we recommend adjusting the policy bundles and evaluation to align with your current organizations security and compliance requirements. 

Additionally, we advise having separate Docker registries for images that are being scanned with Anchore, and images that have passed an Anchore scan. For example, a registry for dev/test images, and a registry to certified, trusted, production-ready images. 