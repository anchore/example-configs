This is an example of a declarative pipeline in using Jenkins.  This example makes use of several elements of Jenkins, including:

Job Parameters: The Tag variable is being set using a job parameter, purely to demonstrate the ability to configure jobs in this way.

Jenkins Credentials Store: Credentials for the Anchore instance, and DockerHub are stored in the Jenkins credentials stored.  For Anchore, the withCredentials construct is used to securely access credentials without exposing them in the logs

Docker: This example is using a Docker container of anchore-cli to perform the interactions with Anchore purely to demonstrate the concept. This could be augmented to use syft, grype, and/or anchorectl.

Notes of interest
The job is setup to pull a source project from a configured git repository, perform the build of that code base including building the container image. (lines 14-29)

The job is configured to check on the availability/reachability of the configured Anchore installation to demonstrate how the pipeline can be guarded from timeouts and failures due to long scan times or network connectivity issues. A status variable is set that is used to guard calls. (lines 40-49 and line 61 shows using this status).

The job is also set up to with configurable wait periods (via the WAIT_TIMEOUT in seconds) to determine how long the pipeline should wait on a scan to complete before proceeding. (lines 64-79).

 

There are many ways this job could be written and configured depending on organizational requirements.  The intention of this example is to demonstrate some of the various methods and capabilities available. Some examples include:

Writing reusable functions and pulling them into an organizational library to abstract away the Anchore interactions

Expanding on the availability checking to look at HTTP status codes as well as other Anchore health endpoints


