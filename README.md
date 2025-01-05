# Self Hosting with lxc and Linux

Notes on setting up LXC containers with bridged networking on Ubuntu 24.04, with the aim of running a set of services on a private network.

The purpose of this is to set up a number of private and public services that include:

* NextCloud - for control and sharing of documents
* GitLab (Community Edition) - for software version control
* Remind - for managing tasks and project management

For this to be cost effective, the hardware needs to be modest. A NUC would be ideal, but in my own case, I am reusing an old Mac Mini (2014) running Ubuntu 24.04. This has an internal SSD fitted.

For bulk storage, I am using a budget Synology NAS.

