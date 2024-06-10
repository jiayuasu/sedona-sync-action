# sedona-sync-action

The action in this repo automatically pushes Sedona docker image and website on a daily basis.

* Website: it pulls content from https://github.com/apache/sedona/tree/website and pushes to https://github.com/apache/sedona-website/tree/asf-site, as the `latest-snapshot` version.
* Docker: it pulls content from sedona master branch and publish to https://hub.docker.com/r/apache/sedona as the `latest` tag.