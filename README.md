This repository contains an Azure DevOps build pipeline for automating TLS certificate issuance on Azure with Let's Encrypt. 

More details [here](./letsencrypt-automatization.md).

The solution is based on the following article [https://bit.ly/34qOsil](https://bit.ly/34qOsil).
Due to updates in the actual version of PoshACME (4.x by January 2021), the in this article mentioned repo and code doesn't work anymore.
I've made additional changes to use an actual version of AzCopy, too.