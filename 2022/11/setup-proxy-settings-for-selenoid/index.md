# Setup proxy settings for Selenoid


Selenoid is a powerful Golang implementation of original Selenium hub code.
It is using Docker to launch browsers. 

There could be situation, when you need to run Selenoid in an environments without 
direct Internet access. And your tests may fail if the browser loads some resources from
Internet resources. This small article will address such issue.

To pass proxy settings into Docker containers that Selenoid starts you need to configure
`browsers.json` on your own:

```json
{
    "chrome": {
        "default": "107.0",
        "versions": {
            "107.0": {
                "image": "nexus.example.com/selenoid/vnc:chrome_107.0",
                "port": "4444",
                "env": [
                  "HTTP_PROXY=http://proxy.example.com:8080/", 
                  "HTTPS_PROXY=http://proxy.example.com:8080/", 
                  "NO_PROXY=127.0.0.1,localhost,docker,.example.com"
                ]
            }
        }
    }
}
```

As you can see is this example you may use internal Nexus repository used for docker images as well.

So, to start Selenoid you can use the following script `selenoid-start.sh`:
```shell
#!/bin/bash

cm selenoid start --force --browsers-json ./browsers.json --args "-limit 6" -r "https://nexus.example.com"
cm selenoid-ui start -r "https://nexus.example.com"
```

`--args "-limit 6"` is used to specify maximum concurrent containers that Selenoid can run.


