# Curator

[Curator](https://github.com/elastic/curator) is a CLI tool to easily automate common actions you may take against elasticsearch. In this directory we have provided a sample curator.yml config file, as well as an action-file.yml. If you run curator with these files (and some needed environment variables), it will delete and elasticsearch indexes older than 10 days. This can easily be configured to be run automatically via Jenkins or any other CI/CD server to prevent the data usage of your ELK stack from being excessivly large.

To use the default config and action file we have provided, you'll first need to set some environment variables with your elasticsearch instance details. Specifically the host(s) and port to access elasticsearch:

- `ELASTICSEARCH_ADDRESS` - a comma seperated list of IP addresses (or hostnames) of your elasticsearch instance(s)   
- `ELASTICSEARCH_PORT` - your elasticsearch port (9200 is the default unless you changed this in your docker-elk configs)

Once these are set, you can simply run the following command to clean up old indices:

```bash
curator --config ./curator.yml ./action-file.yml
```



