# Note for One Medical Infrastructure team

We created this fork because the main [logsearch-boshrelease](https://github.com/cloudfoundry-community/logsearch-boshrelease) repository did not support Elasticsearch v2.4.3. Because this is a temporary solution, we have not configured an external blobstore for Elasticsearch v2.4.3. To use this with BOSH locally, you will need to download the Elasticsearch tarball and add it to the BOSH blobs before uploading the blob to the director of your choice.

0. Checkout `v204.0.0`

0. Run `bosh sync-blobs` to get all blobs locally

0. Download the elasticsearch tarball:
```sh
# If you don't already have wget, install it with `brew install wget`
$ wget https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-2.4.3.tar.gz
```

0. Use `bosh add-blob` to add the elasticsearch v2.4.3 tarball to your list of BOSH blobs
```sh
$ bosh add-blob ./elasticsearch-2.4.3.tar.gz elasticsearch/elasticsearch-2.4.3.tar.gz
```

0. Remove `./elasticsearch-2.4.3.tar.gz` (it's been copied into the blobs folder) and commit changes to config/blobs.yml
```sh
$ rm elasticsearch-2.4.3.tar.gz
$ git commit config/blobs.yml -m "Some temporary message"
```

0. Run `bosh create-release`

  * Do not run this with the `--final` flag. The release created should have a label like `204.0.0+dev.1`

0. Upload your newly created release to the bosh director of your choice
```sh
$ bosh -e <director_alias> upload-release 
```

You're now ready to go with Elasticsearch 2.4.3 on BOSH!

# Logsearch

A scalable stack of [Elasticsearch](http://www.elasticsearch.org/overview/elasticsearch/),
[Logstash](http://www.elasticsearch.org/overview/logstash/), and
[Kibana](http://www.elasticsearch.org/overview/kibana/) for your
own [BOSH](http://docs.cloudfoundry.org/bosh/)-managed infrastructure.

## BREAKING CHANGES

Logsearch < v23.0.0 was based on Elasticsearch 1.x and Kibana 3.

Logsearch > v200 is based on Elasticsearch 2.x and Kibana 4.

There is NO upgrade path.  Sorry :(

## Getting Started

This repo contains Logsearch Core; which deploys an ELK cluster that can recieve and parse logs via syslog
that contain JSON.

Most users will want to combine Logsearch Core with a Logsearch Addon to customise their cluster for a
particular type of logs.  Its likely you want to be following an Addon installation guides - see below
for a list of the common Addons:

  * [Logsearch for CloudFoundry](https://github.com/cloudfoundry-community/logsearch-for-cloudfoundry)

If you are sure you want install just Logsearch Core, read on...

## Installing Logsearch Core

0. Upload the latest logserach release

   * Download the latest logsearch release
   
     NOTE: At the moment you can get working logsearch release by cloning Git repository and creating bosh release from it.

      Example:
   
      ```sh
      $ git clone https://github.com/cloudfoundry-community/logsearch-boshrelease.git
      $ cd logsearch-boshrelease
      $ bosh create release
      ```
   
   * Upload bosh release
   
      Example:

      ```sh
      $ bosh upload release
      ```
   
0. Customise your deployment stub:

   * Make a copy of `templates/stub.$INFRASTRUCTURE.example.yml` to `stub-logsearch.yml`
   
      Example: 
      ```sh
      $ cp logsearch-boshrelease/templates/stub.openstack.example.yml stub-logsearch.yml
      ```
     
   * Edit `stub-logsearch.yml` to match your IAAS settings

0. Generate a manifest with `scripts/generate_deployment_manifest $INFRASTRUCTURE stub-logsearch.yml > logsearch.yml`

   Example: 
   
   ```sh
   $ logsearch-boshrelease/scripts/generate_deployment_manifest openstack stub-logsearch.yml > logsearch.yml
   ```
   
   Notice `logsearch.yml` generated.

0. Make sure you have these 2 security groups configured:

   * `bosh` which allow access from this group itself

   * `logsearch` which allow access to ports 80, 8080, 8888

0. Deploy!

   ```sh
   $ bosh -d logsearch.yml deploy
   ```

## Common customisations:

0. Adding new parsing rules:

        logstash_parser:
          filters: |
             # Put your additional Logstash filter config here, eg:
             json {
                source => "@message"
                remove_field => ["@message"]
             }


### Release Channels

 * The latest stable, final release will be soon available on [bosh.io](http://bosh.io/releases)
 * **develop** - The develop branch in this repo is deployed to our test environments.  It is occasionally broken - use with care!

## Known issues

#### VMs lose connectivity to each other after VM recreation (eg. instance type upgrade)

While this issue is not specific to this boshrelease, it is worth noting.

On certain IAAS'es, (AWS confirmed), the bosh-agent fails to flush the ARP cache of the VMs in the deployment which, in rare cases, results in VMs not being able to communicate with each other after some of them has been recreated. The symptoms of when this happens are varied depending on the affected VMs. It could be anything from HAproxy reporting it couldn't find any backends (eg. Kibana) or the parsers failing to connect to the queue.

To prevent stale ARP entries, set the `director.flush_arp` property of your BOSH deployment to `true`.

The issue, if occurs, should fix itself as the kernel updates incomplete ARP entries, which **should** happen within minutes

This can also be done manually if an immediate manual fix is preferred. This should be done on the VMs that are trying to talk to the VM that has been recreated.

```
arp -d $recreated_vm_ip
```

## License

[Apache License 2.0](./LICENSE)
