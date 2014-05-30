# Autoscale - Setting Up

## Setting up

We'll assume that we are Jo, and we have already set up the DB server, web servers, and an image that builds the the auto scale webservers.

A lot of the autoscaling stuff can be done in the [user interface](http://www.rackspace.com/blog/easily-scale-your-cloud-with-rackspace-auto-scale/); in this article we'll just be using the API.

There are [several SDKs and tools](http://developer.rackspace.com/#home-sdks) that we could use for automating auto-scale, however in this article series we'll be using the plain http api, with the aid of a couple of tools.

I've downloaded and installed these tools onto my laptop:

 * http://stedolan.github.io/jq/
 * https://github.com/jkbr/httpie

They are optional. You can do everything in this article with the plain old [curl](http://docs.rackspace.com/servers/api/v2/cs-devguide/content/curl.html). However using 'http' and 'jq' will make your api usage much easier.

### httpie install

These are the commands I used to install httpie on my Ubuntu Trusty box.

    apt-get install python-setuptools -y
    git clone https://github.com/jakubroztocil/httpie.git
    cd httpie
    sudo python setup.py install

It is beyond the scope of this article to go into installation details for all platforms.

### jq install

These are the commands I used to install it on Ubuntu Trusty:

    wget http://stedolan.github.io/jq/download/linux64/jq
    chmod +x jq
    sudo cp jq /usr/bin/

## Using httpie and jq

We'll be running these commands on the linux command line, we'll store all the data in envirnoment variables.

First lets store our login info in the environment variables:

    USER=your-cloud-account-user-name
    KEY=your-api-key

Of course, substitute your actual Rackspace cloud account's user name and api key, eg:

    USER=misterawesome
    KEY=3a0320a753240a432075342053a70739

### Authentication

This handy one liner will authenticate you and grab all the info about the APIs into an environment variable called `json`:

    json=$(echo "{ \"auth\":{ \"RAX-KSKEY:apiKeyCredentials\":{ \"username\":\"${USER}\", \"apiKey\":\"${KEY}\" } } }" | http POST https://auth.api.rackspacecloud.com/v2.0/tokens)

You can look at all the nifty output using the jq tool:

    echo $json | jq .

We need the authentication token, and the url for the autoscale API. To get the token:

    token=$(echo $json | jq '.access | .token | .id' | sed s/\"//g)
    auth="X-Auth-Token:$token"

You can see the list of APIs provided with:

    echo $json | jq '.access | .serviceCatalog | .[] | .name'

We're after the autoscale API for the ORD datacenter, we can get that with:

    url=$(echo $json | jq '.access | .serviceCatalog | .[] | select(.name == "autoscale") | .endpoints | .[] | select(.region == "ORD") | .publicURL' | sed s/\"//g)

Now that we have the url and the token we can start to use the API.

## List scaling groups

Lets use the [API docs](http://docs.rackspace.com/cas/api/v1.0/autoscale-devguide/content/GET_listGroups_v1.0__tenantId__groups_autoscale-groups.html) to list all the current scaling groups:

    http $url/groups $auth

You shouldn't have any scaling groups initially.

Lets create our first simple scaling group; we'll call it 'group1' for now. We'll need to [put some json data for the request](http://docs.rackspace.com/cas/api/v1.0/autoscale-devguide/content/POST_createGroup_v1.0__tenantId__groups_autoscale-groups.html#POST_createGroup_v1.0__tenantId__groups_autoscale-groups-Request).:

    * launchConfiguration->args->loadBalancerId = 234883 - This is Jos LB ID, you can get yours from the API or from the web interface
    * server
      * name = "Web_" - This will have 01, 02, etc appended to it
      * imageRef = 515b9d07-9ac0-4aff-b836-e1fa08940e9a - The image UUID for the snapshot you'll use to create every web server
      * flavorRef = performance1-1 - Which size server to use
      * networks [uuid=940e0093-ba6b-44f9-b3af-e24556293e9da] - You can get your network UUID from the UI or 'nova network-list'
    * groupConfiguration
      * name = web-heads
      * maxEntitities = 20
      * minEntitities = 1
      * cooldown = 360 - Min time between scalings in seconds (6 minutes for us).
    * scalingPolicies
      * [] -- We'll fill these in later

Lets run it on the command line:

    group=$(echo -e '{
        "launchConfiguration": {
            "args": {
                "loadBalancers":[ { "port":80, "loadBalancerId":234883 } ],
                "server": {
                    "name":"Web_",
                    "imageRef":"515b9d07-9ac0-4aff-b836-e1fa08940e9a",
                    "flavorRef":"performance1-1",
                    "networks":[ { "UUID": "940e0093-ba6b-44f9-b3af-e24556293e9da" } ]
                 }
            },
            "type":"launch_server"
        },
        "groupConfiguration": {
            "name":"web-heads",
            "maxEntities":20,
            "cooldown":360,
            "minEntities":1
        },
        "scalingPolicies": []
    }' | http POST $url/groups $auth
    )

That'll put the result in the `group` environment variable. You can see it with:

    echo $group | jq .

We need the ID:

    group_id=$(echo $group | jq '.group | .id' | sed s/\"//g)

## Conclusion

In this article we started to use load balancer API. We learned about some neat command line tools and shell scripting tricks that make API usage quite pleasant.

In the next article we'll look at making and using a webhook policy. This is a URL that can be called to cause a scale up or a scale down; it allows the autoscaling system to be hooked into the monitoring system.
