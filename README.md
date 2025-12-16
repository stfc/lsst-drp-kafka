# lsst-drp-kafka
Repository to store configuration for Kafka, MM2, ingestD, and prometheus for the LSST DRP.

This repository includes the dockerfiles to deploy a new messaging stack for the LSST DRP at RAL.

To complete the setup you will need:
- to populate db-auth.yaml
- setup x509 credentuals on the host and have them mounted to the ingestD to allow access to echo
- Update vande with the new IP address for the VM that is hosting the prometheus

This repository should be updated when new DPR repos are created. When the number of ingestD's that are required are increased to much more it would be sensible to move the ingestDs to their own docker-compose.yaml like prometheus is.
