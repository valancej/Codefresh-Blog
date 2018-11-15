# Adding Anchore container image scanning to your Codefresh pipelines

## Background

Docker images themselves are commonly built with a continous integration tool like Codefresh. As with any CI pipeline building an artifact, it becomes important to employ the proper security testing tools to ensure a higher lever a confidence when deploying. In the case of Docker images, a mechanism for scanning and inspecting these artifacts.

Anchore is a service that conducts static analysis on container images, and applies user-defined acceptable polices to allow automated container image validation and certification. This means that not only can Anchore users gain a deep insight into the OS and non-OS packages contain within an image, but they have the ability to create governence around the artifact and it's contents via customizable polices. 

