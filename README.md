# Platform Upload Processor II

The Platform Upload Processor II (PUPTOO) is designed to recieve payloads for the `advisor` service
via the message queue, extract facts from the payload, forward the information to the inventory
service.

## Details

The PUPTOO service is a component of the Insights Platform that validates the payloads uploaded
to Red Hat by the insights-client. The service is engaged after an upload has been recieved and stored in
cloud storage.

PUPTOO retrieves payloads via URL in the message, processes it through
insights-core to extract facts and guarantee integrity of the archive, and send the extracted info to the inventory service. 

The service runs in Openshift Dedicated.

## How it Works

![UML](http://www.plantuml.com/plantuml/png/VLB1Rjim3BtxAmJlkWJjslKG344FMuv3W66dfZ1acOwvoP8dKico8Fy-KMxGP6l2Gm8_FfAFZteare5ZRmj6jg2MtvTg6Rm18dHhjR1-Mmo9WGO7xLWDLdFhGp-DW_MwcUfcW-J3LSv6MsmqetVdj3YmzszNejk0OnzsqyuJ9oKX2JgZTZKM5yHCvcFhcUffNNpr32hWkcFbsqlwPsfV1lWLWRZ2ffofKjUckVrmTr--NxbI6-EZOy5l96upEWHqeiOABjoF3nad_0C2oN-5hgft33Hc86pGv6G3ibTsHHrXeHZDi4wB2wT52-e8v6mCULWTpK_WIhu4hH_kasfmZBpBQKrm0bN48LcOgJsmRZJhHDiVLkvGZ5QzMlRPRvquGqe7q-46L1jyltBVxjGuzR-ABxyvQCEaslmhz6FjocsHEuMEn7-Q61-Wq9OLHMFaeYh5iT_ruu_75FxvGSTgkT_BMa_xCWl96sRJTyJGLqFmucU63y-IE0wiJapvv5DdTDfMQLprWxoR7mgmb6pGQ6E04ybFxsvIB1RFeltIa_z7juNsLFRrP_VQHL8l0iganpN5TRScywx6SVHZOiVU_WK0 "PUPToo Processing Flow")

The PUP service workflow is as follows:

  - Recieve a message from `platform.upload.advisor` topic in the MQ
  - PUP downloads the archive from the url specified in the message
  - Insights Core is engaged to open the tar file and extract the facts as determined by the `get_canonical_facts` method in insights-core
  - During extraction it also runs a custom `system-profile` ruleset to extract more information about the ssytem
  - PUP sends the result to inventory via the message queue

### JSON

The JSON expected by the PUP service from the upload service looks like this:

```
{"account": "123456",
 "principal": "654321",
 "request_id": "23oikjonsdv329",
 "size": 234,
 "service": "advisor",
 "b64_identity": "<some big base64 string>",
 "metadata": "{'some_key': 'some_value'}, # optional
 "url": "http://some.bucket.somewhere/1234"}
```

The message sent to the inventory service will be what is above, but also include facts:

```
...
"facts": [{'facts': {"insights_id": "a756b571-e227-46a5-8bcc-3a567b7edfb1",
                    "machine_id": null,
                    "bios_uuid": null,
                    "subscription_manager_id": null,
                    "ip_addresses": [],
                    "mac_addresses": [],
                    "fqdn": "Z0JTXJ7YSG.test"}
            'namespace': 'insights-client',
            'system-profile': {"foo": "bar"}}]
```

**The above facts are managed by the [insights-core](https://www.github.com/RedHatInsights/insights-core) project and may be added or taken away. The README should be updated to reflect those
changes**

Fields:

  - account: The account number used to upload. Will be modified to `account_number` when posting to inventory
  - principal: The upload org_id. Will be modified to `org_id` when posting to inventory
  - request_id: The ID of the individual uploaded archive
  - size: Size of the payload in bytes
  - service: The service name as provided by the MIME type. 
  - url: The url from which the archive will be downloaded
  - facts: An array containing facts for each host
  
If the fact extraction fails, the archive will be considered "bad." A message will be sent back to the upload service so the file can be moved to the rejected bucket.

Failure example:

    {"validation": "failure", "request_id": "23oikjonsdv329"}

### Running

The default environment variables should be acceptable for testing.  
PUPTOO does expect a kafka message queue to be available for connection.

#### Prerequisites

    python3
    venv or pipenv

#### Python

Create a virtualenv using pipenv and install requirements. Once complete you can start the app by running `pup.py`

    pipenv install

#### Running Locally

Activate your virtual environment and run the validator

    pipenv shell
    python ./app.py

#### Running with Docker Compose

Two docker-compose files are made available in this repo for standing up a local dev environment. The `docker-compose.yml` file stands up putoo, kafka, and minio for isolated tested. The `full-stack.yml` file stands up ingress, kafka, puptoo, minio, and inventory components so that the entire first bits of the platform pipeline can be tested. 

Stand Up Isolated Puptoo  
    cd dev && sudo docker-compose up

Stand Up Full stack
    cd dev && sudo docker-compose -f full-stack.yml up 

**NOTE**: The full stack expects you to have an ingress and inventory image available. See those projects for steps for building the images needed. It's also typical for puptoo to fail to start if it can't initially connect to kafka. If this happens, simply run `sudo docker-compose -f full-stack up -d pup` to have it attempt another startup.

## File Processing

The best way to test is by standing up this server and incorporating it with the upload-service. The [insights-upload](https://www.github.com/RedHatInsights/insights-upload) repo has a docker-compose that will get you most of the way there. Other details regarding
posting your archive to the service can be found in that readme.

This test assumes you have an inventory service available and ready to use. Visit the `insights-host-inventory` repo for those instructions. 

## Running with Tests

TODO: There is currently no PUP test suite

## Deployment

The PUPTOO service `master` branch has a webhook that notifies Openshift Dedicated cluster to build a new image in the `buildfactory` project. The image build will then trigger a redployment of the service in `Platform-CI`. To deploy to QA, commits must be cherry picked from master to a branch forked from `stable`. The resulting branch should be submitted as a PR against `stable` where it will be tested by the QE vortex. Once the result is returned, it can be merged.

Automated QE systems will later tag the `stable` image to `qa` where it will deployed in `Platform-QA`. Once there, a Jenkins job can be activated to tag the image to production.

## Contributing

All outstanding issues or feature requests should be filed as Issues on this Github repo. PRs should be submitted against the master branch for any features or changes.

## Running unit tests

TODO: Write some tests

## Versioning

New functionality that may effect other services should increment by `1`. Minor features and bugfixes can increment by `0.0.1`

## Authors

* **Stephen Adams** - **Initial Work** - [SteveHNH](https://github.com/SteveHNH)
